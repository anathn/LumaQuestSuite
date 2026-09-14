# LumaQuest Suite — Epics (Reading Quest Architecture Delivery)

**Source documents:** `docs/Reading_Quest_Architecture.md` (system design, ADR-001–ADR-016), `docs/Reading_Quest_PRD_and_Delivery_Plan.md` (product requirements RR-1–RR-15, epics R1–R11), `docs/Spelling_Quest_PRD_and_Delivery_Plan.md` (existing platform context).

**How this document was built:** The architecture doc describes a modular monolith with a shared platform layer (accounts, tenancy, security, containers, data, observability) plus a Reading Quest domain layer built on top of it (ADR-002, ADR-008). This breakdown mirrors that split: `EPIC-001`–`EPIC-003` are shared-platform infrastructure that must exist before any domain feature can ship; `EPIC-004`–`EPIC-012` are the Reading Quest domain module, one epic per architecture component (`packages/reading-engine`, `packages/reading-aligner`, `packages/content-generation`, etc.) and cross-referenced to the PRD's own `R1`–`R11` epic numbers; `EPIC-013`–`EPIC-014` are cross-cutting operational and release-readiness work described in Architecture §6–§9 and §12.

Every story below is scoped to be implementable by an AI developer without further clarification: it names concrete files/packages from the architecture's component table (§3.1) and repository layout (PRD §20), and cites the architecture section or ADR that constrains its design.

## Epic Index

| ID | Name | Priority | Status | Points | Target Sprint |
|----|------|----------|--------|--------|---------------|
| EPIC-001 | Platform Foundation & Containerized Infrastructure | P0 | ready-for-dev | 19 | Sprint 1 |
| EPIC-002 | Accounts, Tenancy & Security | P0 | ready-for-dev | 18 | Sprint 1 |
| EPIC-003 | Shared Data Platform & Badge Engine | P0 | planning | 14 | Sprint 2 |
| EPIC-004 | Reading Content & Approval Workflow | P0 | planning | 16 | Sprint 2 |
| EPIC-005 | Reading Child Session Experience | P0 | planning | 16 | Sprint 3 |
| EPIC-006 | Speech Alignment Pipeline | P0 | planning | 31 | Sprint 3 |
| EPIC-007 | Hints & Difficult-Word Practice | P0 | planning | 13 | Sprint 3 |
| EPIC-008 | Comprehension Engine | P0 | planning | 14 | Sprint 4 |
| EPIC-009 | Rereading & Progression Scheduling | P1 | planning | 13 | Sprint 4 |
| EPIC-010 | Reading Progress Dashboard | P0 | planning | 13 | Sprint 4 |
| EPIC-011 | Reading Badges Integration | P1 | planning | 11 | Sprint 4 |
| EPIC-012 | AI-Assisted Content Generation | P1 | planning | 23 | Sprint 5 |
| EPIC-013 | Observability, Reliability & Deployment Pipeline | P1 | planning | 17 | Sprint 5 |
| EPIC-014 | Privacy, Accessibility & Release Readiness | P0 | planning | 20 | Sprint 5 |

**Total estimated points:** 238

---

## EPIC-001: Platform Foundation & Containerized Infrastructure

**Priority:** P0 (Must Have)
**Status:** ready-for-dev
**Progress:** 0%
**Estimated Points:** 19
**Target Sprint:** Sprint 1

### Description

Stand up the container-first runtime described in Architecture §6.1–§6.4 and ADR-015: every component (`web`, `api`, `worker`, `postgres`, `redis`, `minio`, `proxy`) ships as an OCI image and runs under Docker Compose, with `dev`/`staging`/`prod` as structurally identical stacks.

### Goal

A developer can run `docker compose up` and get a working, networked stack (proxy, web, api, worker, postgres, redis, minio) on a laptop, and the same Compose definitions deploy to staging/prod with only environment variables and image tags changing.

### Scope

**Included:**
- Dockerfiles for `apps/web`, `apps/api`, `worker` (Celery entrypoint sharing the API image or a slim variant)
- `docker-compose.yml` (base) plus per-environment overrides for `dev`/`staging`/`prod`
- Traefik reverse proxy: TLS via ACME, routing to `web`/`api`, security headers
- Private Docker network per environment; only `proxy` publishes host ports
- CI pipeline: build images on merge to `main`, tag with commit SHA, push to a registry (GHCR)
- Deploy step: `docker compose pull && docker compose up -d`, migrations as a one-off `alembic upgrade head` container run before app containers start

