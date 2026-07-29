# Security

Project Studio measures how work actually happens, so it necessarily sits close to
sensitive material: your repositories, your editor activity, and your AI coding-agent
sessions. This page states plainly what it reads, what it stores, where that data lives,
what it refuses to do, and where the real boundaries are.

We have not undergone a third-party security audit or any formal certification, and we
do not claim one. What follows is the design and the controls that back it.

## Trust model

Project Studio is local-first. There is no account, no cloud service, and no telemetry.
Everything it produces stays on the machine that produced it.

**One security principal.** The MVP has exactly one: the operating-system user and
VS Code profile that owns the local database. Owner, assignee, and reviewer fields on
projects and work items are workflow metadata, not access control. There is no
multi-user permission system in v1.

**Zero network egress.** Project Studio v1 makes no outbound network connections at all.
This is enforced by an automated egress-sentinel test that traps outbound socket and
`fetch` activity across the full lifecycle — workspace registration, provider import,
querying, schema introspection, agent-map generation, and query-helper emission — and
fails the build if a single call occurs. The same test asserts that the query helper
emitted into your workspace contains no network imports. Stated honestly: a test running
inside the extension host cannot trap system calls made inside a separately spawned
process, so the helper's own execution is covered by that source assertion rather than by
a runtime trap.

**Dormant until you opt in.** Installing the extension collects nothing. Bare activation
writes nothing — no database, no files, no events. Collection begins only after you
explicitly register a workspace and pass the onboarding disclosure. Deleting the database
returns the extension to that first-run state, and the absence of a database never
triggers an automatic historical re-import.

**Local desktop only in v1.** In Remote SSH, WSL, dev-container, and browser-based
VS Code windows the extension detects the context, explains itself, and stays dormant. It
does not collect partial data in an environment where the timeline could not be trusted.

**What this does not mean.** No network egress is not encryption at rest, and it is not a
defense against software already running as you. Any process running under your OS user
can read the database file directly. Project Studio's answer to that is minimization, not
pretense: the default collection level stores counters and path metadata, never content
(see below). On Windows, POSIX-style file modes are not expressible; the database and its
side files inherit your user profile's directory permissions, which is the most
restrictive practical setting available. There is no encryption at rest in v1.

## What is read versus what is stored

Project Studio reads evidence your work already produces: VS Code activity, local Git
history via the `git` command line, and AI coding-agent session logs that the agents
themselves already write to disk (Claude Code, Codex, Gemini, plus a generic JSONL
file-drop). Reading is not storing. What is persisted is governed by a per-provider
collection level.

| Level | What is stored |
|---|---|
| 0 — Disabled | Nothing. The provider is not imported. |
| 1 — Identity and attribution | Session identity, provider and model names, timestamps, session state, and the evidence used to attribute work to a project. |
| 2 — Default | Level 1 plus usage counters (token and request totals) and normalized file-path and change metadata. |

Level 2 becomes the default only after the onboarding disclosure, which previews the
fields involved; you can choose Level 1 or Disabled before the first import, and change
levels later from settings.

**Never collected at any shipping level:**

- prompts and prompt text
- model responses and generated output
- file contents and diffs
- keystrokes
- screenshots
- clipboard contents

Secrets are never written to the database, to manifests, to logs, to diagnostics output,
or to exports. The v1 providers are entirely local and read-only and require no
credentials at all.

Untrusted strings that do enter the store — commit messages, branch and tag names, file
names — are stored as data, never as executable SQL or schema identifiers, and are
encoded on output.

## Where your data lives, and how to inspect or remove it

There is one SQLite database per VS Code profile, `portfolio.db`, in the extension's
global storage directory, alongside its write-ahead-log side files, provider checkpoints,
caches, rotated logs, and migration backups. Project folders themselves hold only files
you asked for: an approved project manifest, generated agent maps, the agent-access
helper directory, and exports.

**Inspect.** The database is an ordinary SQLite file with a documented, stable set of
`v_*` views. Open it with any SQLite client, the bundled read-only CLI, the MCP server's
`db_info` tool, or the read-only query helper. Nothing is obfuscated. The **Open
Diagnostics** command shows the extension's activity log, and after **Set Up Agent
Access** the helper directory in your workspace contains a plain-text pointer to the
database path.

**Export.** The database is yours; copying the file is a complete export today. Guided
in-product export, with schema versioning and collection-level-governed content, lands in
the next release.

**Delete.** Deleting the global-storage directory removes the portfolio database and all
machine-owned auxiliary data, and returns the extension to its dormant first-run state.
Deleting the `.project-studio` directory in a workspace removes the generated helper and
staged MCP server; generated agent maps and the project manifest are ordinary files you
can delete. A guided delete command, covering the full inventory — database and side
files, checkpoints, caches, migration backups, rotated logs, diagnostics, staged exports,
and event spools — lands in the next release. Project Studio never deletes an AI
provider's own session logs; those belong to that tool.

