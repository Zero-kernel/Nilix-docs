# Skill Loop v2 Specification

## Five-Stage Self-Evolving Development Loop for Zero-OS (Nilix) Kernel

**Version:** 2.0
**Date:** 2026-07-11
**Based on:** "From Context to Skills: Can Language Models Learn from Context Skillfully?" (Si et al., 2026, arXiv:2604.27660v3) + `ctx2skill-integration-spec.md` v1.0 (the previous version of this plan)
**Status:** IMPLEMENTED — skill files updated per this spec on 2026-07-11
**Convergence note:** Codex MCP unavailable at design time; this spec converged via multi-lens adversarial self-review (standing directive 2026-07-02). The Codex Cooperation Protocol below formalizes that exact fallback so future rounds never need ad-hoc authorization (as R178 did).

---

## 1. Motivation — What v1 Did and What It Left Open

v1 (`ctx2skill-integration-spec.md`) made the three-skill loop **stateful**: evolving skill
documents (audit-skills, fix-skills, dev-skills), Proposer/Generator phases, probe sets, and
Cross-Time Replay. All of that is implemented and stays.

Four structural gaps remained in the **loop topology** itself:

1. **No independent Judge stage.** The paper's Judge is a *neutral* role. In v1 it was embedded
   inside the audit (Phase 5 cross-review) and the fix (Phase 4 convergence) — i.e., each side
   judged itself with help from Codex. Fixes that passed the fix skill's own convergence gate were
   never re-examined by a party that had *no stake* in them. R173 (shell starved Ring-3 — caught
   only by musl-check, not by the fix round's own gate) and the R177 re-run (prior R177 judged
   incomplete + over-rated) are concrete escapes of exactly this class.
2. **Unbounded plan monolith.** `docs/next-phase-plan.md` reached 5,767 lines (v15.12). Every
   stage paid the parsing cost; history and live state were interleaved; single-file contention.
3. **Inconsistent document locations.** Reports actually live in `docs/review/audits/` and dated
   plan snapshots in `docs/review/nextplan/`, but the skill files still said `docs/review/qa-*.md`
   and the monolith path. The skills and the tree had drifted apart.
4. **Hard Codex dependency.** Every skill assumed Codex MCP is reachable. Since 2026-07-02 it has
   not been; R178 handled this with a one-off user authorization and an improvised (excellent)
   fallback. The fallback must be a first-class, specified mode — not an exception.

v2 fixes all four with a **five-stage loop**, relocated documents, and an explicit
**Codex-cooperative vs Claude-solo mode split** in every stage.

---

## 2. The v2 Loop

```
        ┌─────────────────────────────────────────────────────────────────┐
        │                                                                 │
        ▼                                                                 │
[1] kernel-security-audit ──── writes ───→ docs/review/audits/qa-{date}.md│
        │  (Challenger)                                                   │
        │ findings > 0                │ 0 actionable findings             │
        ▼                             └──────────────┐ (short-circuit)    │
[2] kernel-security-audit-fix ─ repairs; updates ────┼→ audit doc statuses│
        │  (Reasoner)           + CURRENT nextplan doc│                   │
        ▼                                             │                   │
[3] kernel-review-fix ──────── writes ───→ docs/review/reviewfix/         │
        │  (Judge + repair)                reviewfix-{date}.md            │
        │                                  + nextplan/audit status sync   │
        ▼                                             │                   │
[4] kernel-next-phase ──────── writes ───→ docs/review/nextplan/          │
        │  (Planner / rollover)            next-phase-plan-{date}.md      │
        ▼                                                                 │
[5] kernel-implement ───────── implements plan items; updates CURRENT ────┘
           (Builder)                       nextplan doc, then loops to [1]
```

One full cycle = one **round pair**: an audit round R{N} (stages 1-3) plus a development
increment (stages 4-5). Any stage can also be invoked standalone by the user; each skill
declares its loop position and hands off explicitly.

### Stage summary

