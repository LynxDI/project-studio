# Changelog

## 0.4.0 — 2026-07-29

Milestone 4: work, review, time, and reports — plus release hardening.

- **Work items and a human review queue.** Create work items, link sessions and
  commits to them, and accept / request revision / reject an AI result. A
  review-ready session moves its item to *review*, never to *done*: only an
  explicit human acceptance completes work. Decisions are append-only audit rows.
- **Human vs AI time, kept separate.** Human active, AI execution, overlap, and
  autonomous intervals are computed from merged UTC intervals and labeled
  distinctly. Leverage reads N/A rather than infinity when the denominator is
  unavailable, and AI execution is never presented as human developer hours.
- **Deterministic daily report.** Generated locally with no model involved, and
  every claim carries the event ids behind it. Overnight work uses a printed,
  configurable window.
- **Project health.** A versioned rule artifact with six dimensions, evidence and
  thresholds on every result. Unknown dimensions never score as failures; fewer
  than three available reads *Not Calculated*.
- **Export and deletion.** Versioned JSON/CSV/Markdown export envelopes, and a
  two-step deletion that first shows an inventory of exactly what it will remove.
- New MCP tools and `pstudio work | review | time | health | report`.

### Security and packaging hardening

- Internal test commands no longer register in production installs — previously
  another installed extension could have invoked them to read your cross-project
  timeline or bypass the adoption gate.
- Agent-home and import settings are read from user scope only, so a cloned
  repository can no longer repoint the reader at a network path.
- Packaging now always rebuilds, and the extension identifier is
  `lynxdi.project-studio`.

## 0.3.0 — 2026-07-29

Milestone 3: every agent provider.

- Codex and Gemini providers (Experimental, Windows): local session history,
  lifecycle, subagent linkage, and token usage where the format exposes it —
  unknown stays unknown, never zero or estimated.
- generic-local provider: emit JSONL into the spool directory from any tool and
  it lands in the timeline.
- Provider Activity card + provider_activity MCP tool + pstudio compare:
  per-provider sessions, failures, tokens, and cost. Activity totals only —
  no composite ranking or quality score.
- Cross-provider reconciliation tests.

## 0.2.2 — 2026-07-29

Dogfood fix: the dashboard was empty at real scale.

- **Schema v3 performance fix.** With a real corpus (1,945 sessions / 61,583
  usage records) `v_sessions` took 100 seconds to resolve session projects and
  `v_project_cost_daily` never completed, so Home and the usage tiles rendered
  their empty state while the data sat correctly in the store. A composite
  index on `events(session_id, occurred_at_ms, id)` covers the subquery's
  filter *and* its ordering: 100,199 ms -> 85 ms; the cost view
  never-completed -> 767 ms. Cost ranking also moved from a per-row correlated
  subquery to a single window-function pass.
- Regression guards: the primary one asserts the **query plan** (scale-
  independent — at fixture scale the broken plan still answers in 9 ms), plus a
  multi-hundred-session timing budget.

## 0.2.1 — 2026-07-29

Prerelease compliance pass.

- LICENSE restated as an explicit Lynx DI commercial proprietary license.
- THIRD-PARTY-NOTICES.md added and shipped in the VSIX (bundled MIT/ISC/BSD
  components: zod; MCP server additionally bundles @modelcontextprotocol/sdk,
  ajv, ajv-formats, fast-deep-equal, fast-uri, json-schema-traverse,
  zod-to-json-schema).
- README updated to describe shipped Milestone 2 features accurately.

## 0.2.0 — 2026-07-29

Milestone 2: AI-agent visibility.

- Claude Code provider: local session history imports into the timeline —
  sessions, lifecycle states (incl. waiting-for-you vs waiting-for-tool),
  subagent linkage, token usage, and estimated cost (price-table versioned;
  unknown models stay unknown, never silently priced).
- AI Fleet view: active/waiting/recent/failed sessions with freshness labels.
- Usage & Cost: per-provider known tokens and estimated cost with explicit
  accounting disclosure; budgets with deduplicated threshold alerts.
- Unassigned queue view with one-click assign-to-project correction rules.
- Home: AI sessions + known-cost tiles, Fleet and Usage cards.
- New MCP tools (list_sessions, session_detail, usage_cost, fleet_status) and
  CLI commands (pstudio sessions/usage/fleet).
- Fix: collection level no longer written to .vscode/settings.json.

## 0.1.0 — 2026-07-28

Initial internal release (PRD Milestones 0–1).

- Project registry with adoption gate: fully dormant until explicit registration.
- Canonical SQLite event store (`node:sqlite`, WAL) with idempotent ingest,
  deterministic attribution (precedence + confidence + candidate queue), and
  correction rules.
- Providers: VS Code activity (focus, workspace, batched material file changes) and
  local Git (read-only, incremental by commit SHA, rebase-safe).
- Surfaces: status bar, Projects and Timeline tree views, Home webview
  (strict CSP, no external resources).
- Agent access: generated agent maps with ownership markers, pre-approved read-only
  query helper, local read-only MCP server, `pstudio` CLI.
- Verified: 178 unit tests, 1M-event benchmark (all filters < 200 ms p95),
  extension-host integration tests on VS Code 1.130, zero-egress sentinel.
