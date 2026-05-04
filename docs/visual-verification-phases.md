# AI Visual Verification Tool

## What We're Building

An AI-powered visual testing tool that reads a Jira ticket's acceptance criteria, navigates the running application in a browser, verifies each criterion directly in the UI, and captures screenshot evidence — all without pre-written test scripts.

The tool is driven entirely by the Jira ticket. The developer runs a single command (`/run-jira-visual-verification`) referencing a ticket ID, and the AI reads the AC, determines what to verify, navigates the app, and posts a structured verification result — including pass/fail per criterion and screenshots — back to the Jira ticket as a comment.

This is fundamentally different from traditional automated testing. There are no selectors to maintain, no brittle scripts tied to specific UI implementations, and no separate test suite to keep in sync with the codebase. The AI interprets what it sees in the same way a human tester would, and can handle variation in implementation as long as the acceptance criterion is met.

---

## What Was Built (Phase 1 Foundation)

The `/run-jira-visual-verification` command, which:

1. **Reads the Jira ticket** — fetches AC, background, description, and comments via the Jira REST API to understand what needs to be verified
2. **Discovers required test data** — presents a numbered list of any ticket-specific values it needs (customer names, order references, postcodes, etc.) and asks the tester to supply them before opening a browser. Does not ask for anything already in the environment config (login credentials, base URL, known references)
3. **Navigates the running app via Playwright** — logs in, navigates to the relevant areas, interacts with the UI to exercise the changes described in the ticket. Handles common browser popups automatically (cookie consent, password save dialogs)
4. **Verifies each acceptance criterion visually** — the AI interprets what it sees on screen and assesses whether each AC is satisfied
5. **Captures screenshots as evidence** at each verification point, saved to `test-runs/{date}/{repo}/screenshots/`
6. **Displays results in the terminal** before posting — shows a full verification summary (PASS / FAIL / PARTIAL / BLOCKED per criterion) and prompts for confirmation before posting to Jira
7. **Posts verification results to the Jira ticket** as a structured, clearly labelled comment

**Four result states** — `PASS`, `FAIL`, `PARTIAL`, `BLOCKED` — so partial verification and obstacles (missing test data, app unavailable, bot protection) are captured without failing the whole run.

### Project structure

```
intygrate-ai-autotesting/
├── .claude/
│   ├── commands/
│   │   ├── run-jira-visual-verification.md  ← /run-jira-visual-verification slash command
│   │   ├── run-jira-code-review.md          ← /run-jira-code-review slash command
│   │   ├── create-spec.md                   ← /create-spec slash command
│   │   ├── run-test.md                      ← /run-test slash command
│   │   └── run-all-tests.md                 ← /run-all-tests slash command
│   └── mcp_settings.json         ← Playwright + Atlassian + filesystem MCP config
├── specs/
│   ├── cartalchemy-v1/
│   │   ├── environment-config.env   ← repo-specific test credentials & URLs
│   │   └── TC-001-*.yaml            ← test specifications
│   └── cartalchemy-v2/
│       ├── environment-config.env
│       └── ...
├── test-runs/
│   └── {YYYY-MM-DD}/
│       └── {repo_name}/
│           ├── results/
│           │   └── verify-{TICKET-ID}-v1/
│           │       └── verify-{TICKET-ID}-v1-result.json
│           └── screenshots/
│               └── verify-{TICKET-ID}-v1/
│                   └── 01-{description}.png
├── source-repos/                 ← git-cloned source repositories
│   ├── cartalchemy-v1/
│   └── cartalchemy-v2/
└── environment-config.env        ← shared Jira credentials (JIRA_HOST, EMAIL, TOKEN)
```

### Current status

Active testing and refinement. Known areas being iterated on:
- Browser popup dismissal (cookie consent, password save dialogs)
- `allowedTools` configuration to reduce confirmation prompts during Playwright steps
- Handling proxy/connection errors from local dev server during test runs