| # | Skill | Role (paper) | Reads | Writes | Next stage |
|---|-------|--------------|-------|--------|-----------|
| 1 | `kernel-security-audit` | Challenger | prior audit doc, latest reviewfix doc, CURRENT nextplan doc, roadmaps | `docs/review/audits/qa-{date}.md`; audit-skills, probe-sets, evolution-log | 2 (findings>0) / 4 (clean round — then it records the clean round in the CURRENT nextplan doc itself) |
| 2 | `kernel-security-audit-fix` | Reasoner | NEWEST audit doc, CURRENT nextplan doc | code fixes; audit doc statuses; CURRENT nextplan doc (post-fix update); fix-skills, evolution-log | 3 (≥1 fix applied) / 4 (nothing to fix) |
| 3 | `kernel-review-fix` (NEW) | Judge (+ corrective Reasoner) | NEWEST audit doc, CURRENT nextplan doc, the fix-stage diff | repairs of defective fixes; `docs/review/reviewfix/reviewfix-{date}.md`; nextplan/audit status sync; review-skills, fix-skills (FA-*/pitfalls), evolution-log | 4 |
| 4 | `kernel-next-phase` (REPURPOSED) | Strategy selection / Cross-Time-informed planning | latest audit + reviewfix docs, CURRENT nextplan doc, roadmaps, evolution-log | NEW `docs/review/nextplan/next-phase-plan-{date}.md` (rollover); PE-* dev-skills, evolution-log | 5 |
| 5 | `kernel-implement` (NEW) | Reasoner (constructive) | CURRENT nextplan doc | code; CURRENT nextplan doc statuses (fix-only style); IM-*/AD-* dev-skills, evolution-log | 1 |

---

## 3. Ctx2Skill Paper Mapping (v2)

| Paper concept | v2 instantiation |
|---|---|
| **Challenger** (generates probing tasks + rubrics) | `kernel-security-audit` — findings ARE the tasks; each finding's root-cause + invariant + proof obligation is its rubric |
| **Reasoner** (solves guided by evolving skills) | `kernel-security-audit-fix` (corrective) and `kernel-implement` (constructive) |
| **Judge** (neutral, binary, strict) | `kernel-review-fix` — a standalone stage; verdict pass runs **without loading audit-skills or fix-skills** (neutrality, § 3.3 of the paper); a fix PASSES only if ALL its rubric elements hold (root cause eliminated at class level, all paths covered, no regression, no new attack surface) |
| **Proposer / Generator** (per side) | Unchanged from v1 — embedded evolution phases in each skill |
| **Failure routing** (failed → Reasoner side) | Review-fix FAIL verdicts → fix-skills FA-*/pitfall entries with the review diagnosis; implementation rework → dev-skills pitfalls |
| **Success routing** (easily solved → Challenger side) | Fixes that pass review trivially → audit Proposer treats that bug class as mastered → tightens audit-skills toward harder probes (sustains adversarial pressure) |
| **Strict adversarial separation** (no prompt sees the opposing side's skill set) | Audit agents never load fix-skills; fix/implement agents never load audit-skills; **review-fix verdict agents load neither** |
| **Cross-Time Replay** (ρ_h × ρ_e, Laplace-smoothed) | Unchanged, owned by audit; review-fix contributes NEW hard probes: any defect that escaped the fix stage's own convergence gate is by construction a hard case |
| **Judge uses a stronger/independent backbone** | MODE C: Codex is the independent backbone. MODE S: fresh-context native agents at max reasoning tier + mandatory orchestrator file:line re-read (the validated backstop — R169/R177/R178) |

The paper's ablations justify three v2 rules: Challenger evolution matters most (keep audit-skills
evolution mandatory every round); Cross-Time Replay is second (keep probe updates mandatory);
decoupled Proposer/Generator beats merged (keep the two-step evolution in every skill). The
routing ablation ("Joint Outcome" degrades) is why review-fix routes failures ONLY to the fix
side and trivial passes ONLY to the audit side — never both.

---

## 4. Document Architecture

### 4.1 Locations and naming (the "qa rules", applied uniformly)

| Artifact | Directory | Filename | Same-day repeats |
|---|---|---|---|
| Audit report | `docs/review/audits/` | `qa-{YYYY-MM-DD}.md` | `qa-{date}-v2.md`, `-v3`, … |
| Review-fix report | `docs/review/reviewfix/` | `reviewfix-{YYYY-MM-DD}.md` | `reviewfix-{date}-v2.md`, … |
| Next-phase plan | `docs/review/nextplan/` | `next-phase-plan-{YYYY-MM-DD}.md` | `next-phase-plan-{date}-v2.md`, … |