**Excluded:**
- Multi-host orchestration (Swarm/Kubernetes) — deferred per §12.1
- Self-hosted STT/LLM containers (`stt`, `llm`) — covered in EPIC-006/EPIC-012
- Managed Postgres migration path — future consideration only

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-001-01 | Monorepo scaffold and container images for web/api/worker | ready-for-dev | 5 |
| STORY-001-02 | Postgres, Redis, and MinIO containers on a private Compose network | ready-for-dev | 3 |
| STORY-001-03 | Traefik reverse proxy with TLS and security headers | ready-for-dev | 3 |
| STORY-001-04 | CI pipeline: build, tag, and push container images | ready-for-dev | 5 |
| STORY-001-05 | Per-environment Compose stacks (dev/staging/prod) with isolated secrets | ready-for-dev | 3 |

### Dependencies

None (foundational epic).

### Acceptance Criteria

- `docker compose up` from a clean checkout brings up all containers healthy on a laptop.
- `postgres`, `redis`, `minio` are unreachable from outside the Docker network (verified by a port-scan test against the host).
- CI produces SHA-tagged images for `web`, `api`, `worker` on every merge to `main`.
- Staging and prod use the same Compose files as dev, differing only in environment/override files.

### Risks & Blockers

- None currently. Single-host deployment is an accepted MVP trade-off (Architecture §8.1); do not attempt HA here.

---

## EPIC-002: Accounts, Tenancy & Security

**Priority:** P0 (Must Have)
**Status:** ready-for-dev
**Progress:** 0%
**Estimated Points:** 18
**Target Sprint:** Sprint 1

### Description

Implement the shared `accounts` module (Architecture §3.1, §5.1, ADR-006): parent-owned accounts, child sub-profiles, and the `family_id` tenancy scope enforced on every query. This is the security boundary every other module depends on.

### Goal

No handler in the system can read or write another family's data, and children never hold independent credentials.

### Scope

**Included:**
- `Family` and `ChildProfile` models and Alembic migrations (§4.2)
- Parent authentication: email + password (Argon2id) and magic-link session creation (`POST /api/v1/auth/session`)
- Redis-backed opaque session store behind an httpOnly, secure, `SameSite=Lax` cookie (§3.2, §5.1)
- Shared FastAPI dependency that resolves and injects `family_id` on every request, with no handler allowed to bypass it
- Child profile listing/creation/selection within an authenticated parent session (`GET/POST /api/v1/children`)
- Secrets management: DB credentials, Redis URL, session signing key injected via Compose secrets/env files, never baked into images (§5.5)

**Excluded:**
- Child-specific credentials or login (explicitly out of scope per ADR-006)
- Reading/spelling domain permissions and feature flags (covered in EPIC-004/ADR-008)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-002-01 | Family and ChildProfile models with Alembic migration | ready-for-dev | 3 |
| STORY-002-02 | Parent authentication: Argon2 password + magic link, Redis session cookie | ready-for-dev | 5 |
| STORY-002-03 | Tenancy-enforcement dependency and cross-family isolation tests | ready-for-dev | 5 |
| STORY-002-04 | Child profile CRUD and in-session child selection | ready-for-dev | 3 |
| STORY-002-05 | Secrets injection via Compose secrets/env files (no secrets in images/repo) | ready-for-dev | 2 |

### Dependencies

- EPIC-001 (needs `postgres`, `redis`, and the `api` container running).

### Acceptance Criteria

- Every table carrying family- or child-specific data is reachable only through the tenancy dependency (verified by an integration test suite per §5.1).
- A parent session cookie is httpOnly, secure, and revocable server-side within one request cycle of logout/password change.
- No credential material appears in any Dockerfile, image layer, or committed file.

### Risks & Blockers

- None currently.

---

## EPIC-003: Shared Data Platform & Badge Engine

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 14
**Target Sprint:** Sprint 2

### Description

Build the platform-wide reward and caching primitives described in Architecture §3.1 (`packages/badge-engine`) and §4.3, shared by both Spelling Quest and Reading Quest per ADR-013.

### Goal

One badge catalog and evaluator serves both games' events; today's-plan lookups are fast and cache-invalidated correctly.

### Scope

**Included:**
- `RewardLedger`, `BadgeDefinition`, `ChildBadge` models and migrations (§4.2)
- Badge evaluator: idempotent awarding keyed by `(child_id, badge_id)`, criteria evaluation against `event_type`
- Badge catalog and per-child badge endpoints (`GET /api/v1/badges`, `GET /api/v1/children/{id}/badges`)
- Redis caching for the badge catalog and for "today's plan" keyed by `child_id` + date, invalidated on session completion or reassignment (§4.3)