**Planned improvements identified during testing:**
- **Negative test scenarios** — the current verification flow tests what *should* work. We need to extend it to also test what *shouldn't* work — e.g. that restricted actions are correctly blocked, that error states display as expected, or that validation rejects invalid input

---

## Phase 1: Developer-Run Verification (Current)

Any developer with a Claude Code account can run visual verification against their local dev environment at any point during development — not just as a gate at PR time.

### How it works

The developer runs from the `company-ai-autotesting` repo with source repos cloned into `source-repos/`:

```bash
cd ~/intygrate-ai-autotesting
claude

/run-jira-visual-verification cartalchemy-v1 MSSA-336
```

The tool connects to their locally running dev server (configured via `TEST_BASE_URL` in the environment config), runs the full verification, and posts results to the Jira ticket.

### When developers use it

- After implementing a feature, to verify AC are met before raising a PR
- After fixing a bug, to confirm the fix works as expected
- Mid-development to sense-check an approach before going further

Catching issues while the code is still being actively worked on is far cheaper than catching them in a formal review or after merge.

### Developer setup required

- Claude Code installed (requires Claude Pro subscription, ~$30/month AUD)
- Jira credentials configured in `environment-config.env`
- Source repo cloned into `source-repos/` and checked out to the correct branch
- Playwright MCP configured in `.claude/mcp_settings.json`

---

## Phase 2: Webhook-Triggered Verification on Dev Environments (Future)

### What this would be

Rather than requiring a developer to manually trigger the verification, Phase 2 automates it via webhooks. When code is deployed to a shared dev environment (e.g. `qa.company.co.uk`), a webhook fires and the verification runs automatically against that environment — no manual action required from the developer.

The result is that by the time a developer checks their deployed fix on the QA environment, the AI has already verified the AC and posted results to Jira.

### Architecture

```
Code deployed to qa.detanet.co.uk
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

### Cost model (AUD)

The server uses a single company-funded Claude Max account (flat subscription), not the pay-per-token API — so cost is predictable regardless of verification volume.

**Claude Max subscription tiers (approx. AUD):**

| Tier | USD price | Approx. AUD | Usage limit |
|---|---|---|---|
| Max 5× | $100/month | ~$150/month | 5× the usage of Pro |
| Max 20× | $200/month | ~$300/month | 20× the usage of Pro |

*(AUD approximate at ~1.5× USD — verify current pricing at claude.com/pricing)*

**Projected monthly cost:**

| Component | AUD/month |
|---|---|
| Claude Max 5× subscription | ~$150 |
| Server hosting | TBD |

### Considerations

**Stable target environment:**
The server needs a consistent, accessible URL to test against. `qa.company.co.uk` (or equivalent per-client QA environments) is the natural target — fixes go there before client delivery, so verifying against QA is exactly the right point in the workflow. No additional infrastructure is needed beyond what already exists.

**Claude Max terms of service:**
Claude Max is Anthropic's high-usage personal subscription. Automated server-side use may fall outside its intended use case. Before investing in this architecture, it's worth confirming with Anthropic whether running Claude Code in an automated pipeline on a shared account is permitted — or whether the Anthropic API would be the more appropriate path for server automation.

**Concurrency and queuing:**
If multiple deployments trigger simultaneously, verification jobs will queue. A simple job queue on the server handles this, but a verification may take a few minutes to start if other jobs are ahead of it.

**Webhook security:**
The server endpoint must validate webhook signatures to prevent unauthorised verification triggers. Standard practice but needs to be implemented correctly.

**Claude Code authentication on the server:**
Claude Code is designed for interactive developer use. Running it non-interactively in a server context requires the company Claude Max account to be authenticated in a way that persists without manual re-authentication on each run. Solvable, but adds setup complexity.

### When Phase 2 makes sense

Phase 2 is worth pursuing once:
1. Phase 1 (developer-run verification) is working reliably and being used consistently
2. The QA environment is stable and accessible from a hosted server
3. The team wants to remove the manual trigger step from the workflow
