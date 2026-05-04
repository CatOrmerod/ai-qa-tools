# AI Code Review

The code review tool replaced our existing Rovo Dev integration, which we were using for AI-assisted code review. I built a custom Claude-powered agent that does significantly more than a generic review — it analyses each PR diff against the specific Jira ticket's acceptance criteria, applies a structured set of coding standards and security rules, and produces a detailed, ticket-scoped report posted directly back to Jira as a comment.

The tool runs as a single slash command (`/run-jira-code-review`) from inside Claude Code. It reads the actual source code, fetches the Jira ticket, runs npm audit if package dependencies changed, and works through six sequential phases before producing a structured output with per-finding classifications and an AC traceability matrix.

For architecture in context with the other tools, see the [system component map](./README.md#system-component-map).

---

## Six-phase pipeline

The review runs six sequential phases. Phases 1–3 gather context (code diff, npm audit, Jira ticket). Phases 4–5 are AI analysis (standards review and AC coverage mapping). Phase 6 produces and optionally posts the result.

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

## Result data model

Reviews are saved as versioned JSON to `test-runs/{TICKET-ID}/review-{TICKET-ID}-vN.json`. Results are never overwritten — each run produces a new version, allowing multiple reviews per ticket to accumulate for audit and comparison.

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

    class Finding {
        +String file
        +String line
        +String rule
        +String description
        +String suggestion
    }

    class ReviewMetadata {
        +String commits
        +String files_changed
    }

    CodeReviewResult "1" --> "*" Finding : critical_issues
    CodeReviewResult "1" --> "*" Finding : warnings
    CodeReviewResult "1" --> "*" Finding : suggestions
    CodeReviewResult "1" --> "1" ReviewMetadata : metadata
```

---

## Planned improvements

**Risk-aware review overlay**

Extend the review to include a risk-sensitivity layer for high-impact areas of the codebase — payment pages, login and authentication flows, shared utility functions, or routes used across multiple modules. When a change touches one of these areas, the review would expand beyond the immediate ticket scope to assess broader interactions and downstream effects.

Still scoping whether this becomes a separate pass (`/run-jira-risk-review`) or an automatic additional check within the existing flow. Key design question: how to define "high risk" in a maintainable way — options include a curated list of paths in the repo config, file-level annotations, or Jira labels on tickets that touch sensitive areas.

**Automatic Bitbucket PR posting**

Enable `/post-pr-summary` to post directly to the Bitbucket pull request rather than requiring manual copy-paste. Currently blocked by SSO — Bitbucket uses separate authentication from Jira, and individual App Passwords are disabled when SSO is enforced at the workspace level.

Two paths to enable this:
- **Option A (recommended):** A Bitbucket workspace admin creates a service token not tied to any individual user account. SSO doesn't affect it. Stored in environment config alongside the Jira token. Permission needed: Pull Requests — Read & Write.
- **Option B:** A workspace admin enables App Passwords for SSO users, allowing each developer to create their own. More granular but requires per-developer setup.

Once either option is in place, a single config update (`BITBUCKET_ACCESS_TOKEN` in `.env`) is all that's needed — the command is already built and ready.

**In-repo deployment**

Roll out the tooling into individual repositories once it has been peer reviewed and tested, so developers can run commands directly from their own repo without a separate checkout.