**Excluded:**
- Reading-specific badge catalog content (EPIC-011)
- Spelling-specific event wiring (assumed to exist from the prior Spelling Quest build)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-003-01 | RewardLedger, BadgeDefinition, ChildBadge models and migrations | planning | 3 |
| STORY-003-02 | Idempotent badge evaluator service | planning | 5 |
| STORY-003-03 | Badge catalog and per-child badge API with Redis cache | planning | 3 |
| STORY-003-04 | Today's-plan Redis cache keyed by child_id + date | planning | 3 |

### Dependencies

- EPIC-002 (tenancy dependency; badge queries are family/child scoped).

### Acceptance Criteria

- The same `(child_id, badge_id)` event never produces two `ChildBadge` rows, under concurrent evaluation.
- Badge catalog reads hit Redis, not Postgres, after the first request per deploy version.
- Today's-plan cache is invalidated within the same request that completes a session or reassigns content.

### Risks & Blockers

- None currently.

---

## EPIC-004: Reading Content & Approval Workflow

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 16
**Target Sprint:** Sprint 2

*Maps to PRD epic R2 and functional requirement RR-2.*

### Description

Implement the `content` module's non-AI path (Architecture §3.1, §3.4, ADR-011): parents create, edit, version, approve, and assign reading material. Approved `ReadingVersion` rows are immutable (ADR-007).

### Goal

A parent can paste or write a word list, sentence, or passage; preview it; approve it; and assign it to a child, with every child session reading from a frozen, versioned snapshot.

### Scope

**Included:**
- `ReadingItem`, `ReadingVersion`, `TargetWord`, `Question` models and migrations (§4.2, PRD §12)
- CRUD for reading items (word sets, sentences, passages) with `draft` → `approved` → `assigned` → `archived` states (PRD Epic R2)
- Freeze-on-approve endpoint (`POST /api/v1/reading-versions/{id}/approve`) that makes a version immutable; editing approved content creates a new version, never an in-place mutation (ADR-007)
- Reading assignment endpoint (`POST /api/v1/reading-assignments`) linking a child to an approved version
- Reuse/duplication of existing items for a new assignment

**Excluded:**
- LLM-assisted drafting (EPIC-012)
- Physical book tracking / `BookLog` (bundled into EPIC-004 backlog for a later sprint if prioritized; not in Sprint 2 scope)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-004-01 | ReadingItem/ReadingVersion/TargetWord/Question models and migrations | planning | 5 |
| STORY-004-02 | Reading item CRUD API (create/edit/archive word sets, sentences, passages) | planning | 5 |
| STORY-004-03 | Version freeze-and-approve endpoint with immutability guarantee | planning | 3 |
| STORY-004-04 | Reading assignment API (assign approved version to a child) | planning | 3 |

### Dependencies

- EPIC-002 (tenancy — `ReadingItem.family_id`).
- EPIC-003 not required, but assignment completion will later feed the badge ledger (EPIC-011).

### Acceptance Criteria

- A parent can paste, preview, approve, assign, and reopen (duplicate) a passage end-to-end via the API.
- Attempting to mutate an approved `ReadingVersion` fails; editing produces a new version row instead.
- Every `ReadingAssignment` references exactly one approved `ReadingVersion`.

### Risks & Blockers

- None currently.

---

## EPIC-005: Reading Child Session Experience

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 16
**Target Sprint:** Sprint 3

*Maps to PRD epic R3 and functional requirement RR-3, RR-5. Corresponds to the PRD's recommended first milestone (§20): a complete manual-mode session before speech is added.*

### Description

Build the child-facing reading session shell in `apps/web` and the session lifecycle in `apps/api`/`packages/reading-engine`: preview, supported read, meaning check, completion — usable with zero microphone access (RR-3, Architecture §3.1).

### Goal

A child can complete an entire reading session — preview, read, answer questions, finish — without a working microphone, and a page refresh never loses confirmed progress.

### Scope

**Included:**
- `ReadingSession` model and migration (§4.2)
- Session lifecycle API: `POST /api/v1/reading-sessions`, `POST /api/v1/reading-sessions/{id}/complete`
- `GET /api/v1/children/{id}/reading/today` — today's reading plan
- Child reading home screen and session shell (`apps/web`)
- Focus mode and display settings: single-line/sentence view, adjustable font/line spacing, reading ruler, dyslexia-friendly typeface option, reduced motion, high contrast without color-only meaning (PRD §6)
- Manual/parent-assisted reading mode: no audio capture required, parent marks the read as complete/observed

