# AI QA Automation Suite

An AI-powered testing and code review toolkit built on top of Claude Code and the Playwright MCP server. I built this independently at Intygrate to replace and extend our existing QA tooling — it started as an experiment and has grown into a suite of tools the team actively relies on.

The suite consists of three tools, each driven by a Claude Code slash command:

| Tool | Command | What it does |
|------|---------|--------------|
| [AI Code Review](./code-review.md) | `/run-jira-code-review` | Reviews a PR diff against Jira AC, coding standards, and security rules — posts a structured report to Jira |
| [AI Visual Verification](./visual-verification.md) | `/run-jira-visual-verification` | Reads a Jira ticket, navigates the running app in a browser, verifies each AC visually, and posts pass/fail results with screenshots to Jira |
| [CSV Test Runner](./csv-test-runner.md) | `/run-csv-tests` | Runs baseline regression tests directly from a structured CSV spreadsheet — no separate spec files required |

---

## System component map

All three tools run inside Claude Code on the developer's machine. They read from the local file system (configs, source repos, CSV files), drive a browser via the Playwright MCP server, and communicate with Jira and Bitbucket over HTTPS.

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

## Configuration cascade

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
1. `environment-config.env` — Jira credentials
2. `specs/proj/environment-config.env` — project credentials and admin URL
3. `specs/proj/environment-config.proj-qa.env` — QA base URL override

---

## Project structure

```
intygrate-ai-autotesting/
├── .claude/
│   ├── commands/
│   │   ├── run-jira-visual-verification.md
│   │   ├── run-jira-code-review.md
│   │   ├── run-csv-tests.md
│   │   ├── post-pr-summary.md
│   │   └── create-spec.md
│   └── mcp_settings.json
├── specs/
│   ├── cartalchemy-v1/
│   │   └── environment-config.env
│   └── cartalchemy-v2/
│       └── environment-config.env
├── test-runs/
│   └── {YYYY-MM-DD}/
│       └── {repo_name}/
│           ├── results/
│           └── screenshots/
├── source-repos/
│   ├── cartalchemy-v1/
│   └── cartalchemy-v2/
└── environment-config.env
```
