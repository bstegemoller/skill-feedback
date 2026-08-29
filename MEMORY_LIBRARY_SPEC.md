# Memory Library — Design Specification

**Status:** Draft v0.3 — ideation output, not yet implemented
**Working name:** the Stacks (a file-based, librarian-curated memory library for LLM agents)
**Changes in v0.2:** adds the steward role (audit, health, bounded self-tuning), the
change-authority ladder (P8), work-proportional triggers replacing time schedules,
and prompts-as-files so instruction revisions are git-gated diffs.
**Changes in v0.3:** adds the sorting-room scale function (parallel triage workers
under a single-cataloger invariant, P9), per-agent intake quotas with trust weights
for multi-agent fleets, and per-session access logs (resolving open question 1).

---

## 1. Problem statement

Agents accumulate knowledge across sessions but have nowhere durable, structured, and
*trustworthy* to keep it. Commercial memory frameworks (Letta, Zep, Mem0, LangMem) have
made storage and retrieval a commodity, but none of them do **curation over time**:
usage-driven tiering, graceful demotion, and keeping a small "skim" layer honest as the
store grows for months or years. That lifecycle work — the librarian's second job — is
the gap this design fills.

Design target: local-first or hybrid deployments, where background consolidation is
nearly free (idle hardware) and where small query-time models benefit most from
pre-digested, low-noise context.

## 2. Goals and non-goals

**Goals**

1. Multi-resolution memory: skim / moderate / deep, with progressive disclosure.
2. A librarian process that consolidates, catalogs, dedupes, supersedes, and re-tiers —
   entirely off the read path.
3. A steward process that audits the librarian's work, tracks system health, and
   corrects drift — without ever grading its own homework or silently rewriting rules.
4. Usage-driven structure: what is accessed often is fast to reach; what is not sinks.
5. Work-proportional maintenance: curation and audit effort scale with how much the
   library is actually growing and being used, not with the calendar.
6. Full provenance, human-auditable history, rollback — the library lives in git.
7. Works with weak query-time models: the top index must always fit the smallest
   context window the system serves.

**Non-goals**

- Building retrieval infrastructure (vector DBs, rerankers, graph engines). Adopt one
  later as a sidecar if plain-file search stops being enough (see §12 Phase 3).
- Per-user chat personalization ("remember my preferences") — existing products do this.
- Autonomous self-modification of instructions. Agents may tune numbers within
  human-set bounds; instruction changes are proposals a human approves (P8).

## 3. Core principles

These are the load-bearing decisions. Everything in §4–§11 follows from them.
**P1–P9 are the constitution: no agent process may edit this section or the rules it
states. Only a human changes the constitution.**

