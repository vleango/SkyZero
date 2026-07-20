# SkyZero — Design Spec

**Date:** 2026-07-19
**Status:** Approved (brainstorming complete; this doc is the validated design)
**Mockup:** https://claude.ai/code/artifact/7e9358f0-5071-4976-99ff-7d06b1bb174e

## 1. What SkyZero is

SkyZero is a self-hosted agentic development platform: a web dashboard plus
autonomous Claude Code workers that implement tickets, refine prompts into
briefs, keep dependencies fresh, triage production errors, and babysit their
own PRs — across multiple companies' projects — with the human approving
every diff before anything is pushed.

The operator (single user) manages three working identities (bigcartel,
workwave, famoustitle), each with its own Claude subscription, git identity,
GitHub account, and issue tracker. SkyZero keeps those identities structurally
isolated while giving one unified queue and dashboard.

## 2. Goals and non-goals

**Goals (v1)**

- Launch agent tasks from tracker tickets or free-form prompts; batch tickets
  into one branch with one commit per ticket.
- Hard human gate: nothing is pushed until the user approves the diff in the
  dashboard.
- Full auditability: every task keeps a structured timeline and transcript,
  retained forever.
- Maintenance automations (dep updates, error triage, CI babysitting) as
  first-class task types through the same pipeline and gates.
- Runs on a laptop now, ports to a Synology DS920+ NAS (Celeron J4125,
  20GB RAM) with nothing but `docker compose up` plus credential provisioning.

**Non-goals (v1) — the someday list**

- Reviewing PRs assigned to the user (other people's code).
- Uptime/health checks of deployed apps (cut entirely).
- Notification channels beyond the dashboard (no email/Slack/push).
- OpenSearch/Grafana metrics export (Postgres events table covers v1).
- Multi-user support.

## 3. Deployment constraints

The end state is unattended operation on a Synology DS920+ (4-core Celeron
J4125, 20GB RAM, btrfs, Container Manager). That hardware is the design's
hard constraint from day one:

- Weak CPU means slow tasks are acceptable; disruption is not. Workers are
  cgroup-capped (~2 CPU / 4GB each) and concurrency-limited (§6).
- No JVM-weight services (OpenSearch rejected). No Redis (SolidQueue).
- Everything runs via docker-compose from the first commit. Porting to the
  NAS = `docker compose up` + provisioning credentials (`claude setup-token`
  per account, SSH keys, gh tokens).
- Development happens on the laptop for the first ~2 weeks (user away from
  the NAS until ~Aug 2).

## 4. Stack

**Rails 8 monolith + Hotwire + SolidQueue + Postgres**, everything in Docker.

Alternatives considered:

- **TS monorepo** (Next.js dashboard, Node workers, pg-boss) — the prior
  external recommendation. Rejected: its core argument was "the Agent SDK is
  TS-native," but workers are containerized Claude Code CLI running headless,
  so that advantage evaporates at the container boundary.
- **Light TS** (Hono + HTMX) — rejected: hand-rolls durable jobs, cron,
  and live updates that Rails 8 ships as batteries.

Why Rails 8 wins: SolidQueue gives durable jobs + cron with no Redis; Turbo
Streams give live dashboard updates; the app is fundamentally a CRUD monolith
around a state machine — Rails' home turf. Fewest moving parts on constrained
hardware. Postgres over SQLite for job-queue concurrency (and template
databases, §8). The user knowing Rails was a tiebreaker only.

## 5. Domain model: Accounts → Projects → Repos

**Account** = a working identity. Owns: a Claude subscription (long-lived
OAuth token via `claude setup-token`), a concurrency slot and quota, a git
identity, a GitHub account, and tracker credentials.

| Account | Git identity | Tracker | gh | Model |
|---|---|---|---|---|
| bigcartel | tha.leang@bigcartel.com | Linear | vleango | Opus 4.8 |
| workwave | tha.leang@workwave.com | Jira | tl562 | Opus 4.8 |
| famoustitle | vleango@gmail.com | GitHub Issues | vleango | Sonnet 5 |

**Project** = a deployable codebase group under an account, with its own
compose services, baseline environment, and agent-config layer.

- bigcartel → **compose-dev** (repos: admin, storefront)
- workwave → **commcenter** (repos: slingshot-web-app, slingshot-frontend),
  **slingshot-build-images** (Jira, same as commcenter). Default branch is
  `master`.
- famoustitle → **jpreader**, **famoustitle** (both GitHub Issues)

**Repo** = a git repository within a project. A task belongs to exactly one
project; the agent decides during analysis which repo(s) the work touches.
A multi-repo task uses one branch name across repos and produces one PR per
repo touched.

**Identity isolation is structural, not conventional:** a worker container is
provisioned with exactly one account's identity kit (OAuth token, git config,
SSH key, gh auth, tracker creds), and a preflight check verifies git
config/gh auth match the account before any work starts. Wrong-identity
commits are impossible by construction.