Shared rules (inherited from the QA report convention):

1. **Never overwrite** — glob for `{prefix}-{today}*.md` before creating; append the next `-vN`.
2. **Self-contained header** — Date, round/version, status counts, build/lint status, and (new in
   v2) **Mode: Codex-cooperative | Claude-solo** with session IDs or the substitution note.
3. **CURRENT-document discovery** — newest date wins; within a date, highest `-vN` wins
   (**numeric** compare: v10 > v9 — do not sort lexicographically).
4. **Round continuity** — read the newest prior document of the same type to derive the next
   round number / version; sequential, no gaps or duplicates.

### 4.2 Identifier scheme

- Audit findings: `R{N}-{k}` (implementation), `D{sev}-{PILLAR}-{id}` (design) — unchanged.
- Review-fix defects: `RF{N}-{k}` where N is the **audited round** (reviewfix docs carry the same
  round number as the audit round whose fixes they review; no separate counter).
- Plan documents: keep the `**Version:** {major}.{minor}` line (informational); the dated
  filename is the discovery key.

### 4.3 Plan ownership matrix (who writes which plan sections)

| Plan section | Stage 2 (fix) | Stage 3 (review-fix) | Stage 4 (next-phase) | Stage 5 (implement) | Stage 1 (audit, clean-round only) |
|---|---|---|---|---|---|
| New P0/P1/P2/P3 items from findings | ✔ (from the audit doc) | — | ✔ (carry/re-rank) | — | — |
| Item fix statuses / checkboxes | ✔ | ✔ (verdict sync: verified / reopened) | carry | ✔ (implemented items) | — |
| Key metrics / cumulative stats | ✔ | ✔ (if changed) | ✔ (recompute) | ✔ | ✔ (streak++) |
| Issue velocity row for round N | ✔ | — | carry | — | ✔ (0-finding row) |
| Risk assessment table | ✔ (new risks) | ✔ (reopened risks) | ✔ (re-rank) | — | — |
| 1.0-Preview gate / 0-HIGH streak | ✔ | ✔ | ✔ | ✔ (if gate item completed) | ✔ |
| Rollover (new dated doc, digest, re-rank) | — | — | ✔ (sole owner) | — | — |

Stages 2/3/5 update the CURRENT dated plan doc **in place**; only stage 4 creates a new dated
plan doc. Stages are sequential, so there is never more than one writer at a time.

### 4.4 Monolith migration (one-time, executed by stage 4's first v2 run)

1. Copy `docs/next-phase-plan.md` (v15.12, 5,767 lines) **verbatim** to
   `docs/review/nextplan/next-phase-plan-{header-date}.md` (header date 2026-07-10; if taken,
   apply `-vN`). This snapshot becomes the CURRENT plan doc and the complete historical archive.
2. Verify the copy on BOTH environments (md5 local vs remote) before touching the original.
3. Replace `docs/next-phase-plan.md` (both environments) with a pointer stub: the new directory,
   the discovery rule, and the migration date. Never delete the path — external references and
   memory notes point at it.
4. All subsequent rollovers (stage 4) create compacted dated docs: OPEN items carried forward in
   full; completed rounds compressed to one-line digests pointing at their audit/reviewfix docs;
   full history remains in the older dated snapshots. This permanently bounds live-plan size.
5. OPTIONAL, later: carve individual `## R{n} ({date})` history sections out of the legacy
   snapshot into per-date archive files (only where that date has no snapshot yet), at most 1-2
   per cycle. Never required for correctness.

---

## 5. Codex Cooperation Protocol (the mode split)

