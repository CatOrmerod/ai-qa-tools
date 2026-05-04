# AI Visual Verification

I built this tool to replace the manual AC verification step that typically happens at the end of a development ticket. Rather than a developer or tester manually clicking through the app to check each acceptance criterion, the tool reads the Jira ticket, navigates the running application in a real browser, and verifies each AC directly in the UI — then posts a structured pass/fail result with screenshot evidence back to the Jira ticket.

This is fundamentally different from traditional automated testing. There are no selectors to maintain, no brittle scripts tied to specific UI implementations, and no separate test suite to keep in sync with the codebase. The AI interprets what it sees the same way a human tester would, and can handle variation in implementation as long as the acceptance criterion is met.

The tool is driven entirely by the Jira ticket. A developer runs a single command (`/run-jira-visual-verification`) referencing a ticket ID — the AI reads the AC, determines what to verify, navigates the app, and posts results back to Jira. No setup per ticket. No test scripts to write.

For architecture in context with the other tools, see the [system component map](./README.md#system-component-map).

---

## Interaction sequence

The visual verification tool is inherently interactive — Claude Code and the Playwright MCP server trade messages in a tight loop while navigating the application. This sequence diagram shows the full flow from command invocation to Jira posting.

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

## Result states

Four result states are used so partial verification and obstacles are captured accurately without failing the whole run:

| State | Meaning |
|-------|---------|
| `PASS` | AC is fully satisfied as observed in the UI |
| `FAIL` | AC is not met |
| `PARTIAL` | AC is partially met — some conditions satisfied, others not |
| `BLOCKED` | Verification could not be completed (missing test data, app unavailable, bot protection) |

---

## Result data model

Verification results are saved as versioned JSON. Results are never overwritten — each run produces a new version, allowing multiple verifications per ticket to accumulate.

```mermaid
classDiagram
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

    class VerifyMetadata {
        +String browser
        +String viewport
        +String ai_model
    }

    VisualVerificationResult "1" --> "*" Verification : verifications
    VisualVerificationResult "1" --> "*" Screenshot : screenshots
    VisualVerificationResult "1" --> "1" VerifyMetadata : metadata
```

**Storage paths:**

| Artifact | Path |
|----------|------|
| Result JSON | `test-runs/{TICKET-ID}/verify-{TICKET-ID}-vN.json` |
| Screenshots | `test-runs/{TICKET-ID}/screenshots/vN/NN-description.png` |

---

## Current status and known issues

Active testing and refinement. Known areas being iterated on:

- Browser popup dismissal (cookie consent, password save dialogs)
- `allowedTools` configuration to reduce confirmation prompts during Playwright steps
- Handling proxy/connection errors from local dev server during test runs

Planned improvements identified during testing:

- **Negative test scenarios** — the current verification flow tests what should work. Needs extending to also test what shouldn't — restricted actions correctly blocked, error states displaying as expected, validation rejecting invalid input.

---

## Phase 2 — Webhook-triggered verification (planned)

Rather than requiring a developer to manually trigger verification, Phase 2 would automate it via webhooks. When code is deployed to a shared environment, a webhook fires and the verification runs automatically — no manual action required.

```
Code deployed to qa.company.co.uk
        ↓
Deployment webhook → Company server (HTTPS endpoint)
        ↓
Server validates webhook signature
        ↓
Queues verification job (extracts ticket ID from branch name)
        ↓
Claude Code CLI runs /run-jira-visual-verification
  ├── Fetches Jira ticket AC
  ├── Navigates qa.company.co.uk via Playwright
  ├── Verifies each acceptance criterion visually
  ├── Captures screenshots as evidence
  └── Posts full results to Jira ticket
        ↓
Developer sees verification results in Jira automatically
```