| # | Principle | Rationale |
|---|-----------|-----------|
| P1 | **The librarian is never on the read path.** Reads traverse static files; the librarian only runs at consolidation time. | An LLM gatekeeper on queries doubles latency, serializes access, and is a single point of failure — fatal on local hardware that runs one model at a time. |
| P2 | **Progressive disclosure with a hard cap at the top.** Tier 0 (`INDEX.md`) must fit the smallest context window served, by config, enforced every cycle. | Each tier's job is to tell the reader whether descending is worth it. Growth goes *down* the tiers, never up. This is the anti-bloat guarantee. |
| P3 | **Compression is a view, not a replacement.** Deep storage is append-mostly; every summary links to its sources; nothing is deleted, only superseded or demoted. | Prevents summary drift (curated text diverging from what sources actually said) and preserves temporal questions ("what did we believe in March?"). |
| P4 | **Claims carry sources, never bare truths.** Unverified ingested content is recorded as "source X asserts Y." | Injection defense: intake content eventually becomes trusted context for future agents. A poisoned document must not become an anonymous fact. |
| P5 | **Git is the trust boundary.** Every librarian and steward cycle is one reviewable commit with a human-readable report. | Provenance, diff review, attribution, and rollback for free — the agent drafts, the human keeps the send button on anything high-consequence. |
| P6 | **Strongest model writes, any model reads.** Consolidation and audit are asynchronous, so they can afford the best available model (or an occasional cloud call). Query-time agents can be small. | A weak librarian corrupts the library and errors compound silently. Never let the cheapest model hold the pen on long-term memory. |
| P7 | **Tiering is computed from actual use**, not guessed at write time. | Frequency + recency with decay (ACT-R base-level activation) is a proven model of "stronger pathways." The access log is ground truth. |
| P8 | **Change authority is layered.** Data changes are autonomous; parameter changes are autonomous only within human-declared bounds; instruction changes are proposals gated by a human; the constitution is human-only. No agent audits its own output. | A system that can rewrite its own operating instructions unsupervised is a self-modifying instruction store — the drift it was built to correct can then live in the corrector. The recursion must terminate at the human + git, not at another agent. |
| P9 | **Many hands may sort; one mind catalogs.** Parallel workers may triage, draft, and annotate — read-only against the library, writing only to staging. Integration (reconciling, superseding, cataloging, re-tiering, cap enforcement) is done by exactly one librarian per cycle, and the library is never sharded into per-topic sub-librarians. | The librarian's product is global coherence — contradictions span items, the taxonomy is one tree, the cap is one budget. Parallel writers or shards reintroduce every distributed-consistency problem the single-writer design exists to avoid. The serial step stays cheap because workers pre-digest its input. |

### The change-authority ladder (P8, concretely)

| Layer | Examples | Who changes it | Gate |
|-------|----------|----------------|------|
| **Data** | entries, catalogs, `INDEX.md`, activation scores | Librarian | Autonomous; git-reviewable after the fact |
| **Parameters** | decay `d`, tier thresholds, trigger counts, audit sample size | Steward | Autonomous **within bounds declared in `config.yml`**; one change per cycle, rationale logged |
| **Instructions** | `prompts/*.md` (librarian, steward, reader protocol), the bounds themselves | Steward **proposes** a diff into `meta/proposals/` | **Human approves and applies** |
| **Constitution** | §3 principles, cap semantics, the ladder itself | Human only | — |

## 4. Repository layout

```
library/
├── INDEX.md                  # Tier 0 — skim. Hard token cap (config). The card catalog's front page.
├── catalog/                  # Tier 1 — moderate. One curated file per topic.
│   └── <topic>.md
├── stacks/                   # Tier 2 — deep. Full entries, append-mostly.
│   └── <topic>/<entry-id>.md
├── intake/                   # Short-term memory. Raw, untrusted, unprocessed.
│   └── <YYYYMMDD-HHMMSS>-<slug>.md
├── prompts/                  # Operational instructions, as versioned files.
│   ├── reader-protocol.md    # what every querying agent follows (§6–§7)
│   ├── librarian.md          # the consolidation-cycle prompt (§8)
│   └── steward.md            # the audit-cycle prompt (§10)
└── meta/
    ├── config.yml            # caps, decay, thresholds, trigger counts, quotas, tuning bounds
    ├── access/               # per-session read logs (JSONL), one file per session
    │   └── <session-id>.jsonl
    ├── access.log            # canonical merged log (librarian output, §8.5)
    ├── staging/              # sorting-room worker output: triage annotations, entry drafts
    ├── activation.json       # computed scores per entry/topic (librarian output)
    ├── misses.log            # queries the library could not answer (JSONL)
    ├── proposals/            # steward's instruction-change drafts, awaiting human review
    ├── reports/              # librarian cycle reports, one per cycle
    │   └── <date>-cycle.md
    └── health/               # steward health reports, one per steward cycle
        └── <date>-steward.md
```

Notes:

- Everything is plain markdown/JSONL. Readable by a human, greppable by an agent,
  diffable by git. No database in the core loop.
- Prompts live in the repo so "revise the instructions" is a concrete, reviewable
  diff — not an invisible edit to some agent's configuration.
