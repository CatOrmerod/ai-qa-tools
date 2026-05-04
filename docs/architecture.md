# Architecture

Detailed diagrams for each tool in the AI QA Automation Suite. For a high-level system overview, see the [README](../README.md).

---

## 1. System Component Map

All tools run inside Claude Code on the developer's machine. They read from a local file system (configs, source repos, CSV files), drive a browser via the Playwright MCP server, and communicate with Jira and Bitbucket over HTTPS.

```mermaid
flowchart LR
    subgraph DEV["Developer Machine"]
        subgraph CC["Claude Code — Slash Commands"]
            CR["/run-jira-code-review"]
            VV["/run-jira-visual-verification"]
            TR["/run-csv-tests"]
            PS["/post-pr-summary"]
        end

        subgraph MCP["MCP Server"]
            PW["Playwright MCP\n(@playwright/mcp)"]
        end

        subgraph FS["File System"]
            ENV["environment-config.env\n(Jira creds, browser settings)"]
            SPECS["specs/{repo}/\n(env config per repo + environment)"]
            REPOS["source-repos/{repo}/\n(cloned source code)"]
            CSV["docs/*.csv\n(test case spreadsheets)"]
            RESULTS["test-runs/\n(JSON results + screenshots)"]
        end
    end

    subgraph CLOUD["Atlassian Cloud"]
        JIRA["Jira REST API v3\n(/rest/api/3/issue)"]
        BB["Bitbucket REST API v2\n(/2.0/repositories)"]
    end

    subgraph APP["Test Environment"]
        WEB["Web Application"]
        ADMIN["Admin Interface"]
    end

    CR -->|"reads diff"| REPOS
    CR -->|"reads"| ENV
    CR -->|"reads"| SPECS
    CR -->|"writes results"| RESULTS
    VV -->|"reads"| ENV
    VV -->|"reads"| SPECS
    VV -->|"writes results + screenshots"| RESULTS
    TR -->|"reads"| CSV
    TR -->|"reads"| ENV
    TR -->|"reads"| SPECS
    TR -->|"writes results + screenshots"| RESULTS
    PS -->|"reads"| RESULTS

    CR -->|"GET issue + comments\nPOST comment"| JIRA
    VV -->|"GET issue + comments\nPOST comment\nPOST attachments"| JIRA
    PS -->|"GET dev-status\n(find linked PR)"| JIRA
    PS -->|"POST PR comment"| BB

    VV -->|"drives browser"| PW
    TR -->|"drives browser"| PW
    PW -->|"HTTP"| WEB
    PW -->|"HTTP"| ADMIN
```

---

## 2. AI Code Review — Six-Phase Pipeline

The code review runs six sequential phases. Phases 1–3 gather context (code diff, npm audit, Jira ticket). Phases 4–5 are AI analysis (standards review and AC coverage mapping). Phase 6 produces and optionally posts the result.

```mermaid
flowchart TD
    subgraph P1["Phase 1 — Setup & Git"]
        A1["Load environment-config.env\n+ specs/repo/environment-config.env"] --> A2["git fetch origin\ngit checkout feature branch"]
        A2 --> A3["git diff {base}...HEAD\n(list changed files)"]
        A3 --> A4{"package.json\nin diff?"}
        A4 -->|"yes"| A5["npm audit --json\nper folder (website / admin / root)"]
        A5 --> A6["Classify: CLEAN / INFO_ONLY /\nREVIEW_RECOMMENDED / ACTION_REQUIRED"]
        A4 -->|"no"| A7["Skip audit"]
        A6 --> A8["Present changed file list\nfor developer confirmation"]
        A7 --> A8
    end

    subgraph P2["Phase 2 — Jira Context"]
        B1["GET /rest/api/3/issue/{id}\n?fields=summary,description,assignee,\nduedate,customfield_10015,timeoriginalestimate"] --> B2["GET /rest/api/3/issue/{id}/comment"]
        B2 --> B3["Extract acceptance criteria\nfrom description + comments"]
        B3 --> B4["Check mandatory fields:\nsummary · description · start date\ndue date · assignee · estimate"]
    end

    subgraph P3["Phase 3 — Code Reading"]
        C1["Read full content of each changed file"] --> C2["Read related files for context\n(callers, schemas, route handlers, tests)"]
    end

    subgraph P4["Phase 4 — Standards Review"]
        D1["JavaScript\nnaming · const/let · arrow fn\nno var · no commented-out code"]
        D2["Error handling\nroute() wrapper · Pino logger\nconsistent return · no empty catch"]
        D3["Security\nSSRF · path traversal · rate limiting\nheaders · crypto · sensitive data exposure"]
        D4["Express / API\nresponse format · sanitization\nHTTP method semantics · breaking changes"]
        D5["MongoDB\ndirect driver · naming conventions\nprojections · N+1 · transactions · indexes"]
        D6["Testing\ncoverage · quality\nhardcoded data · implementation vs outcome"]
        D1 & D2 & D3 & D4 & D5 & D6 --> D7["Classify each finding:\nCritical · Warning · Suggestion · Positive"]
    end

    subgraph P5["Phase 5 — AC Coverage"]
        E1["For each acceptance criterion —\nidentify which files address it"] --> E2["Coverage per criterion:\nIMPLEMENTED · PARTIAL · MISSING · UNCLEAR"]
        E2 --> E3["Delivery verdict:\nCOMPLETE · INCOMPLETE · OVER-SCOPED · UNCLEAR"]
    end

    subgraph P6["Phase 6 — Output"]
        F1["Overall review status:\nAPPROVED · NEEDS_DISCUSSION · CHANGES_REQUESTED"] --> F2["Save versioned JSON\ntest-runs/TICKET-ID/review-vN.json"]
        F2 --> F3{"Confirm post\nto Jira?"}
        F3 -->|"yes"| F4["POST /rest/api/3/issue/{id}/comment\nADF: headings · tables · code blocks\nAC traceability matrix"]
        F3 -->|"no"| F5["Display full report\nin terminal"]
    end

    A8 --> B1
    B4 --> C1
    C2 --> D1
    C2 --> D2
    C2 --> D3
    C2 --> D4
    C2 --> D5
    C2 --> D6
    D7 --> E1
    E3 --> F1
```