**Excluded:**
- Microphone capture and speech alignment (EPIC-006)
- Hint ladder mechanics (EPIC-007) — session shell should have a slot for hints but not implement scoring
- Comprehension question rendering logic (EPIC-008) — session shell should have a slot for the meaning-check stage

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-005-01 | ReadingSession lifecycle API (start/complete) | planning | 3 |
| STORY-005-02 | Today's reading plan endpoint and child home screen | planning | 3 |
| STORY-005-03 | Focus mode and accessibility display settings | planning | 5 |
| STORY-005-04 | Manual/parent-assisted reading mode (no microphone required) | planning | 5 |

### Dependencies

- EPIC-004 (needs an approved `ReadingVersion` to assign and read).
- EPIC-002 (child selection, tenancy).

### Acceptance Criteria

- A full session (start → preview → read → complete) works with microphone permission denied or unavailable.
- Text scaling and line-focus settings render without clipping or horizontal scrolling at any supported viewport width.
- Refreshing mid-session resumes from the last confirmed step, not from the beginning.

### Risks & Blockers

- None currently.

---

## EPIC-006: Speech Alignment Pipeline

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 31
**Target Sprint:** Sprint 3

*Maps to PRD epic R4 and functional requirement RR-6. Governed by ADR-004 (pluggable speech provider), ADR-005/ADR-014 (no raw audio retention by default), and ADR-009 (provisional results).*

### Description

Implement `packages/speech-adapter` and `packages/reading-aligner` (Architecture §3.1, §3.3): capture short audio segments, transcribe them, align the transcript to the expected passage text, classify the result, and route low-confidence spans to a confirmation step instead of auto-scoring them as errors.

### Goal

Spoken reading produces confidence-aware evidence — never a silent false failure — and the system keeps working when speech services are degraded or unavailable.

### Scope

**Included:**
- `speech-adapter` interface with two implementations: self-hosted `faster-whisper` container and a hosted STT API, selected by environment configuration (ADR-004)
- `ReadingEvent` model and migration, including `token_position`, `event_type`, `observed_text`, `confidence`, `hint_level` (§4.2)
- Alignment engine classifying `correct`, `omission`, `substitution`, `insertion`, `self_correction`, `help_requested`, `speech_uncertain`, `parent_corrected` (PRD §7)
- Confidence-threshold gate: low-confidence spans require child/parent confirmation before counting as an error (ADR-009)
- `POST /api/v1/reading-sessions/{id}/audio` (transcribe + align a short segment) and `POST /api/v1/reading-sessions/{id}/events` (confirm/correct)
- Audio handling: short-lived, encrypted MinIO objects with an automatic deletion lifecycle rule; no raw audio retained by default (ADR-005, ADR-014)
- Correction preserves the original machine-generated event as a separate `parent_corrected` row (ADR-007) rather than mutating it

**Excluded:**
- Opt-in long-term audio retention for parent comparison (EPIC-014 — separate consent/storage/deletion flow per ADR-014)
- Hint-ladder scoring logic that consumes `hint_level` (EPIC-007)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-006-01 | Speech adapter interface and self-hosted faster-whisper implementation | planning | 8 |
| STORY-006-02 | Hosted STT adapter implementation, selectable via configuration | planning | 5 |
| STORY-006-03 | Transcript-to-passage alignment engine with event classification | planning | 8 |
| STORY-006-04 | Confidence thresholds and child/parent confirmation flow | planning | 5 |
| STORY-006-05 | Segment audio capture/upload with short-lived encrypted MinIO storage and auto-delete | planning | 5 |

### Dependencies

- EPIC-005 (session shell to embed capture/confirmation UI into).
- EPIC-001 (MinIO container, optional `stt` container).

### Acceptance Criteria

- Alignment tests cover repeated words, skipped lines, restarts, and self-corrections (§11.3, Architecture §5.3).
- No low-confidence span is ever auto-scored as an error without a confirmation step.
- A parent correction never deletes or overwrites the original machine-generated event row.
- Audio objects in MinIO are deleted automatically per the configured lifecycle rule; a test verifies none persist past it.
- Switching the speech adapter implementation is a configuration change with no domain-logic code change (ADR-004).

### Risks & Blockers

- Child speech recognition accuracy is a known technical risk (Architecture §11.1); mitigated by confidence thresholds and the manual fallback mode in EPIC-005.