- Write permissions by role: ordinary agents → `intake/`, their own `meta/access/`
  file, and `misses.log` only. Sorting-room workers (§8a) → `meta/staging/` only.
  Librarian → curated tiers, `access.log`, `activation.json`, `reports/`. Steward →
  `config.yml` (bounded values only), `health/`, `proposals/`. Nobody but a human →
  `prompts/`, §3, or the bounds in `config.yml`. Convention + review in Phase 1;
  commit hooks in Phase 2.
- Per-session access files exist so parallel sessions (including ephemeral cloud
  containers that must push each cycle) never contend on one log — each session
  appends to its own file and the librarian merges them (§8.5).

## 5. Entry format

Every file in `stacks/` (and, lighter-weight, `catalog/`) carries YAML frontmatter:

```yaml
---
id: 2026-08-29-usb-pd-negotiation      # stable, never reused
created: 2026-08-29
updated: 2026-08-29
status: active                          # active | superseded
superseded_by: null                     # entry id, when status: superseded
valid_from: 2026-08-29                  # temporal validity window
valid_until: null
sources:                                # provenance — required, ≥1
  - kind: url | file | session | agent-observation
    ref: <link, path, or session id>
    retrieved: 2026-08-29
confidence: reported                    # verified | corroborated | reported
tags: [hardware, usb, power]
links: [2026-07-11-usb-c-cable-testing] # related entry ids
---
```

Body rules:

- Facts the librarian has verified or corroborated may be stated directly.
- Everything else is attributed: "According to `<source>`, …". Confidence is explicit
  (P4). The librarian may upgrade `reported → corroborated` only when a second
  independent source lands, and records both.
- Superseding: when new information contradicts an entry, the old entry gets
  `status: superseded`, `superseded_by`, and `valid_until`; the new entry links back.
  Both remain readable. Nothing is deleted (P3).

## 6. Read protocol (any agent, any model)

The read path is file I/O plus one log append. **No LLM other than the querying agent
itself is involved** (P1).

1. Read `library/INDEX.md` (always small enough to load whole, by P2).
2. If a topic line answers the question at skim level — done. Log the access.
3. Otherwise follow the index pointer to `catalog/<topic>.md`. If the curated summary
   suffices — done. Log the access at depth 1.
4. Otherwise descend to the specific `stacks/` entries the catalog cites. Log at depth 2.
5. Append one JSONL line per touched artifact to this session's own file,
   `meta/access/<session-id>.jsonl`:
   `{"ts": "...", "id": "...", "depth": 0|1|2, "agent": "...", "query_hint": "..."}`
6. **On a miss** (no thread exists): log it to `meta/misses.log`, search externally,
   and write findings — with sources — to `intake/`. The library learns from its own
   misses; the librarian reads `misses.log` as collection-development guidance
   ("we keep getting asked about X and have nothing").

Logging is best-effort append-only; a failed log write never blocks a read.

## 7. Write protocol (ordinary agents)

Ordinary agents write **only** to `intake/` and the logs. An intake note is
free-form markdown plus minimal frontmatter (source, timestamp, submitting agent).
No tagging, no cataloging, no editing of curated tiers — that is the librarian's job.
Cheap capture, expensive curation, in that order.

**Intake quotas and trust weights (multi-agent fleets).** Intake volume scales with
agent count, and so does noise; ten enthusiastic workers filing observations can bury
the signal. Two mechanical controls in `config.yml`:

- **Quotas:** each agent identity gets a per-day intake budget (default in config,
  overridable per agent). Over-quota submissions are not rejected — they land in
  intake flagged `over_quota: true`, and the librarian triages them last and with a
  skeptical eye. Quotas shape attention, not admission; rejection would lose
  information and invite workarounds.