---

## 3. AI Visual Verification — Interaction Sequence

The visual verification tool is inherently interactive: Claude Code and Playwright MCP trade messages in a tight loop while navigating the application. This sequence diagram shows the full flow from command invocation to Jira posting.

```mermaid
sequenceDiagram
    actor Dev as Developer
    participant CL as Claude Code
    participant JR as Jira API
    participant PL as Playwright MCP
    participant AP as Running App
    participant FS as File System

    Dev->>CL: /run-jira-visual-verification repo TICKET-ID
    CL->>JR: GET /issue/TICKET-ID (summary, description, components)
    JR-->>CL: ticket data
    CL->>JR: GET /issue/TICKET-ID/comment
    JR-->>CL: comments (scope changes, clarifications)
    CL->>CL: build ordered AC checklist from ticket

    Note over CL,AP: Phase 3 — Self-serve test data discovery
    CL->>PL: browser_navigate(admin dashboard URL)
    PL->>AP: HTTP GET /admin
    AP-->>PL: admin page
    PL-->>CL: accessibility snapshot
    CL->>PL: browser_click / browser_navigate (browse records)
    PL->>AP: further requests
    AP-->>PL: product / order / customer data
    PL-->>CL: test records identified
    Note over CL: Only asks developer if data cannot be found

    Note over CL,AP: Phase 4 — Verify each acceptance criterion
    loop For each acceptance criterion
        CL->>PL: browser_navigate(target URL)
        PL->>AP: HTTP GET
        AP-->>PL: page response
        opt Auth required for this criterion
            CL->>PL: browser_navigate(/login)
            CL->>PL: browser_fill_form(email, password)
            PL->>AP: POST /login
            AP-->>PL: session established
            opt Company selection required
                CL->>PL: browser_select_option(customer number)
                PL->>AP: company selection submitted
                AP-->>PL: scoped session active
            end
            CL->>PL: browser_navigate(target URL)
            PL->>AP: HTTP GET (authenticated)
            AP-->>PL: authenticated page
        end
        CL->>PL: browser_snapshot()
        PL-->>CL: accessibility tree + page state
        CL->>CL: evaluate criterion (PASS / FAIL / PARTIAL / BLOCKED)
        CL->>PL: browser_take_screenshot()
        PL-->>CL: screenshot data
        CL->>PL: browser_console_messages()
        PL-->>CL: console logs + errors
        CL->>FS: save screenshot as NN-description.png
    end

    CL->>FS: save verify-TICKET-ID-vN.json
    CL->>Dev: display full verification report

    Dev->>CL: confirm post to Jira

    loop For each screenshot
        CL->>JR: POST /issue/TICKET-ID/attachments (multipart/form-data)
        JR-->>CL: attachment ID + media URL
    end
    CL->>JR: POST /issue/TICKET-ID/comment (ADF with results table + screenshot links)
    JR-->>CL: comment ID
    CL->>Dev: Jira link + pass/fail summary
```

---

## 4. CSV Test Runner — Filter Logic and Execution

The tool applies a two-stage filter before executing: first by application and phase status (removing deferred/not-applicable rows), then by the run filter (phase1, critical-path, all-current, or a custom tag). Session reuse avoids re-authentication between consecutive tests running under the same user role.

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