---

## EPIC-007: Hints & Difficult-Word Practice

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 13
**Target Sprint:** Sprint 3

*Maps to PRD epic R5 and functional requirement RR-7. Governed by ADR-012 (hint level is part of mastery evidence).*

### Description

Implement the six-level hint ladder (PRD §3) inside `packages/reading-engine`, and use confirmed `ReadingEvent`s to schedule later practice of words a child struggled with.

### Goal

Children get progressively stronger help instead of an immediate answer, and words that needed help come back for review — while words fully supplied to the child never count as independently read.

### Scope

**Included:**
- Hint ladder state machine: wait/encourage → highlight first letter/part → play first sound/syllable → syllable/grapheme breakdown → speak full word → repeat-and-reread (PRD §3)
- `hint_level` recorded on every relevant `ReadingEvent`
- Difficult-word extraction from confirmed events (words needing hint level 4+ or repeated misses)
- Scheduling difficult words into a later session's warm-up/contextual retry (PRD §6 step 2, step 10)

**Excluded:**
- Full reread-of-passage scheduling (EPIC-009) — this epic covers word-level, not passage-level, review

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-007-01 | Six-level hint ladder engine with hint_level tracking on ReadingEvent | planning | 5 |
| STORY-007-02 | Difficult-word extraction and scheduling for later review | planning | 5 |
| STORY-007-03 | Contextual retry of previously-hinted words in a later session | planning | 3 |

### Dependencies

- EPIC-006 (`ReadingEvent` model and confirmation flow must exist).

### Acceptance Criteria

- A word supplied via the final hint level is never counted as independently read (ADR-012), verified by a unit test on the mastery-evidence calculation.
- A word confirmed difficult in one session appears in a warm-up or contextual retry within a later session.
- Decreasing hint-level use over time on the same word is observable in the stored evidence (feeds EPIC-010).

### Risks & Blockers

- Hints creating dependence is a named product risk (PRD §19); mitigated by the graduated ladder and tracked support level.

---

## EPIC-008: Comprehension Engine

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 14
**Target Sprint:** Sprint 4

*Maps to PRD epic R6 and functional requirement RR-8.*

### Description

Build the comprehension question engine (Architecture §3.1) covering literal, sequence, vocabulary, main-idea, and simple-inference question types, all approved alongside the passage (ADR-011).

### Goal

The game verifies understanding, not just reading-aloud ability, using deterministic scoring wherever the question type allows it.

### Scope

**Included:**
- `Question` (already modeled in EPIC-004) and `QuestionAttempt` models/migration
- `POST /api/v1/reading-sessions/{id}/answers` endpoint recording responses and deterministic results
- Support for literal, sequence, vocabulary-in-context, main-idea, and multiple-choice inference question types (PRD §8)
- Incorrect-answer remediation: point the child back to the relevant `source_span` before revealing the answer
- Parent-reviewed retell mode: free-text/audio response stored for parent review, no automated high-stakes score (PRD §8, ADR-011 spirit — no unreviewed automated judgment reaching the child)

**Excluded:**
- Open-response inference scoring by an LLM (explicitly excluded by PRD §8 — "no high-stakes LLM score")

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-008-01 | Question and QuestionAttempt models and migration | planning | 3 |
| STORY-008-02 | Comprehension answer API with deterministic scoring by question type | planning | 5 |
| STORY-008-03 | Incorrect-answer remediation linking back to source span | planning | 3 |
| STORY-008-04 | Parent-reviewed retell mode (no automated high-stakes score) | planning | 3 |

### Dependencies

- EPIC-004 (`Question` model, approved alongside `ReadingVersion`).
- EPIC-005 (meaning-check stage slot in the session shell).

### Acceptance Criteria

- Every automatic (non-retell) question type resolves to exactly one parent-approved expected answer.
- An incorrect response returns the `source_span` reference before any answer reveal.
- Retell responses are stored and surfaced for parent review but never produce an automated pass/fail score.

### Risks & Blockers

- None currently.

---

## EPIC-009: Rereading & Progression Scheduling