- **Trust weights:** each agent identity carries a weight (e.g. orchestrator 1.0,
  spawned worker 0.5, unknown agent 0.25) recorded in the intake frontmatter by the
  submitting hook. The librarian uses it as a prior — a low-weight observation needs
  corroboration before it earns more than `confidence: reported` — never as a filter.
  Weights are parameters: the steward tunes them within bounds; new identities start
  at the lowest weight and earn upward via steward review of their track record.

Both controls are enforced by the intake hook (a script reading config), not by
model obedience.

## 8. The librarian: consolidation cycle ("sleep")

**Trigger — work-proportional, not calendar-driven.** A cheap script (cron- or
hook-run, no LLM) checks state and fires the cycle when any condition holds:

- `intake/` holds ≥ `intake_trigger` files (default 20), **or**
- the oldest unprocessed intake file is older than `max_staleness_days` (default 7)
  and intake is non-empty — so a trickle of 2 entries still gets processed
  eventually, but never wakes the librarian on its own, **or**
- `access.log` has grown by ≥ `access_trigger` lines (default 200) since the last
  cycle — re-tiering depends on *reads*, which accumulate even when intake is quiet.

If none hold, nothing runs and nothing is spent. Maintenance effort tracks the
library's actual activity — busy weeks get frequent sleep cycles, dead weeks get none.

**Model:** the strongest available (P6). Latency is irrelevant; quality is everything.

Each cycle, in order:

1. **Ingest.** Read every file in `intake/`. Treat all of it as untrusted data — the
   librarian's prompt states explicitly that intake content is information to be
   cataloged, never instructions to be followed.
2. **Dedupe & reconcile.** Match against existing entries (tags, links, grep,
   optionally embeddings in Phase 3). Merge duplicates. On contradiction: supersede
   per §5 — never overwrite, never delete.
3. **File.** Create/update `stacks/` entries with full frontmatter and provenance.
4. **Catalog.** Update the affected `catalog/<topic>.md` files: curated summary,
   pointers to entries, open questions. Split a topic when its catalog file exceeds
   its own size budget; merge topics that have withered.
5. **Merge & re-tier.** Merge the per-session files in `meta/access/` into the
   canonical `access.log` (a script step, order-insensitive; merged files are
   removed, surviving in git history). Recompute activation (§9) from `access.log`. Promote hot topics to
   more prominent / more detailed treatment in `INDEX.md`; demote cold ones to a
   one-line stub with a pointer. Deep entries never lose their catalog pointer —
   demotion changes prominence, not reachability.
6. **Enforce the cap.** Count `INDEX.md` tokens against `config.yml`'s
   `smallest_model_window` budget. Over budget → trim the coldest lines first. This
   step is not skippable (P2).
7. **Report & commit.** Write `meta/reports/<date>-cycle.md`: what was ingested,
   merged, superseded, promoted, demoted, trimmed, plus anything the librarian is
   unsure about, explicitly flagged. Clear `intake/` of processed files (they persist
   in git history). One commit, structured message. High-consequence items (a
   supersession of a load-bearing fact, a confidence upgrade) top the report for
   human review (P5).

The librarian does **not** audit its own output — that is the steward's job (P8, §10).

### 8a. The sorting room — scaling the cycle (P9)

When intake outgrows what one cycle handles comfortably, the system scales through
an ordered ladder of levers — each cheaper than the next, each tried (or ruled out)
before the next engages:

1. **More frequent cycles (automatic).** The work-proportional trigger already fires
   more often under load, shrinking batch size. Chronic overload usually means
   `intake_trigger` is set too high — a bounded steward tune, not an architecture
   change.
2. **Mechanical & cheap-model triage.** Hash-based dedupe, quota flagging, and
   junk/duplicate/substantive classification are scripts plus (at most) a small
   cheap model. P6 cuts both ways: judgment needs the strongest model; sorting
   doesn't. No agents spawned.
