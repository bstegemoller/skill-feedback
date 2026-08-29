# Memory Library — Design Specification

**Status:** Draft v0.1 — ideation output, not yet implemented
**Working name:** the Stacks (a file-based, librarian-curated memory library for LLM agents)

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
3. Usage-driven structure: what is accessed often is fast to reach; what is not sinks.
4. Full provenance, human-auditable history, rollback — the library lives in git.
5. Works with weak query-time models: the top index must always fit the smallest
   context window the system serves.

**Non-goals**

- Building retrieval infrastructure (vector DBs, rerankers, graph engines). Adopt one
  later as a sidecar if plain-file search stops being enough (see §11 Phase 3).
- Per-user chat personalization ("remember my preferences") — existing products do this.
- Autonomous self-modification without review. The librarian proposes via commits;
  a human can always audit and revert.

## 3. Core principles

These are the load-bearing decisions. Everything in §4–§10 follows from them.

| # | Principle | Rationale |
|---|-----------|-----------|
| P1 | **The librarian is never on the read path.** Reads traverse static files; the librarian only runs at consolidation time. | An LLM gatekeeper on queries doubles latency, serializes access, and is a single point of failure — fatal on local hardware that runs one model at a time. |
| P2 | **Progressive disclosure with a hard cap at the top.** Tier 0 (`INDEX.md`) must fit the smallest context window served, by config, enforced every cycle. | Each tier's job is to tell the reader whether descending is worth it. Growth goes *down* the tiers, never up. This is the anti-bloat guarantee. |
| P3 | **Compression is a view, not a replacement.** Deep storage is append-mostly; every summary links to its sources; nothing is deleted, only superseded or demoted. | Prevents summary drift (curated text diverging from what sources actually said) and preserves temporal questions ("what did we believe in March?"). |
| P4 | **Claims carry sources, never bare truths.** Unverified ingested content is recorded as "source X asserts Y." | Injection defense: intake content eventually becomes trusted context for future agents. A poisoned document must not become an anonymous fact. |
| P5 | **Git is the trust boundary.** Every librarian cycle is one reviewable commit with a human-readable report. | Provenance, diff review, attribution, and rollback for free — the same human-gate philosophy as automated-feedback pipelines: the agent drafts, the human keeps the send button on anything high-consequence. |
| P6 | **Strongest model writes, any model reads.** Consolidation is asynchronous, so it can afford the best available model (or an occasional cloud call). Query-time agents can be small. | A weak librarian corrupts the library and errors compound silently. Never let the cheapest model hold the pen on long-term memory. |
| P7 | **Tiering is computed from actual use**, not guessed at write time. | Frequency + recency with decay (ACT-R base-level activation) is a proven model of "stronger pathways." The access log is ground truth. |

## 4. Repository layout

```
library/
├── INDEX.md                  # Tier 0 — skim. Hard token cap (config). The card catalog's front page.
├── catalog/                  # Tier 1 — moderate. One curated file per topic.
│   ├── <topic>.md
│   └── ...
├── stacks/                   # Tier 2 — deep. Full entries, append-mostly.
│   └── <topic>/
│       ├── <entry-id>.md
│       └── ...
├── intake/                   # Short-term memory. Raw, untrusted, unprocessed.
│   └── <YYYYMMDD-HHMMSS>-<slug>.md
└── meta/
    ├── config.yml            # caps, decay constant, thresholds, schedule, smallest-model window
    ├── access.log            # append-only JSONL, written by readers
    ├── activation.json       # computed scores per entry/topic (librarian output)
    ├── misses.log            # queries the library could not answer (JSONL)
    └── reports/
        └── <date>-cycle.md   # librarian's consolidation report, one per cycle
```

Notes:

- Everything is plain markdown/JSONL. Readable by a human, greppable by an agent,
  diffable by git. No database in the core loop.
- `intake/` is the only directory ordinary agents may write to (plus the two logs).
  Only the librarian writes `INDEX.md`, `catalog/`, `stacks/`, `activation.json`.
  Enforced by convention and review in Phase 1; by a commit hook in Phase 2.

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
5. Append one JSONL line per touched artifact to `meta/access.log`:
   `{"ts": "...", "id": "...", "depth": 0|1|2, "agent": "...", "query_hint": "..."}`
6. **On a miss** (no thread exists): log it to `meta/misses.log`, search externally,
   and write findings — with sources — to `intake/`. The library learns from its own
   misses; the librarian reads `misses.log` as collection-development guidance
   ("we keep getting asked about X and have nothing").

Logging is best-effort append-only; a failed log write never blocks a read.

## 7. Write protocol (ordinary agents)

Ordinary agents write **only** to `intake/` and the two logs. An intake note is
free-form markdown plus minimal frontmatter (source, timestamp, submitting agent).
No tagging, no cataloging, no editing of curated tiers — that is the librarian's job.
Cheap capture, expensive curation, in that order.

## 8. The librarian: consolidation cycle ("sleep")

