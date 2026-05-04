# CSV Test Runner

I built this tool to solve a specific problem that emerged as the CartAlchemy platform grew to serve multiple customers: fixing something for one customer can break it for another. Each customer runs a different ERP, different theme configuration, and different feature set — so a change to shared platform code carries real regression risk across the whole customer base. We needed a way to run baseline tests across all customer configurations before any production deployment.

The CSV spreadsheet is the single source of truth: each row is a test case scoped to a specific customer (application), and the runner reads it directly, filters to the relevant subset, handles authentication per user role, navigates the app via Playwright, and records a pass/fail result with screenshots. No individual spec files, no test framework boilerplate.

The tagging system is what makes it practical day-to-day. If a change touches wishlist functionality, you can run every test case tagged `wishlist` across all customers. If you just need a critical path end-to-end check before a deployment, `critical-path` gets you there in a focused run without executing the full suite. The filter combinations — by customer, phase status, priority, and tag — mean you can scope a run precisely to what's actually at risk.

The approach was validated against an initial customer deployment with a 566-test-case spreadsheet covering four user role types, role × feature access combinations, and flows including ERP integration, cart approval workflows, and WebAuthn. Additional customers are added by extending the spreadsheet with their application tag and environment config.

For architecture in context with the other tools, see the [system component map](./README.md#system-component-map).

---

## Filter logic and execution flow

```mermaid
flowchart TD
    subgraph S1["Step 1 — Environment Setup"]
        A1["Load environment-config.env\n(shared Jira + browser settings)"] --> A2["Load specs/{customer}/environment-config.{env}.env\n(env-specific URL override)"]
        A2 --> A3{"TEST_BASE_URL contains\ndev / test / qa / staging?"}
        A3 -->|"no"| A4["ABORT — production guard"]
        A3 -->|"yes"| A5["Map user roles to credential variables\nBusiness · Sales Rep · CS · Admin · Group"]
    end

    subgraph S2["Step 2 — CSV Filtering"]
        B1["Read CSV file\n(Test Case ID · Category · Priority ·\nURL Path · Auth Required · User Role ·\nTags · Phase Status · Phase Notes)"] --> B2["Application filter\ninclude rows where Application = {customer}"]
        B2 --> B3["Phase status filter"]
        B3 --> B4["Exclude: deferred · not-applicable"]
        B3 --> B5["Flag for review: needs-update"]
        B3 --> B6["Include: current"]
        B6 --> B7{"Run filter"}
        B7 -->|"phase1"| B8["High priority\n+ critical-path tag\n+ Business user role\n+ key categories only"]
        B7 -->|"critical-path"| B9["All High priority\n+ all critical-path tagged"]
        B7 -->|"all-current"| B10["All rows with\ncurrent phase status"]
        B7 -->|"custom tag"| B11["All rows containing\nmatching tag"]
        B8 & B9 & B10 & B11 --> B12["Print selection summary:\nN selected · N skipped · N flagged"]
    end

    subgraph S3["Step 3 — Test Execution"]
        C1["For each selected test case\n(in CSV order)"] --> C2{"Auth Required\n= true?"}
        C2 -->|"yes"| C3{"Active browser\nsession for this role?"}
        C3 -->|"yes — reuse session"| C5
        C3 -->|"no — authenticate"| C4["Playwright: navigate to /login\nfill credentials for User Role\nsubmit form"]
        C4 --> C4a{"Company Selection\nRequired = true?"}
        C4a -->|"yes"| C4b["Select customer\nvia TEST_PROJ_CUSTOMER_NUMBER"]
        C4a -->|"no"| C5
        C4b --> C5
        C2 -->|"no"| C5["Playwright: navigate to\n${TEST_BASE_URL}${URL Path}"]
        C5 --> C6["Claude: verify Expected Result\nby observing live UI\n(Test Steps used as hints, not script)"]
        C6 --> C7["Playwright: capture screenshot\n+ read console messages"]
        C7 --> C8["Record result:\nPASS · FAIL · ERROR · SKIP"]
        C8 --> C9{"More tests?"}
        C9 -->|"yes — same role"| C1
        C9 -->|"yes — role change"| C10["Clear session\nre-authenticate as new role"]
        C10 --> C1
    end

    subgraph S4["Step 4 — Results"]
        D1["Save per-test JSON\ntest-runs/{date}/{customer}-{env}/results/{spec_id}/"] --> D2["Save screenshots\ntest-runs/{date}/{customer}-{env}/screenshots/{spec_id}/"]
        D2 --> D3["Generate Markdown summary\ncsv-run-summary.md\n(counts · table · failure details · flagged tests)"]
    end

    A5 --> B1
    B4 & B5 --> B12
    B12 --> C1
    C9 -->|"no"| D1
```

---

## CSV structure

Each row in the spreadsheet maps to one test case:

| Column | Description |
|--------|-------------|
| `Test Case ID` | Unique identifier (e.g. `TC-001`) |
| `Application` | Which platform this test applies to |
| `Category` | Feature area (e.g. Login, Cart, Checkout) |
| `Priority` | High / Medium / Low |
| `URL Path` | Path appended to `TEST_BASE_URL` |
| `Auth Required` | true / false |
| `User Role` | Business · Sales Rep · CS · Admin · Group |
| `Tags` | Comma-separated (e.g. `critical-path, regression`) |
| `Phase Status` | current · deferred · not-applicable · needs-update |
| `Phase Notes` | Reason for deferred/needs-update status |
| `Expected Result` | What the AI verifies in the UI |
| `Test Steps` | Optional hints — not a strict script |

---

## Result data model

```mermaid
classDiagram
    class CSVTestResult {
        +String spec_id
        +String source = "csv"
        +String csv_row
        +String status
        +String timestamp
        +Number duration_seconds
        +String environment
        +String target_url
        +String user_role
        +String outcome_summary
        +Verification[] verifications
        +String[] must_not_violations
        +String[] unexpected_issues
        +Screenshot[] screenshots
        +String[] console_errors
        +String[] execution_log
        +CSVMetadata metadata
    }

    class Verification {
        +String criterion
        +String source
        +String result
        +String observed
        +String screenshot
        +String detail
    }

    class Screenshot {
        +String name
        +String path
        +String description
        +String timestamp
    }

    class CSVMetadata {
        +String customer
        +String environment
        +String csv_file
        +String filter
        +String priority
        +String[] tags
        +String category
    }

    CSVTestResult "1" --> "*" Verification : verifications
    CSVTestResult "1" --> "*" Screenshot : screenshots
    CSVTestResult "1" --> "1" CSVMetadata : metadata
```

**Storage paths:**

| Artifact | Path |
|----------|------|
| Per-test result JSON | `test-runs/{YYYY-MM-DD}/{customer}-{env}/results/{spec_id}/{spec_id}-result.json` |
| Screenshots | `test-runs/{YYYY-MM-DD}/{customer}-{env}/screenshots/{spec_id}/` |
| Run summary | `test-runs/{YYYY-MM-DD}/{customer}-{env}/csv-run-summary.md` |

---