3. **The sorting room: parallel triage workers.** When unprocessed intake exceeds
   `worker_trigger`, the librarian spawns up to `max_workers` ephemeral workers.
   Workers do the **per-item, read-only** work: triage classification, draft
   frontmatter and summaries, source spot-checks, lookups of likely-related
   entries. They read the library freely but write **only** to `meta/staging/`,
   and their output is treated exactly like intake — untrusted drafts the librarian
   uses or discards. The change-authority ladder doesn't acquire a new rung;
   workers live below the data layer.

**What stays serial, always:** integration — cross-item reconciliation,
supersession decisions, catalog updates, re-tiering, cap enforcement. One librarian
per cycle does all of it (P9). This does not bog down: integrating pre-triaged,
pre-summarized, junk-filtered items is a fraction of raw processing, so the sorting
room shrinks the serial step's cost rather than competing with it. Many hands sort;
one mind catalogs.

**Hard stop:** the library is never sharded into per-topic sub-librarians. That
reintroduces distributed consistency (cross-topic links, supersessions spanning
shards, ownership of the one index cap) — the exact costs the single-writer design
avoids. Revisit only if a single librarian with a full sorting room demonstrably
cannot keep up; for a personal or small-team library, that is likely never.

**Deployment note:** the sorting room pays off only where parallelism is real
(e.g. cloud subagent fan-out). On a single local GPU, spawning workers just queues
serial inference plus orchestration overhead — there, lever 2 is the whole game.

## 9. Activation scoring

Per entry/topic, from `access.log`, ACT-R base-level activation:

```
A = ln( Σ_j  t_j ^ (−d) )
```

where `t_j` = time since access *j* (in days, floor 0.05) and `d` = decay rate
(default **0.5**, tunable within bounds in `config.yml`). Frequency and recency both
raise A; time decays it.

- Thresholds `θ_hot > θ_warm` (config) map A to tiers: hot → featured in `INDEX.md`;
  warm → normal catalog presence; cold → stub + deep storage.
- **Hysteresis:** a topic must cross a threshold on two consecutive cycles to move,
  preventing churn from one-off spikes.
- New entries get a grace period (default 3 cycles) at warm before decay applies —
  the equivalent of "recently shelved, give it a chance."
- Depth-0 accesses weight less than depth-2 (skimming the index is not the same
  signal as reading the entry); default weights 0.25 / 0.75 / 1.0.

## 10. The steward: audit and self-correction cycle

The system-admin role. The librarian does the work; the steward checks the work,
watches trends, and keeps the machine tuned. Separation of duties is the point:
in v0.1 the librarian audited its own summaries, which invites self-consistency
bias — a model reviewing its own output tends to approve it. The steward always
runs in a **fresh context with no memory of producing the content it audits**, and
preferably a different strong model than the librarian used.

**Trigger — proportional to curation activity, not the calendar:**

- every `steward_every_n_cycles` librarian cycles (default 5), **or**
- immediately, when a librarian report contains explicit flags, **or**
- on demand (a human asks for a health check).

Cadence therefore scales with knowledge growth automatically: a library ingesting
heavily gets audited often; a quiet one costs nothing. Same dumb-script trigger
mechanism as §8 — counting cycle reports is a file count, not an LLM call.

Each steward cycle:

1. **Read the trend, not the snapshot.** Load the last N librarian reports and the
   metrics of §13. A single bad cycle is noise; three cycles of rising drift flags
   or index-cap pressure is a signal.
2. **Adversarial audit sample.** Pick `audit_sample_n` random catalog claims and
   re-verify each against its linked `stacks/` sources — actively looking for drift,
   unsupported confidence upgrades, and orphaned links. (Moved here from the
   librarian's cycle, per P8's no-self-audit rule.)
3. **Spot-check judgment calls.** Review a sample of recent supersessions, tier
   moves, and index trims: were they justified by the evidence and the activation
   data, or is the librarian over-merging / over-trimming?