## The agent access surface, and its escalation boundary

Project Studio is designed to be read by your coding agent. That surface is deliberately
read-only end to end in v1: there is no write tool to hijack.

- **A bundled MCP server**, whose tools open the database with a read-only connection.
- **A pre-approved query command** (`.project-studio/bin/query.mjs`), so an agent can ask
  a precise question without a permission prompt for every call.
- **Generated project maps** (`AGENTS.md`, `CLAUDE.md`, `GEMINI.md`) and a **CLI**.

Every result carries provenance and freshness labels so an agent's claims stay auditable.

**The boundary, stated plainly.** A pre-approved command is a convenience, and
conveniences have a cost: any agent that also holds file-write permission in your
workspace can alter what that pre-approved entry runs, and then invoke it without a
prompt appearing. The file that tells the helper which database to open sits at the same
repository trust level, so an agent that can rewrite it can point the helper at another
SQLite file your OS user can read. Treat workspace write permission as what it already
is — code-execution capability, the same way `package.json` scripts and Git hooks already
are — and grant it accordingly.

What bounds the blast radius:

- the database connection is opened **read-only at the SQLite level**, so writes are
  rejected by the engine, not merely by our code;
- a **statement guard** admits only `SELECT`/`WITH` queries and denies `ATTACH`,
  `DETACH`, `PRAGMA`, `VACUUM`, `REINDEX`, and `ANALYZE`;
- a **row cap** limits every call to 500 rows, with JSON-only output and an explicit
  truncation flag;
- **machine-owned files are rewritten to canonical content on every regeneration cycle**,
  so tampering with the helper or its database pointer survives at most one cycle;
- the helper directory is **git-ignored by default**, so a tampered copy does not
  propagate to anyone who clones the repository;
- **agent permission files are written create-if-absent only** — existing permissions are
  never merged into, widened, or rewritten;
- **generated maps are never written over a file that is not ours**: a map without our
  ownership marker is left untouched, and a marked section is preserved for your own
  notes;
- what is readable at all is limited by the collection level — at the default, metadata
  and counters, not content.

An agent following hostile instructions can still exfiltrate whatever it is able to read,
through its own channels. Project Studio cannot firewall another tool's network access.
The control that matters there is the one above: keep the stored data minimal.

## Webview hardening

Project Studio's dashboard is a VS Code webview, hardened accordingly:

- a strict Content Security Policy that starts from `default-src 'none'`;
- a fresh nonce generated per render for local scripts and styles; no `unsafe-inline`,
  no `unsafe-eval`;
- explicit resource roots and vendored assets only — no remote scripts, styles, fonts, or
  images, and no external resource can be loaded at all;
- a versioned, schema-validated, allow-listed message envelope in both directions;
- the webview receives view-model data only. It holds no secrets, no filesystem access,
  and no ability to issue SQL.

## Untrusted input

Repository content and provider log files are treated as hostile input, because anything
that can land in a repository or a log line can reach the parser.

- Provider inputs declare bounded limits — maximum file size, line size, payload size,
  batch size, recursion depth, and decompression ratio. Oversized input fails soft with a
  diagnostic rather than consuming the extension host.
- A malformed record is quarantined with a diagnostic; its valid siblings still import.
  Malformed-input fixtures are a gate an adapter must pass before it is considered
  supported.
- Every SQL statement is parameterized; provider payloads never become SQL text or schema
  identifiers.
- Subprocesses are invoked with an executable and an argument array and an explicit
  working directory — never a shell string — so a hostile branch or file name cannot
  become a command. Git is the only subprocess, and it runs with terminal prompting
  disabled.
- All provider-supplied paths are canonicalized and checked against declared roots before
  use, with symlinks, junctions, UNC paths, case variants, traversal sequences, and
  multi-root layouts covered by a required test matrix.
- Workspace-scoped settings are never trusted to decide policy: trust-affecting settings
  are read from user scope only, and a cloned repository cannot enable collection,
  repoint the database, or widen agent permissions by shipping a `.vscode` configuration.
- Only one coordinated writer holds the database at a time, protected by a heartbeat lease
  with atomic takeover; read-only surfaces never take the writer role.

## Reporting a vulnerability

Report suspected vulnerabilities at
<https://github.com/LynxDI/project-studio/issues>.

Please do not include sensitive data in a public issue — no database files, no session
logs, no repository paths, no credentials, no screenshots containing customer or employer
information. A description of the behavior, the affected version, and the platform is
enough to start; we will arrange a private channel before you send anything sensitive.

If you believe an issue is actively exploitable, say so in the first line and hold the
reproduction details until we reply.