## 6. Concurrency

- **One slot per account** (3 total): an account's Claude subscription serves
  one running task at a time.
- **Machine cap**: 3 concurrent workers on the laptop, 2 on the NAS.
- Paused tasks (awaiting input or review) do **not** hold slots.
- The queue is account-aware: a queued bigcartel task waits for the bigcartel
  slot even if other slots are free.

## 7. Task lifecycle

### States

```
draft → queued → implementing → awaiting_review → preparing → publishing → done
                   ↑↓      ↖        ↓
             awaiting_input  ← revising
```

Plus **failed**, reachable from any running state (retryable; workspace kept
for inspection/resume). A failure during preparing or publishing also lands
in failed rather than silently completing.

- **awaiting_input** — mid-run, the agent hits a question only the user can
  answer. The session pauses, the question surfaces on the dashboard, the
  answer is appended to the task timeline, and the session resumes in the
  same worktree. Questions may be structured:
  `{question, options: [{label, description, recommended}], allow_free_text}`,
  rendered as a radio list. Agents are instructed to journal decisions and
  ask only when genuinely blocked on a user-visible call.
- **revising** — user requested changes at review; the agent continues in the
  same worktree with the feedback, then returns to awaiting_review.
- **failed** — retryable; the workspace is kept.

### Gates