4. **Health report.** Write `meta/health/<date>-steward.md`: metric trends, audit
   findings, defect list. **Defects in content are reported, not fixed** — the
   steward never edits entries or catalogs; the librarian repairs flagged items
   next cycle. (An auditor that also edits the books stops being an auditor.)
5. **Tune parameters — within bounds.** Adjust values in `config.yml` (decay `d`,
   thresholds, trigger counts, sample size) only inside the human-declared bounds,
   at most one parameter per cycle, with the observed rationale logged in the health
   report. Bounds themselves are instruction-layer: the steward may propose changing
   them, never change them.
6. **Propose instruction revisions.** When drift traces to the *instructions* — the
   librarian prompt is producing systematic over-compression, the reader protocol
   is being misread by small models — draft the revised prompt as a diff in
   `meta/proposals/`, with evidence attached. A human reviews, applies (or rejects),
   and commits. The steward never edits `prompts/` directly (P8).
7. **Commit.** One commit: health report, any bounded parameter change, any proposal.

**What the steward is not:** a third tier of watcher. Nobody audits the steward;
the human reading its health reports and diffs is the terminus of the recursion.
Adding a watcher-watcher just moves the same trust question up a level at double
the cost — git plus a human is the fixed point.

### Trigger configuration (excerpt of `config.yml`)

```yaml
triggers:
  librarian:
    intake_trigger: 20          # fire at N unprocessed intake files
    max_staleness_days: 7       # ... or when the oldest has waited this long
    access_trigger: 200         # ... or after this many new access-log lines
  steward:
    every_n_cycles: 5           # audit after every 5 librarian cycles
    on_flags: true              # or immediately when a cycle report carries flags
scaling:                        # §8a sorting room
  worker_trigger: 40            # unprocessed intake beyond which workers spawn
  max_workers: 4                # hard cap on parallel triage workers
intake:                         # §7 quotas & trust weights
  default_quota_per_day: 10
  trust_weights:
    orchestrator: 1.0
    worker: 0.5
    unknown: 0.25
bounds:                         # steward may tune inside these; only a human edits the bounds
  decay_d:        [0.3, 0.7]
  intake_trigger: [10, 50]
  audit_sample_n: [5, 50]
  worker_trigger: [20, 100]
  max_workers:    [1, 8]
  default_quota_per_day: [5, 50]
  trust_weights:  [0.1, 1.0]    # per-identity values tunable in this range
```

## 11. Threat model & guardrails (summary)

| Risk | Guardrail |
|------|-----------|
| Librarian on read path → latency/bottleneck | P1; read protocol is file-only by construction |
| Skim layer bloat → context overflow on small models | P2; cap enforced every cycle, trims coldest first |
| Summary drift | P3 provenance links + steward audit sampling (§10.2) |
| Prompt injection via ingested content | P4 claims-with-sources; librarian treats intake as data; human review of flagged promotions |
| Weak model corrupting curation | P6; librarian runs strongest model, asynchronously; steward trend-detects quality decay (§13) |
| Self-review bias (auditor approving its own work) | P8; steward runs fresh-context / different model; librarian never self-audits |
| Runaway self-modification (drift living in the corrector) | P8 change-authority ladder; prompts and bounds are human-gated; constitution immutable to agents |
| Silent bad edits | P5; one commit per cycle, human-readable report, git revert |
| Maintenance overhead / unnecessary runs | Work-proportional script triggers (§8, §10); zero LLM cost when idle |
| Intake flooding by a noisy agent fleet | §7 per-agent quotas (attention-shaping, not rejection) + trust-weight priors; steward tunes both within bounds |
| Parallel workers corrupting coherence | P9 single-cataloger invariant; workers are read-only + staging-only; integration never parallelized; no sharding |
| Concurrent sessions contending on the access log | Per-session log files merged at cycle time (§4, §8.5) |
| Tier churn | §9 hysteresis + grace period; steward flags sustained churn as a tuning signal |
| Lost history / temporal questions | §5 supersession with validity windows; append-mostly stacks |