**Trigger:** scheduled (e.g. nightly, when local hardware is idle) and/or when
`intake/` exceeds a size threshold. **Model:** the strongest available (P6). Latency
is irrelevant; quality is everything.

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
5. **Re-tier.** Recompute activation (§9) from `access.log`. Promote hot topics to
   more prominent / more detailed treatment in `INDEX.md`; demote cold ones to a
   one-line stub with a pointer. Deep entries never lose their catalog pointer —
   demotion changes prominence, not reachability.
6. **Enforce the cap.** Count `INDEX.md` tokens against `config.yml`'s
   `smallest_model_window` budget. Over budget → trim the coldest lines first. This
   step is not skippable (P2).
7. **Audit sample.** Pick N random catalog claims and re-check them against their
   linked sources; flag drift in the report. (Cheap continuous defense against P3
   violations.)
8. **Report & commit.** Write `meta/reports/<date>-cycle.md`: what was ingested,
   merged, superseded, promoted, demoted, trimmed, flagged. Clear `intake/` of
   processed files (they persist in git history). One commit, structured message.
   Anything flagged high-consequence (a supersession of a load-bearing fact, a
   confidence upgrade, drift findings) is listed at the top of the report for human
   review (P5).

## 9. Activation scoring

Per entry/topic, from `access.log`, ACT-R base-level activation:

```
A = ln( Σ_j  t_j ^ (−d) )
```

where `t_j` = time since access *j* (in days, floor 0.05) and `d` = decay rate
(default **0.5**, in `config.yml`). Frequency and recency both raise A; time decays it.

- Thresholds `θ_hot > θ_warm` (config) map A to tiers: hot → featured in `INDEX.md`;
  warm → normal catalog presence; cold → stub + deep storage.
- **Hysteresis:** a topic must cross a threshold on two consecutive cycles to move,
  preventing churn from one-off spikes.
- New entries get a grace period (default 3 cycles) at warm before decay applies —
  the equivalent of "recently shelved, give it a chance."
- Depth-0 accesses weight less than depth-2 (skimming the index is not the same
  signal as reading the entry); default weights 0.25 / 0.75 / 1.0.

## 10. Threat model & guardrails (summary)

| Risk | Guardrail |
|------|-----------|
| Librarian on read path → latency/bottleneck | P1; read protocol is file-only by construction |
| Skim layer bloat → context overflow on small models | P2; cap enforced every cycle, trims coldest first |
| Summary drift | P3 provenance links + §8.7 audit sampling |
| Prompt injection via ingested content | P4 claims-with-sources; librarian treats intake as data; human review of flagged promotions |
| Weak model corrupting curation | P6; librarian runs strongest model, asynchronously |
| Silent bad edits | P5; one commit per cycle, human-readable report, git revert |
| Tier churn | §9 hysteresis + grace period |
| Lost history / temporal questions | §5 supersession with validity windows; append-mostly stacks |

## 11. Implementation phases

**Phase 1 — files + manual sleep (a weekend).**
Directory layout, entry format, `config.yml`. The read/write protocols shipped as an
agent skill (progressive-disclosure instructions). The librarian is a prompt you run
as an ordinary interactive session: "run a consolidation cycle per §8." Human reviews
the diff before it merges. Proves the curation loop with zero infrastructure.

**Phase 2 — scheduled sleep + enforcement.**
Consolidation as a scheduled job (cron / Claude Code routine) on idle hours. A hook
appends `access.log` lines automatically. A pre-commit check rejects non-librarian
writes to curated tiers and INDEX-cap violations mechanically, so the constraint
doesn't depend on model obedience.

**Phase 3 — optional retrieval sidecar.**
Only if grep + catalog navigation stops scaling: a local embedding index over
`stacks/` (small local embedding model, ~tens of MB), rebuilt during the sleep cycle.
Still no LLM on the read path; the index is just a better card catalog. If needs grow
past that, adopt an existing engine (e.g. a temporal knowledge graph) *underneath*
the catalog rather than building one.

## 12. Metrics (read them in the cycle report)

- **Hit rate by depth** — how often depth 0/1 sufficed. Rising depth-0 sufficiency
  means the skim layer is earning its cap.
- **Miss rate per topic** — collection gaps; librarian's acquisition list.
- **INDEX token count over time** — must stay flat under the cap while stacks grow.
- **Supersessions per cycle** — how alive the knowledge is.
- **Drift flags from audit sampling** — should trend to zero; if not, the librarian
  model is too weak (see P6).
- **Demotion/promotion volume** — with hysteresis, should settle; sustained churn
  means thresholds need tuning.

## 13. Open questions

1. Multi-agent write contention on `access.log` in parallel sessions (likely fine as
   append-only JSONL; verify).
2. Topic taxonomy governance — librarian-invented tags will drift; consider a small
   controlled vocabulary in `config.yml` that the librarian may propose additions to
   (flagged for human review).
3. When two libraries merge (e.g. per-project libraries into a personal one), whose
   activation history wins? Deferred to Phase 3+.
4. Whether `misses.log` should trigger *proactive* acquisition (librarian searches
   externally during sleep) or stay reactive. Proactive is powerful but widens the
   injection surface; if adopted, findings enter as `confidence: reported` intake
   like everything else.