**Priority:** P1 (Should Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 13
**Target Sprint:** Sprint 4

*Maps to PRD epic R7 and functional requirement RR-9. Governed by ADR-010 (fluency is not reading speed).*

### Description

Implement the reread scheduler (Architecture §3.1, run via the Celery worker per §7.1) that brings a passage back for a later attempt and compares performance without reducing fluency to words-per-minute.

### Goal

A previously read passage is scheduled for a later reread; improvement is measured across accuracy, hint use, self-correction, and comfortable pacing — never speed alone — and shown to the child without shaming.

### Scope

**Included:**
- Reread eligibility and due-date logic (a passage is not rereadable immediately after first exposure)
- Celery worker job scanning for reread-due assignments (§3.1, §7.1)
- Comparison logic: accuracy delta, hint-use delta, self-correction delta, pacing, combined per ADR-010 — never words-per-minute alone
- Daily plan balancing new content vs. reread content, capped so a missed-practice streak doesn't create an oversized catch-up session

**Excluded:**
- Dashboard presentation of reread results (EPIC-010)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-009-01 | Reread eligibility and due-date scheduler (Celery job) | planning | 5 |
| STORY-009-02 | Reread comparison logic (accuracy, hint use, self-correction, pacing) | planning | 5 |
| STORY-009-03 | Balanced daily plan selection (new vs. reread), capped catch-up | planning | 3 |

### Dependencies

- EPIC-006 (needs `ReadingEvent` history to compare across attempts).
- EPIC-007 (needs hint-level history for the comparison).

### Acceptance Criteria

- A reread never appears in the same session as the passage's first read.
- Reread "improvement" evidence always includes at least one non-speed dimension (ADR-10 unit test).
- A child who missed several days does not receive an outsized catch-up session on return.

### Risks & Blockers

- Fluency becoming a speed contest is a named product risk (PRD §19); mitigated by the multi-dimension comparison in this epic.

---

## EPIC-010: Reading Progress Dashboard

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 13
**Target Sprint:** Sprint 4

*Maps to PRD epic R8 and functional requirement RR-11.*

### Description

Build the parent-facing reading dashboard (Architecture endpoint `GET /api/v1/children/{id}/reading/progress`) that reports evidence and trends instead of one reductive score.

### Goal

A parent can see sessions, minutes, word-level evidence, comprehension results, and reread improvement, and can trace any summary figure back to the sessions that produced it.

### Scope

**Included:**
- Progress aggregation endpoint combining `ReadingSession`, `ReadingEvent`, `QuestionAttempt`, and reread comparison data
- Dashboard UI: minutes/sessions, independently-read words, recurring difficult words, accuracy trend, reread improvement, comprehension by question type, self-corrections/retries, speech-uncertain regions shown separately from confirmed errors
- Filters by date range, content, and skill dimension (PRD §10)
- Drill-down from any summary metric to its source session(s)

**Excluded:**
- A single blended "reading level" score — explicitly excluded by PRD §10 and ADR-010/ADR-013

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-010-01 | Reading progress aggregation endpoint | planning | 5 |
| STORY-010-02 | Parent dashboard UI (sessions, words, comprehension, trends) | planning | 5 |
| STORY-010-03 | Drill-down from summary metric to source sessions, with filters | planning | 3 |

### Dependencies

- EPIC-006, EPIC-007, EPIC-008, EPIC-009 (all produce the evidence this dashboard aggregates).

### Acceptance Criteria

- Every figure shown reconciles exactly with the confirmed events/attempts behind it (no double counting of corrected events).
- A parent can click any summary number and see the underlying session(s).
- Speech-uncertain regions are visually and numerically distinct from confirmed errors.

### Risks & Blockers

- None currently.

---

## EPIC-011: Reading Badges Integration

**Priority:** P1 (Should Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 11
**Target Sprint:** Sprint 4

*Maps to PRD epic R9 and functional requirement RR-12. Governed by ADR-013 (shared badges, separate learning models).*

### Description

Wire Reading Quest events into the shared badge engine built in EPIC-003, and seed the reading-specific badge catalog from PRD §9.

### Goal

Reading activity awards badges through the same catalog and evaluator as Spelling Quest, displayed in one unified, filterable collection.

### Scope

**Included:**
- Reading `event_type`s registered with the shared `RewardLedger`/evaluator (independent word read, self-correction, hard-word success, reread improvement, comprehension answer, passage completion, book finish, reading-day streak)
- Seed data for the 13-badge reading catalog (`READ_FIRST_WORD` … `BOOK_FINISH`, PRD §9)
- Criteria + idempotency + replay tests specific to reading events
- Unified badge collection UI showing both games' badges with a game filter

**Excluded:**
- The shared evaluator/catalog engine itself (already built in EPIC-003)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-011-01 | Reading badge catalog seed data (13 badges) | planning | 3 |
| STORY-011-02 | Wire reading events into the shared badge evaluator | planning | 5 |
| STORY-011-03 | Unified badge collection UI with per-game filters | planning | 3 |

### Dependencies

- EPIC-003 (shared badge engine).
- EPIC-006, EPIC-007, EPIC-008 (source events for badge criteria).

### Acceptance Criteria

- No reading event awards the same badge to the same child twice, including under replay/retry.
- A word supplied through the final hint level does not satisfy an "independently read" badge criterion (consistency with ADR-012).
- A parent viewing the badge collection sees reading and spelling badges together with a working filter.

### Risks & Blockers

- None currently.

---

## EPIC-012: AI-Assisted Content Generation

**Priority:** P1 (Should Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 23
**Target Sprint:** Sprint 5

*Maps to PRD epic R10 and functional requirement RR-13. Governed by ADR-011 (approval + versioning required).*

### Description

Add the LLM-assisted drafting path (Architecture §3.1, §3.4, `packages/content-generation`) on top of the manual authoring flow from EPIC-004, entirely inside the parent workflow — never in a live child session (ADR-003 spirit extended by ADR-011).

### Goal

A parent can generate a level-appropriate, constrained passage and question set from target words, then review, edit, and approve it before it can ever reach a child.

### Scope

**Included:**
- LLM adapter interface with a self-hosted implementation (e.g., Ollama container) and a hosted-API implementation, selected by configuration
- `POST /api/v1/reading-items/{id}/generate` — constrained prompt builder taking level, length, topic, and target words
- Automated checks: length, banned-topic filtering, target-word coverage, question/answer consistency (PRD §4)
- Parent edit UI for the full generated draft (passage text, vocabulary, questions, expected answers) before approval
- Generation failure must not block manual content creation (falls back cleanly to the EPIC-004 manual path)

**Excluded:**
- Any code path that shows generated content to a child before parent approval (hard requirement, not a future consideration)
- OCR/camera import (RR-15, explicitly future per Architecture §12.1)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-012-01 | LLM adapter interface and self-hosted (Ollama) implementation | planning | 5 |
| STORY-012-02 | Hosted LLM adapter implementation, selectable via configuration | planning | 3 |
| STORY-012-03 | Constrained prompt builder (level, length, topic, target words) | planning | 5 |
| STORY-012-04 | Automated content checks (length, banned topics, target-word coverage, Q/A consistency) | planning | 5 |
| STORY-012-05 | Parent review/edit UI before approval | planning | 5 |

### Dependencies

- EPIC-004 (generated drafts become `ReadingItem`/`ReadingVersion` rows through the same approval pipeline).
- EPIC-001 (optional `llm` container).

### Acceptance Criteria

- No generated content is ever assigned to a child without passing through the EPIC-004 approval endpoint.
- A simulated LLM/provider failure still allows a parent to author content manually.
- Changing the underlying model or prompt after a version is approved does not alter that version's stored content (ADR-011).

### Risks & Blockers

- LLM generating unsafe/weak content is a named risk (Architecture §11.2); mitigated by automated checks plus mandatory review in this epic.

---

## EPIC-013: Observability, Reliability & Deployment Pipeline

**Priority:** P1 (Should Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 17
**Target Sprint:** Sprint 5

### Description

Implement the monitoring, logging, and disaster-recovery posture described in Architecture §8 and §6.3.

### Goal

Operators can see request/provider latency and error rates, get alerted on backlog/error spikes, and can rebuild the stack from the registry plus the latest backup within a documented RTO.

### Scope

**Included:**
- Structured JSON logs from every container, correlated by request ID
- Prometheus metrics (request latency, provider call latency/error rate, queue depth) + Grafana dashboards + alert rules (error-rate spikes, queue backlog, disk pressure on `postgres`)
- Loki (or hosted sink) log aggregation
- Error tracking (Sentry or GlitchTip) that never logs raw audio, transcripts, or full child responses
- Nightly `pg_dump`/WAL archiving to encrypted off-host storage, with a documented restore runbook and a periodic restore drill
- Migration-then-deploy sequencing automation for the CI/CD pipeline (§6.3 steps 3–4)
- Uptime/synthetic check against the public health endpoint

**Excluded:**
- Multi-host/HA failover (explicitly deferred, §12.1)

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-013-01 | Structured JSON logging with request-ID correlation across containers | planning | 3 |
| STORY-013-02 | Prometheus + Grafana + Loki stack with alert rules | planning | 5 |
| STORY-013-03 | Error tracking (Sentry/GlitchTip) with child-data-safe scrubbing | planning | 3 |
| STORY-013-04 | Postgres backup (pg_dump/WAL) and documented restore runbook | planning | 3 |
| STORY-013-05 | Automated migration-then-deploy sequencing in the CI/CD pipeline | planning | 3 |

### Dependencies

- EPIC-001 (container stack to instrument).

### Acceptance Criteria

- A request can be traced end-to-end across `api`/`worker` logs by correlation ID.
- An alert fires on a simulated error-rate spike and on Celery queue backlog.
- A documented restore drill successfully rebuilds a database from the latest backup.
- No error-tracking event ever contains raw audio, a full transcript, or a full child response body.

### Risks & Blockers

- None currently.

---

## EPIC-014: Privacy, Accessibility & Release Readiness

**Priority:** P0 (Must Have)
**Status:** planning
**Progress:** 0%
**Estimated Points:** 20
**Target Sprint:** Sprint 5

*Maps to PRD epic R11. Governed by ADR-005/ADR-014 (audio retention) and the product's COPPA-aware posture (Architecture §5.4).*

### Description

Close out the privacy, accessibility, and authorization gaps required before inviting real families, per Architecture §5.4 and PRD §20's Definition of Done.

### Goal

Reading Quest is safe, accessible, and operable for invited families, with no privacy or tenancy gap left untested.

### Scope

**Included:**
- Opt-in audio retention flow: explicit consent record, encrypted storage separate from the default non-retained path, and a visible deletion control (ADR-014)
- Data export and deletion controls per family
- Screen-reader, keyboard-only, text-scaling, and reduced-motion accessibility audit and fixes across the child and parent UI
- Tenant-isolation and content-authorization security test suite covering every new Reading Quest data path
- Loading, empty, offline, denied-microphone-permission, retry, and provider-failure UI states

**Excluded:**
- New feature work — this epic is verification and hardening of what EPIC-001–EPIC-013 already built

### Stories

| ID | Title | Status | Points |
|----|-------|--------|--------|
| STORY-014-01 | Opt-in audio retention: consent record, encrypted storage, deletion control | planning | 5 |
| STORY-014-02 | Family-level data export and deletion controls | planning | 5 |
| STORY-014-03 | Accessibility audit and fixes (screen reader, keyboard, text scaling, reduced motion) | planning | 5 |
| STORY-014-04 | Tenant-isolation and content-authorization security test suite | planning | 5 |

### Dependencies

- All prior epics (this is the release-readiness gate).

### Acceptance Criteria

- A parent can opt into, view, and delete retained recordings independently of the default (non-retained) audio path.
- A family can export and delete their data on request.
- Core child and parent flows pass a keyboard-only and screen-reader walkthrough.
- The security test suite fails the build if any new endpoint is reachable without the correct `family_id` scope.

### Risks & Blockers

- Legal review is still required before broad marketing to children or schools (Architecture §5.4) — outside engineering scope, flagged here as a release dependency, not a blocker for this epic's engineering work.

---

## Epic Status Tracking

| ID | Status | Progress | Notes |
|----|--------|----------|-------|
| EPIC-001 | ready-for-dev | 0% | Sprint 1. No blockers. |
| EPIC-002 | ready-for-dev | 0% | Sprint 1. Depends on EPIC-001 containers being up. |
| EPIC-003 | planning | 0% | Sprint 2. Story files not yet written. |
| EPIC-004 | planning | 0% | Sprint 2. Story files not yet written. |
| EPIC-005 | planning | 0% | Sprint 3. Story files not yet written. |
| EPIC-006 | planning | 0% | Sprint 3. Highest-point epic; heaviest technical risk (speech accuracy). |
| EPIC-007 | planning | 0% | Sprint 3. Story files not yet written. |
| EPIC-008 | planning | 0% | Sprint 4. Story files not yet written. |
| EPIC-009 | planning | 0% | Sprint 4. Story files not yet written. |
| EPIC-010 | planning | 0% | Sprint 4. Story files not yet written. |
| EPIC-011 | planning | 0% | Sprint 4. Story files not yet written. |
| EPIC-012 | planning | 0% | Sprint 5. Story files not yet written. |
| EPIC-013 | planning | 0% | Sprint 5. Story files not yet written. |
| EPIC-014 | planning | 0% | Sprint 5 (release gate). Story files not yet written. |

Detailed `docs/stories/STORY-XXX-XX.md` files exist today for the Sprint 1 stories (EPIC-001, EPIC-002). Remaining stories will get detailed story files during their sprint's planning step, per `agents/PROJECT_MANAGER_INSTRUCTIONS.md`.