## 12. Implementation phases

**Phase 1 — files + manual cycles (a weekend).**
Directory layout, entry format, `config.yml`, the three prompt files. The read/write
protocols shipped as an agent skill (progressive-disclosure instructions). Librarian
and steward are prompts you run as ordinary interactive sessions when the trigger
conditions look met; a human reviews every diff. Proves both loops with zero
infrastructure.

**Phase 2 — scripted triggers + enforcement.**
The trigger script from §8/§10 runs cheaply on a schedule and launches a cycle
*only* when thresholds are met (the schedule is just when the counter gets checked —
the LLM never runs on an empty library). A hook appends `access.log` lines
automatically. A pre-commit check mechanically rejects out-of-role writes
(non-librarian edits to curated tiers, agent edits to `prompts/`, out-of-bounds
parameter values, INDEX-cap violations), so the ladder doesn't depend on model
obedience.

**Phase 3 — optional retrieval sidecar.**
Only if grep + catalog navigation stops scaling: a local embedding index over
`stacks/` (small local embedding model, ~tens of MB), rebuilt during the sleep cycle.
Still no LLM on the read path; the index is just a better card catalog. If needs grow
past that, adopt an existing engine (e.g. a temporal knowledge graph) *underneath*
the catalog rather than building one.

## 13. Metrics (the steward reads these as trends, §10.1)

- **Hit rate by depth** — how often depth 0/1 sufficed. Rising depth-0 sufficiency
  means the skim layer is earning its cap.
- **Miss rate per topic** — collection gaps; librarian's acquisition list.
- **INDEX token count over time** — must stay flat under the cap while stacks grow.
- **Supersessions per cycle** — how alive the knowledge is.
- **Audit-drift findings per steward cycle** — should trend to zero; a rising trend
  indicts the librarian model or its prompt (steward proposes accordingly).
- **Demotion/promotion volume** — with hysteresis, should settle; sustained churn
  means thresholds need tuning (a bounded steward adjustment).
- **Intake volume and junk ratio per agent identity** — feeds quota and trust-weight
  tuning; a rising junk ratio from one identity is a targeted signal, not a reason
  to throttle the whole fleet.
- **Sorting-room utilization** — how often workers spawn and their draft acceptance
  rate. Workers spawning every cycle means `intake_trigger` is too high (lever 1
  before lever 3); a low acceptance rate means worker triage instructions need a
  steward proposal.

## 14. Open questions

1. ~~Multi-agent write contention on `access.log` in parallel sessions.~~
   **Resolved in v0.3:** per-session log files under `meta/access/`, merged by the
   librarian at cycle time — no shared-file writes on the read path at all.
2. Topic taxonomy governance — librarian-invented tags will drift. Partially
   addressed in v0.2: the steward monitors tag sprawl as a health metric and
   proposes controlled-vocabulary changes via `meta/proposals/`; a human ratifies.
   Open: the starting vocabulary itself.
3. When two libraries merge (e.g. per-project libraries into a personal one), whose
   activation history wins? Deferred to Phase 3+.
4. Whether `misses.log` should trigger *proactive* acquisition (librarian searches
   externally during sleep) or stay reactive. Proactive is powerful but widens the
   injection surface; if adopted, findings enter as `confidence: reported` intake
   like everything else.
5. Evaluating the steward's parameter tuning: with a single library there is no
   control group, so "did that decay change help?" is hard to answer rigorously.
   Likely pragmatic answer: before/after trend comparison over several cycles, and
   a human sanity check in the health report — accept that this is engineering
   judgment, not science.
6. Trust-weight governance at fleet scale: new agent identities start at the lowest
   weight and earn upward via steward review, but "identity" is only as strong as
   the hook that stamps it — a misconfigured or spoofed agent name inherits the
   wrong prior. Fine for a personal or small-team deployment where identities are
   few and human-known; needs real thought (signing? allowlists?) before any
   open-fleet use.