## 5. Configuration Cascade

Each tool resolves its configuration by layering three files. Later layers override earlier ones, allowing the same shared credentials to serve multiple repos and environments.

```mermaid
flowchart TD
    L1["environment-config.env\nShared — checked in at repo root"]
    L2["specs/{repo}/environment-config.env\nRepo-specific — one per source repo"]
    L3["specs/{repo}/environment-config.{environment}.env\nEnvironment override — one per target env"]
    L4["Resolved runtime config\nused by all tools"]

    L1 -->|"base"| L2
    L2 -->|"overrides shared"| L3
    L3 -->|"overrides repo"| L4

    V1["Defined in shared:\nJIRA_HOST\nJIRA_EMAIL\nJIRA_API_TOKEN\nBROWSER · HEADLESS\nVIEWPORT_WIDTH/HEIGHT\nTIMEOUT_DEFAULT"]
    V2["Defined in repo:\nTEST_ENVIRONMENT\nTEST_BASE_URL\nTEST_ADMIN_URL\nJIRA_DEFAULT_PROJECT\nTEST_USER_EMAIL / _PASSWORD\nTEST_ADMIN_EMAIL / _PASSWORD\nCustomer-specific test data vars"]
    V3["Overrides in env file:\nTEST_BASE_URL  (env-specific URL)\nTEST_ENVIRONMENT  (label only)"]

    L1 -. "variables" .-> V1
    L2 -. "variables" .-> V2
    L3 -. "variables" .-> V3
```

**Example:** running `/run-csv-tests PROJ PROJ-qa` loads:
1. `environment-config.env` (Jira credentials)
2. `specs/proj/environment-config.env` (proj credentials, admin URL)
3. `specs/proj/environment-config.proj-qa.env` (QA base URL override)

---

## 6. Result Data Model

All three tools write versioned JSON to `test-runs/`. The schemas share a common shape (type, ticket/spec ID, status, timestamp, verifications, screenshots) but differ in their metadata and source context.

```mermaid
classDiagram
    class CodeReviewResult {
        +String type = "code_review"
        +String ticket_id
        +String ticket_summary
        +String repo_name
        +String status
        +String timestamp
        +String base_branch
        +String feature_branch
        +Finding[] critical_issues
        +Finding[] warnings
        +Finding[] suggestions
        +String ac_coverage
        +String delivery_verdict
        +String npm_audit_result
        +ReviewMetadata metadata
    }

    class VisualVerificationResult {
        +String type = "visual_verification"
        +String ticket_id
        +String ticket_summary
        +String repo_name
        +String status
        +String timestamp
        +String environment
        +String base_url
        +Verification[] verifications
        +String[] console_errors
        +Screenshot[] screenshots
        +VerifyMetadata metadata
    }

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

    class Finding {
        +String file
        +String line
        +String rule
        +String description
        +String suggestion
    }

    class Screenshot {
        +String name
        +String path
        +String description
        +String timestamp
    }

    class ReviewMetadata {
        +String commits
        +String files_changed
    }

    class VerifyMetadata {
        +String browser
        +String viewport
        +String ai_model
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

    CodeReviewResult "1" --> "*" Finding : critical_issues
    CodeReviewResult "1" --> "*" Finding : warnings
    CodeReviewResult "1" --> "*" Finding : suggestions
    CodeReviewResult "1" --> "1" ReviewMetadata : metadata

    VisualVerificationResult "1" --> "*" Verification : verifications
    VisualVerificationResult "1" --> "*" Screenshot : screenshots
    VisualVerificationResult "1" --> "1" VerifyMetadata : metadata

    CSVTestResult "1" --> "*" Verification : verifications
    CSVTestResult "1" --> "*" Screenshot : screenshots
    CSVTestResult "1" --> "1" CSVMetadata : metadata
```

**Storage paths:**

| Tool | Path |
|------|------|
| Code Review | `test-runs/{TICKET-ID}/review-{TICKET-ID}-vN.json` |
| Visual Verification | `test-runs/{TICKET-ID}/verify-{TICKET-ID}-vN.json` |
| Visual Verification screenshots | `test-runs/{TICKET-ID}/screenshots/vN/NN-description.png` |
| CSV Test Runner (per test) | `test-runs/{YYYY-MM-DD}/{customer}-{env}/results/{spec_id}/{spec_id}-result.json` |
| CSV Test Runner screenshots | `test-runs/{YYYY-MM-DD}/{customer}-{env}/screenshots/{spec_id}/` |
| CSV Test Runner summary | `test-runs/{YYYY-MM-DD}/{customer}-{env}/csv-run-summary.md` |

Results are always versioned (never overwritten), allowing multiple runs per ticket or test session to accumulate for audit and comparison purposes.