Every stage runs with a **driver** (performs the stage's core work) and an independent
**counterparty** (adversarially checks it). Three modes, selected at skill start:

- **MODE C — Codex-cooperative** (default). The Codex MCP tool is available and the first call
  succeeds (`success == true`). Driver = Claude Code; counterparty = Codex, the independent
  second backbone per the CLAUDE.md protocol.
- **MODE S — Claude-solo.** Codex MCP absent, first call fails after one corrected retry, or the
  user says to skip Codex. Driver = Claude Code; counterparty = fresh-context native agent
  fleets. The loop NEVER blocks on Codex (standing directive 2026-07-02).
- **MODE D — Codex-delegated.** Codex ALONE performs the stage's core work. Selected only on
  explicit user request ("let Codex run this stage/round") or when the operator executes the
  skill under the Codex harness itself (as R178's header records: "Codex orchestrator + native
  parallel subsystem auditors"). Two sub-cases — **D-via-MCP**: Claude Code dispatches the full
  stage brief; Codex works in `read-only` sandbox for analysis stages or `workspace-write` on
  the local mirror for write stages; Claude Code retains the NON-DELEGABLE duties (verification
  backstop re-read, remote build/lint/test via ssh-skill, dual-write sync, document-convention
  enforcement, session bookkeeping, and demotion D→C after two failed verifications of the same
  defect). **D-native**: the Codex harness executes the skill file directly; all environment
  invariants and document conventions bind unchanged; Claude-specific surfaces get recorded
  Codex-native substitutions (the R178 "Runtime adaptation" section is the canonical example).

Mode is recorded in every produced document header. Substitutions and demotions are recorded
explicitly ("what replaced what") — never silently claimed as the original pass (R178
precedent). Operational detail: `loop-protocol.md` § 4.

### 5.1 Division of labor (MODE C / MODE S; MODE D inverts driver and counterparty per § 4.3 of loop-protocol.md)

| Step | MODE C (Codex + Claude) | MODE S (Claude only) |
|---|---|---|
| Requirement alignment | Send requirement + preliminary plan to Codex; ask for missing constraints, edge cases, plan refinement | Explicit self-interrogation: enumerate ambiguities/risks/edge cases in the working notes before any change |
| Prototype (fix / feature) | Codex `sandbox="read-only"`, unified-diff ONLY; Claude critically evaluates, then REWRITES to production quality (never applies the prototype verbatim) | Two independent read-only design agents (Workflow) propose approaches from different angles; orchestrator selects/synthesizes and writes |
| Independent audit pass (stage 1) | Codex bug-finding session over the same scope as the agent fleet | One additional fleet of fresh-context native skeptic agents (2 independent passes total, as R178 ran) |
| Adversarial review of an implemented diff | Codex classifies SAFE/UNSAFE/INEFFICIENT/INCOMPLETE/OVER-SCOPED; Claude verifies every Codex claim by re-reading cited code; iterate to convergence (≤3, then escalate) | 2 fresh-context reviewer agents per change (no implementer context, distinct lenses, prompted to REFUTE); orchestrator adjudicates; iterate identically |
| Stage-3 verdict (Judge) | Codex judges each fix against its rubric (paper-faithful independent backbone); Claude re-verifies | Fresh-context judge agents loading NEITHER audit-skills NOR fix-skills; max reasoning tier |
| Plan review (stage 4) | Codex reviews the draft plan: priority sanity, dependency order, mis-ranked severity | Dedicated self-review lenses: dependency-order, severity-consistency, gate-coherence |
| Final arbitration | **Always Claude Code**: re-read every cited file:line; Codex is not authoritative | Orchestrator re-read (identical — this is the invariant backstop) |
| File edits, dual-write, remote exec, git | **Always Claude Code.** Codex never modifies files (read-only sandbox in all loop uses) | Same |

### 5.2 Invariants that hold in BOTH modes

1. Convergence gate semantics unchanged: zero UNSAFE / INCOMPLETE findings before acceptance;
   escalate to the user after 3 failed iterations.
2. The orchestrator independently re-reads every CRITICAL/HIGH finding and every changed region
   at file:line before acceptance — this, not the reviewer count, is the real backstop
   (validated R169; re-validated by the R177 re-run and R178).
3. Direction of fail-closed differs by artifact and is mode-independent:
   **findings fail toward KEEP** (never silently drop a possible bug);
   **changes fail toward KILL** (never ship an unverified fix).
4. All Codex sessions (MODE C and MODE D): `cd` = the **local Windows root** `D:\project\Zero-os`
   (Codex runs locally); store every `SESSION_ID` in the produced document; check
   `success == true`; one corrected-parameter retry, then fall back (C→S for that step; D→C for
   the stage) and record it.
5. Other unavailable tool surfaces degrade the same way (record the substitution): ACE MCP →
   `rg`/`Read` + native agents; Workflow tool → manual structured fan-out.

---

## 6. Per-Skill Deltas (v1 → v2)

### 6.1 `kernel-security-audit`
- Report target: `docs/review/audits/qa-{date}.md` (was `docs/review/`).
- Plan input: CURRENT nextplan doc via discovery rule (was the monolith path).
- **No longer updates the plan** after a findings round — the report instead carries a
  **Plan Impact Handoff** section (proposed P0/P1 items, gate impact, dependency notes) that
  stage 2 materializes into the plan. Exception: on a **clean round** (0 actionable findings)
  stages 2-3 don't run, so the audit itself appends the clean-round record (streak++, velocity
  row) to the CURRENT nextplan doc.
- Phase numbering de-duplicated (two "Phase 3"s and two "Phase 7"s in v1).
- Codex Cooperation Protocol section; MODE S formalizes the R178 pattern.
- Loop handoff section (→ fix, or → next-phase on clean round).

### 6.2 `kernel-security-audit-fix`
- Input: NEWEST audit doc in `docs/review/audits/` (or the user-named round).
- Document updates phase now: audit doc statuses + **CURRENT nextplan doc post-fix update**
  (new items from the audit's Plan Impact Handoff, statuses, metrics, velocity row, risks, gate).
- Codex Cooperation Protocol; MODE S variants for prototype + bidirectional review.
- Loop handoff (→ review-fix normally; → next-phase if zero fixes were applied).

### 6.3 `kernel-review-fix` (NEW)
- Scope the modification set from the audit doc statuses + `// R{N}-{id} FIX:` markers + git
  status/diff; unrelated uncommitted changes are listed OUT-OF-SCOPE, never reviewed silently.
- Verdict pass per fix (Judge): PASS / PARTIAL / FAIL against the finding's rubric; neutrality
  rules; then **repair** every confirmed defect with full fix discipline (root cause, class
  elimination, convergence gate, per-repair build/lint/test).
- Report: `docs/review/reviewfix/reviewfix-{date}.md`, header/section/naming rules mirroring the
  QA format; defects numbered `RF{N}-{k}`.
- Status sync: nextplan + audit doc ("FIXED (review-verified)" / reopened+re-repaired notes).
- Evolution: review-skills.md (RV-* lenses, small, capped); routes FA-*/pitfalls to fix-skills;
  contributes escaped-defect hard probes to probe-sets.
- Loop handoff (→ next-phase).

### 6.4 `kernel-next-phase` (REPURPOSED — planner)
- No longer implements code. Produces the round's NEW dated plan doc (rollover): integrate
  stage 1-3 outcomes, carry OPEN items, re-rank, refresh gate/metrics/risks/dependency map,
  digest completed material.
- Executes the § 4.4 migration on first v2 run.
- Ends by presenting the top 3-5 pending items and handing off to `kernel-implement`.
- Keeps PE-* skill evolution; IM-*/AD-* consumption moves to kernel-implement.
- No build gate (no code changes); consistency checks instead. Codex plan-review in MODE C.

### 6.5 `kernel-implement` (NEW — builder)
- Inherits the v1 kernel-next-phase execution lifecycle: baseline check, work queue from the
  CURRENT nextplan doc, context retrieval, Workflow orchestration for complex items, prototype
  (MODE C/S), central implementation, remote verify, bidirectional convergence, fix-only plan
  update, IM-*/AD-* skill evolution, QA handoff → `kernel-security-audit`.
- References dev-skills.md and fix-skills.md cross-skill (files stay where they are).

---

## 7. Trigger Routing (frontmatter descriptions)

| User intent | Skill |
|---|---|
| "run a security audit / QA round / audit the kernel" | kernel-security-audit |
| "fix the audit findings / fix R{N} issues" | kernel-security-audit-fix |
| "review the fixes / verify the repairs / review-fix round" | kernel-review-fix |
| "update the plan / replan / roll over the next-phase plan / what should we plan next" | kernel-next-phase |
| "continue development / implement the next item / execute plan item X" | kernel-implement |
| "run the loop / continue the cycle" | start at the stage after the last completed one (each doc header names its stage) |

---

## 8. Success Metrics (v2 additions over v1 § 10)

| Metric | Baseline (pre-v2) | Target |
|---|---|---|
| Fix-defect escape rate (defects found by NEXT audit round in prior round's fixes) | measure from R170-R178 | −50% (caught by stage 3 instead) |
| Review-fix verdict yield (RF defects per fix round) | n/a (new) | >0 early (catching what convergence missed), trending → 0 |
| Live plan doc size | 5,767 lines | ≤1,500 lines steady-state |
| Loop stall on tool unavailability | ad-hoc user authorization (R178) | 0 (MODE S auto-engages, recorded) |
| Plan-parse cost per stage | full monolith | CURRENT doc only |

---

## 9. Risks and Mitigations

| Risk | Mitigation |
|---|---|
| Stage 3 rubber-stamps (Judge conditioned by fix context) | Fresh-context verdict agents; neither skill set loaded; verdict BEFORE repair sub-phase; orchestrator re-read |
| Rollover loses open work | Rollover checklist: every non-DONE item in doc N must appear in doc N+1 or be explicitly marked DEFERRED with rationale; diff the open-item ID sets before finalizing |
| Same-day version collisions | Mandatory pre-create glob + numeric -vN rule |
| Stale cross-references to the monolith path | Pointer stub kept at the old path forever; skills use the discovery rule |
| MODE S weaker than MODE C | Invariant backstop (orchestrator re-read) is mode-independent; MODE S adds a second independent fleet; direction-correct fail-closed rules |
| Two stages writing the plan concurrently | Sequential stages + single CURRENT doc + in-place updates only between rollovers |
| Skill-doc cross-writing conflicts (review-fix → fix-skills) | review-fix may only ADD FA-*/pitfall/CA-* entries and examples; only the owner restructures FX-* strategies |

---

## Appendix A — File Inventory (v2 implementation)

| File | Action |
|---|---|
| `docs/skills/skill-loop-v2-spec.md` | NEW — this document |
| `docs/skills/ctx2skill-integration-spec.md` | Header status updated (implemented; topology superseded by this spec; evolution machinery still authoritative) |
| `.claude/skills/shared/loop-protocol.md` | NEW — operational loop reference shared by all five skills |
| `.claude/skills/shared/self-play-protocol.md` | Integration-points table extended to five skills |
| `.claude/skills/kernel-security-audit/SKILL.md` | Updated per § 6.1 |
| `.claude/skills/kernel-security-audit/references/report-format.md` | Path + Mode line + Plan Impact Handoff template |
| `.claude/skills/kernel-security-audit/references/audit-methodology.md` | Plan-update section replaced by the handoff model |
| `.claude/skills/shared/workflow-orchestration.md` | § 10 mode note (C/S/D) |
| `.claude/skills/shared/evolution-log.md` | Stale artifact path corrected to `docs/review/audits/` |
| `.claude/skills/kernel-security-audit-fix/SKILL.md` | Updated per § 6.2 |
| `.claude/skills/kernel-review-fix/SKILL.md` | NEW per § 6.3 |
| `.claude/skills/kernel-review-fix/references/reviewfix-format.md` | NEW — report template |
| `.claude/skills/kernel-review-fix/references/review-skills.md` | NEW — RV-* bootstrap |
| `.claude/skills/kernel-next-phase/SKILL.md` | REWRITTEN per § 6.4 |
| `.claude/skills/kernel-next-phase/references/plan-parsing.md` | Discovery-rule + path updates |
| `.claude/skills/kernel-next-phase/references/plan-update-style.md` | Ownership matrix (§ 4.3) |
| `.claude/skills/kernel-implement/SKILL.md` | NEW per § 6.5 |

All files dual-written to local (`D:\project\Zero-os`) and remote
(`/home/dev/workspace/project/rsproject/Zero-os`). No git operations (manual-commit rule).