Two human gates: **at launch** (drafts don't run until launched) and
**before push**. Nothing is ever pushed before the user approves the diff in
the dashboard — per-commit and combined views, with a git-fetch-from-the-NAS
escape hatch for local inspection. After approval:

1. **preparing** — lint, tests, cleanup (≈ `/prepare-pr`).
2. **publishing** — rebase, push, open PR(s) via the project's gh account,
   link tickets (≈ `/make-pr`).

### Batching

N tickets → one task → one branch, one commit per ticket.

### Timeline

Every task keeps a structured event timeline following the gz:timeline
contract (decisions with options + why, open items tracked to closure, honest
verification scope, current STATUS). The contract lives in the common
agent-config layer (§9); timelines are exportable as `timeline.md`. Timelines
and transcripts are retained forever in Postgres and readable from the task
page after completion.

## 8. Task environments

Per-task isolation with copy-on-write cheapness, built per project:

- **Bare mirror volume** cloned from origin. SkyZero **never** touches the
  user's local checkouts (`~/projects/*`, `~/worktrees/*`) — hard invariant.
- **Git worktree per task** off the mirror.
- **Gems / node_modules volumes** cloned from a golden baseline via
  `cp --reflink` (instant on btrfs).
- **Database** via Postgres template databases: each project has a pg
  container; a task DB is `CREATE DATABASE task_N TEMPLATE baseline`.
- **Baseline refresh** by a scheduled job, only when Gemfile.lock, yarn.lock,
  or schema change. A ticket that adds gems/migrations applies a +1 delta on
  top of the baseline inside its own task environment.
- **Task services** (declared per project: pg, redis, …) run on an isolated
  per-task Docker network with **zero published ports** — no web server, no
  vite, no Traefik.
- **Browser tests**: per-project mode — `sidecar` (standalone-chromium
  container), `in-image`, or `ci_only`. NAS default is `ci_only` (headless
  Chromium is too heavy there).

## 9. Agent configuration layering

Four layers, mounted as the worker's `~/.claude`, later layers winning:

1. **Common** — SkyZero base CLAUDE.md + shared skills: the timeline
   contract, commit conventions, the never-push rule.
2. **Per-project** — project CLAUDE.md + project skills.
3. **Repo's own CLAUDE.md** (whatever the repo checks in).
4. **Task** — the task's prompt/brief text.

Layers 1–2 live as git-tracked files in the SkyZero repo (`agent-config/`)
and are editable from the dashboard. Contents will be authored during
implementation; the mechanism and precedence are fixed now.

## 10. New Task composer

The composer is **company-first**: the top selector picks one of the three
companies (accounts), not a project.

### Mode A — From tickets

- Lists **all tickets assigned to me** at that company across its projects
  (famoustitle merges GitHub issues from jpreader + famoustitle); each row
  carries a project chip.
- Batch-select → one branch, one commit per ticket.
- A selection spanning multiple projects **auto-splits at launch into one
  task per project** (a task belongs to exactly one project). One launch
  click may therefore create N tasks.

### Mode B — From a prompt

- Free-form idea (jpreader-style feature work).
- An explicit **target-project select** (prompts carry no ticket metadata to
  infer the project from).
- A **required requirements-refinement chat** before Launch unlocks: it is
  repo-aware (read-only mirror checkout — no worker, no worktree), sees
  attachments, and produces a structured brief that serves as both ticket
  body and worker task brief.
- Optional **"file as ticket at launch"** — filed under the user's tracker
  identity, ungated since the content is user-approved. (Agent-drafted
  follow-up tickets arising elsewhere remain gated drafts.)

### Attachments

Mockups/docs attach to the requirements chat, flow into the filed ticket, and
are mounted read-only into the worker.

There is no free-text "extra instructions" box — standing guidance belongs in
the project CLAUDE.md layer (§9).

## 11. Maintenance flows

All are just task types flowing through the same state machine and gates:

- **Dependency updates** — scheduled; bump + test, then the normal review
  gate.
- **Error triage** — poll AppSignal (and later peers); investigate; output
  is either a fix branch (gated as usual) or a gated draft ticket.
- **CI babysitting** — for SkyZero-created PRs only. Red CI → a `ci_fix`
  task in the original worktree. Human review comments on the PR → drafted
  fixes/replies, gated. **No auto-rebase.**
- Code review in v1 = agent self-review only.

## 12. Dashboard UI

(The approved interactive mockup is the visual source of truth.)

**Sidebar:** a large "+ New task" button first, a gap, then Overview / Tasks /
Projects; per-account slot lines in the footer; Debug at the bottom.

**Overview page**, in order:

1. **Awaiting you** — questions + reviews as cards with uniform primary
   buttons and a one-line timeline trail. No inline Q&A — the card opens the
   task page for the full question with options.
2. **Agents working · N of 3 account slots · machine cap** — the running
   list.
3. **Project tiles** — baseline freshness, open PRs, last task (no uptime).
4. **Claude account quota cards** — one per account.
5. **Activity feed.**

**Tasks page:** Active table plus a Completed section grouped by company.
Completed rows show tickets and PR links (multi-repo:
web-app#412 + frontend#98) and open a read-only task page with the full
timeline and transcript.

**Debug page:** an `api_requests` table backed by a single instrumented HTTP
client (Faraday middleware; adapters cannot bypass it) with per-service
token-bucket throttles, 429 backoff (raising poller intervals), ETag/304
caching, caller attribution per request, per-service tiles showing
remaining-limit headers, and 14-day retention. Anthropic usage is shown as
sessions, not requests.

## 13. Security

- Secrets in Rails encrypted credentials.
- Dashboard binds to localhost/LAN only; Tailscale later for remote access.
- A token-scrubber pass over transcripts before storage/display.

## 14. Testing strategy

- Standard Rails testing for the app.
- **Adapter contract tests** for the tracker integrations (Linear, Jira,
  GitHub Issues): one shared contract exercised against each adapter.
- **E2E** against a fixture project and a scratch GitHub repo, exercising the
  full pipeline: launch → implement → review gate → publish.

## 15. Build order

Tenant onboarding order: **jpreader first** (lowest stakes), then bigcartel,
then workwave.

## 16. Resolved questions

- Cross-project ticket batches auto-split into one task per project
  (confirmed 2026-07-19).
- slingshot-build-images uses Jira (same as commcenter).
- Repo selection within a project is the agent's call at analysis time.
