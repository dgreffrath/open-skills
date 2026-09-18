---
name: github-oss-ecosystem-improvements
description: "Produce a prioritized, review-ready improvement plan for an installed open-source tool by researching its GitHub ecosystem. Use when the user says 'search GitHub for things to add/improve X', 'how can I improve my bot/project/tool', 'what should I add next', or asks for N+ improvement ideas sourced from GitHub."
---

# GitHub OSS Ecosystem Improvement Research

Find and compile improvements (features, plugins, filters, dashboards, security fixes) for a specific installed open-source project by surveying its broader GitHub ecosystem, then deliver a prioritized review document full of concrete, verifiable sources.

The deliverable is a **review document** (e.g. `PROJECT_IMPROVEMENTS.md`) in the user's project folder — not unsolicited code changes.

## When to use

- Use case 1: User asks to "search GitHub" for improvements / additions to a specific project (shortcut: "30 improvements minimum").
- Use case 2: User wants feature ideas for an installed tool (bot, CLI, webapp, dashboard) sourced from real repos/docs.
- Use case 3: User wants a prioritized backlog to review before any code is written.

## Workflow

### 1. Identify the real project and its current state
- Find the actual project on disk (glob/ls for likely names). Only ask if genuinely ambiguous.
- Capture: exact version, existing config, enabled features, plugins/strategies, dashboards, runtime/monitoring scripts.
- Note what the config ALREADY has so recommendations do not duplicate (check `config.json`, strategy files, `user_data/`, logs).

### 2. Survey GitHub ecosystem
- Prefer GitHub MCP `search_repositories`. Use **short queries / `topic:` filters** — long multi-word queries often return 0 results (space-separated terms are ANDed and over-restrict).
- Follow with `search_issues` / `search_pull_requests` (merged features are strong "it works" evidence).
- WebFetch the official docs (`docs.<tool>.io`) for **built-in-but-not-enabled** features — usually the cheapest wins.
- Verify 2–3 high-value repos via `get_file_contents` before citing them.

### 3. Write the review document
- File: `<project_dir>/PROJECT_IMPROVEMENTS.md`.
- One section per category (built-in features, plugins/strategies, risk, monitoring, AI/ML, ops/security, adjacent tools).
- Each item: title, priority (🔴 Do first / 🟡 Strong value / 🟢 Nice to have), effort (S/M/L), why-it-matters-for-THIS-setup (reference their actual files), GitHub source + star count (or docs link), ready-to-paste config snippet where useful.
- Total items must meet or exceed the user's minimum (e.g. ≥30).
- End with an "execution order" table (quick wins → long-term).

### 4. Security rules
- NEVER paste credentials/keys/tokens into the doc, even if found in the user's config.
- If secrets sit in plaintext config, call it out as a 🔴 item (move to env vars + rotate) WITHOUT reproducing them.
- Research-only unless the user explicitly asks you to implement.

### 5. Definition of done
- Existing skill cover this? Was one incomplete/outdated? Did a reusable workflow emerge? Document any skill changes via this repo's contribution flow.

## Required tools / APIs

- GitHub CLI (`gh`) + GitHub MCP server: repository/search/issue tooling
- WebFetch / WebSearch: official docs and release notes
- Bash: version checks (`python -c "import X; print(X.__version__)"`, `git log`, `gh auth status`)

## Skills

### basic_usage

1. Locate the project + version.
2. Run grouped GitHub searches (short queries, `topic:` filters, PR/issues after).
3. Fetch docs for built-ins the instance hasn't enabled.
4. Write `PROJECT_IMPROVEMENTS.md` with N+ proven items + sources + priority + effort.
5. Add execution-order table; flag plaintext-secret items as 🔴.

**Example prompts/queries:**

```bash
# GitHub MCP
search_repositories(query="topic:freqtrade", sort="stars")
search_repositories(query="freqtrade dashboard", sort="stars")
search_issues(query="fee handling in backtest", owner="freqtrade", repo="freqtrade")

# docs
webfetch(url="https://docs.freqtrade.io/en/latest/plugins/")
```

**Node.js example (get the project version to scope recommendations):**

```javascript
const { execSync } = require('child_process');
console.log(execSync('python -c "import freqtrade; print(freqtrade.__version__)"').toString().trim());
```

## Checklist before finishing

- [ ] Items all link real repos/docs; star counts match at research time
- [ ] User's minimum item count met (offer surplus)
- [ ] No secrets in output; secret-in-config flagged without printing it
- [ ] Version-accurate (don't recommend a feature already disabled by their version)
- [ ] Execution-order table present