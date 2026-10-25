# Ctx2Skill Integration Specification

## Self-Evolving Skill Framework for Zero-OS Kernel Development

**Version:** 1.0
**Date:** 2026-05-27
**Based on:** "From Context to Skills: Can Language Models Learn from Context Skillfully?" (Si et al., 2026)
**Status:** IMPLEMENTED — the skill-evolution machinery below (skill documents, Proposer/Generator phases, probe sets, Cross-Time Replay) is live in the skill files and remains authoritative. The three-skill **loop topology** in § 1-2 is superseded by the five-stage loop in `skill-loop-v2-spec.md` (2026-07-11): audit → audit-fix → review-fix (Judge) → next-phase (planner) → implement → loop, with relocated documents (`docs/review/audits|reviewfix|nextplan`) and an explicit Codex-cooperative vs Claude-solo mode split.

---

## 1. Motivation & Problem Statement

Our three kernel development skills (`kernel-security-audit`, `kernel-security-audit-fix`, `kernel-next-phase`) already form a natural adversarial loop:

```
  Audit (Challenger)  ──── finds vulnerabilities ────→  Fix (Reasoner)
         ↑                                                     │
         │                                                     │
   next round verifies                                    fixes bugs
   fixes + finds new bugs                                     │
         │                                                     ↓
         └──────────────  Next-Phase (Orchestrator)  ←─────────┘
```

However, this loop is **stateless across rounds**. Each audit round starts from the static methodology in `audit-methodology.md`. Each fix round uses the same static patterns in `workflow-details.md`. The system has no mechanism to:

1. **Accumulate detection expertise** — Which probing strategies yield the most true positives? Which subsystems consistently harbor which bug classes?
2. **Accumulate fix expertise** — Which fix approaches converge fastest? Which patterns recur and should be addressed structurally rather than per-instance?
3. **Prevent adversarial collapse** — Are we over-specializing on one bug class (e.g., unbounded allocation) while neglecting others (e.g., namespace isolation)?
4. **Learn from failure** — When a finding is a false positive, or a fix fails convergence, that diagnostic insight is lost.

With 160+ QA rounds completed, these are not hypothetical problems — they represent accumulated knowledge that is currently only implicit in the report archive.

---

## 2. Architecture Overview

### 2.1 Ctx2Skill Concept Mapping

| Ctx2Skill Concept | Our System | Role |
|---|---|---|
| **Challenger** | `kernel-security-audit` | Generates probing findings (tasks + rubrics) |
| **Reasoner** | `kernel-security-audit-fix` | Solves findings (implements fixes) |
| **Judge** | Cross-review (audit Phase 5, fix Phase 4) | Binary verdict on each finding/fix |
| **Challenger Skills** | `docs/skills/audit-skills.md` | Evolving audit methodology |
| **Reasoner Skills** | `docs/skills/fix-skills.md` | Evolving fix patterns & strategies |
| **Proposer** (per side) | New phase in each skill | Diagnoses why findings/fixes succeeded or failed |
| **Generator** (per side) | New phase in each skill | Updates the skill document based on diagnosis |
| **Cross-Time Replay** | `docs/skills/probe-sets.md` | Prevents over-specialization across rounds |
| **Dev Skills** | `docs/skills/dev-skills.md` | Evolving implementation patterns (next-phase) |

### 2.2 Co-Evolution Loop

```
Round N:
  1. Audit loads audit-skills.md → guides 10 agents' probing strategies
  2. Audit produces findings → Judge (cross-review) filters to true positives
  3. Audit Proposer analyzes: which skills led to true positives? which to false positives?
  4. Audit Generator updates audit-skills.md

  5. Fix loads fix-skills.md → guides fix approach selection
  6. Fix implements fixes → Judge (convergence gate) validates each fix
  7. Fix Proposer analyzes: which patterns converged quickly? which failed?
  8. Fix Generator updates fix-skills.md

  9. Cross-Time Replay evaluates: are we over-specialized? Update probe sets.
  10. Next-Phase loads dev-skills.md for any remaining plan items

Round N+1:
  → Audit uses updated audit-skills.md (evolved Challenger strategy)
  → Fix verification uses historical context from fix-skills.md
  → Loop continues
```

### 2.3 Key Design Constraints

1. **Skills are natural-language Markdown** — readable, editable, auditable by humans
2. **Skills are prepended to agent prompts** — just like Ctx2Skill prepends to LM system prompts
3. **No parameter updates** — all evolution is through document edits, not model fine-tuning
4. **Skills are context-specific** — tied to the Zero-OS kernel codebase, not generic
5. **Dual-write** — all skill documents written to both local and remote

---

## 3. Skill Document Specifications

### 3.1 Audit Skills (`docs/skills/audit-skills.md`)

**Owner:** `kernel-security-audit` skill
**Consumed by:** Audit agents (Phase 4b), Codex design review (Phase 3)
**Updated after:** Each audit round (Phase 7.5)

**Structure:**

```markdown
# Audit Skills — Evolved Detection Methodology
# Version: {N} (updated after Round {R})

## Meta
- Total rounds contributing: {count}
- Last updated: {date}
- Skill selection method: Cross-Time Replay (see probe-sets.md)

## Vulnerability Detection Skills

### VD-{id}: {Skill Name}
- **Bug class:** {e.g., unbounded allocation, lock ordering violation, TOCTOU}
- **Detection strategy:** {Specific code patterns to search for, grep patterns, structural indicators}
- **Subsystem affinity:** {Which subsystems this skill is most effective in}
- **Historical yield:** {True positive rate across rounds where this skill was active}
- **Probe pattern:** {Concrete code pattern that reliably indicates this bug class}
- **False positive indicators:** {Patterns that look like this bug class but aren't}
- **Example findings:** {R###-# IDs of confirmed findings discovered by this skill}

### VD-{id}: ...

## Subsystem Risk Profile

### {Subsystem Name}
- **Historical bug density:** {bugs per round, trend}
- **Dominant bug classes:** {ranked list}
- **Known-clean areas:** {areas audited multiple times with no findings — reduce attention}
- **Persistent weak spots:** {areas that keep producing findings — increase attention}
- **Coverage gaps:** {areas not yet thoroughly audited}

## Anti-Patterns (False Positive Avoidance)

### FP-{id}: {Pattern Name}
- **Description:** {What looks like a bug but isn't}
- **Why it's safe:** {The invariant or design choice that makes it correct}
- **Rounds where this was misreported:** {R### list}
- **How to distinguish from real bugs:** {Differentiating criteria}

## Probing Strategy Evolution Log

### Round {R}: {Summary}
- Skills added: {list}
- Skills refined: {list}
- Skills deprecated: {list}
- True positive rate: {X}%
- New bug classes discovered: {list}
- Adversarial collapse indicators: {any over-specialization detected?}
```

### 3.2 Fix Skills (`docs/skills/fix-skills.md`)

**Owner:** `kernel-security-audit-fix` skill
**Consumed by:** Fix implementation (Phase 3), Codex prototype (Phase 3a)
**Updated after:** Each fix round (Phase 6.5)

**Structure:**

```markdown
# Fix Skills — Evolved Remediation Patterns
# Version: {N} (updated after Round {R} fixes)

## Meta
- Total fix rounds contributing: {count}
- Last updated: {date}
- Total fixes tracked: {count}
- Average convergence iterations: {N}

## Fix Pattern Skills

### FX-{id}: {Pattern Name}
- **Bug class addressed:** {e.g., lock-cleanup deadlock, unbounded Vec}
- **Fix strategy:** {Detailed approach — targeted vs aggressive, with decision criteria}
- **Code template:** {Concrete code pattern for the fix}
- **Convergence history:** {Average iterations to converge, common Codex objections}
- **Pitfalls:** {What went wrong when this was applied incorrectly}
- **When to escalate to aggressive:** {Specific indicators that targeted fix is insufficient}
- **Example fixes:** {R###-# IDs and file:line references}
- **Effectiveness:** {Recurrence rate — does this bug class come back after this fix?}

### FX-{id}: ...

## Convergence Acceleration Skills

### CA-{id}: {Skill Name}
- **Scenario:** {When this skill applies — e.g., "Codex flags UNSAFE on lock-based fix"}
- **Resolution strategy:** {How to resolve the disagreement efficiently}
- **Evidence pattern:** {What code evidence convinces Codex / Claude Code}
- **Historical success rate:** {How often this resolution strategy worked}

## Failed Approach Registry

### FA-{id}: {Approach Name}
- **Bug class:** {What was being fixed}
- **What was tried:** {The approach that failed}
- **Why it failed:** {Root cause of failure — missed edge case, wrong abstraction, etc.}
- **What worked instead:** {The successful alternative}
- **Round:** {R###}
- **Lesson:** {Generalizable insight}

## Fix Strategy Evolution Log

### Round {R}: {Summary}
- Patterns added: {list}
- Patterns refined: {list}
- Failed approaches recorded: {list}
- Average convergence iterations: {N}
- Aggressive vs targeted ratio: {X/Y}
```

### 3.3 Development Skills (`docs/skills/dev-skills.md`)

**Owner:** `kernel-next-phase` skill
**Consumed by:** Implementation phase (Phase 4), approach selection
**Updated after:** Each development item completion (Phase 7.5)

**Structure:**

```markdown
# Development Skills — Evolved Implementation Patterns
# Version: {N} (updated after {item})

## Meta
- Total items contributing: {count}
- Last updated: {date}

## Implementation Skills

### IM-{id}: {Skill Name}
- **Task type:** {e.g., systematic audit, feature implementation, design finding resolution}
- **Effective approach:** {What works well for this type of task}
- **Subsystem context:** {Which subsystems this applies to}
- **Common pitfalls:** {What to avoid}
- **Verification strategy:** {How to confirm correctness beyond build/lint/test}
- **Example items:** {Plan item IDs}

### IM-{id}: ...

## Architecture Decision Skills

### AD-{id}: {Decision Name}
- **Context:** {When this decision type arises}
- **Preferred approach:** {What we've learned works best}
- **Trade-offs:** {What was considered and rejected, and why}
- **Safety implications:** {How this affects the Safety > Efficiency > Speed hierarchy}

## Plan Execution Skills

### PE-{id}: {Skill Name}
- **Scenario:** {e.g., "multiple P1 items with shared dependencies"}
- **Execution strategy:** {Ordering, batching, verification approach}
- **Historical outcome:** {How well this worked in past rounds}
```

### 3.4 Probe Sets (`docs/skills/probe-sets.md`)

**Owner:** Cross-Time Replay mechanism (shared across skills)
**Updated after:** Each audit round

**Structure:**

```markdown
# Probe Sets — Cross-Time Replay Data
# Version: {N}
# Last updated: {date}

## Purpose

Probe sets prevent adversarial collapse by maintaining representative test cases
that evaluate whether evolving skills remain balanced. Inspired by Ctx2Skill's
Cross-Time Replay mechanism.

## Hard Probe Set (Qh)

Bugs that were MISSED by audits or took multiple rounds to discover.
These probe the audit skills' ability to find non-obvious vulnerabilities.

| Probe ID | Bug Description | Round First Missed | Round Finally Found | Bug Class | Subsystem |
|----------|----------------|-------------------|--------------------|-----------|-----------| 
| HP-{id}  | {description}  | R{N}              | R{M}               | {class}   | {subsys}  |

## Easy Probe Set (Qe)

Bugs that were found quickly and repeatedly — canonical examples of each bug class.
These probe whether audit skills maintain baseline detection capability.

| Probe ID | Bug Description | Canonical Round | Bug Class | Subsystem | Detection Skill |
|----------|----------------|-----------------|-----------|-----------|-----------------|
| EP-{id}  | {description}  | R{N}            | {class}   | {subsys}  | VD-{id}         |

## Skill Evaluation History

Records the "utility score" (ρ_h × ρ_e) for each skill set version.

| Skill Version | ρ_h (hard rate) | ρ_e (easy rate) | Utility Score | Selected? |
|---------------|-----------------|-----------------|---------------|-----------|
| audit-v{N}    | {rate}          | {rate}          | {score}       | {yes/no}  |

## Adversarial Collapse Indicators

Track whether audits are over-specializing:

- **Bug class concentration:** If >60% of findings in last 3 rounds are from the same bug class → collapse warning
- **Subsystem concentration:** If >50% of findings are from the same 2 subsystems → collapse warning
- **Declining easy rate:** If ρ_e drops below 0.7 → skills may have regressed on basics
- **New bug class discovery rate:** If 0 new bug classes in last 5 rounds → exploration deficit

## Current Status
- Last evaluation: Round {R}
- Current selected audit skill version: audit-v{N}
- Collapse warnings: {none | list}
```

### 3.5 Evolution Log (`docs/skills/evolution-log.md`)

**Owner:** Shared across all skills
**Purpose:** Audit trail for skill changes, enabling rollback and trend analysis

**Structure:**

```markdown
# Skill Evolution Log

## Entry Format

### {Date} — Round {R} — {Skill Type} — {Action}
- **Trigger:** {What triggered this evolution — finding pattern, failure analysis, etc.}
- **Proposer diagnosis:** {What the Proposer identified}
- **Generator action:** {What was added/modified/removed in the skill document}
- **Rationale:** {Why this change improves the skill set}
- **Cross-Time Replay result:** {Was this version selected or superseded?}

## Entries (newest first)

(Initially empty — populated as the system runs)
```

---

## 4. Skill Modifications — kernel-security-audit

### 4.1 New Phase 1.5: Load Audit Skills

Insert between current Phase 1 (Prepare) and Phase 2 (Codebase Indexing).

**Purpose:** Load the evolved audit skill set to guide the current round's probing strategy.

**Steps:**

1. Read `docs/skills/audit-skills.md` (if exists; if not, this is the bootstrap round)
2. Read `docs/skills/probe-sets.md` (if exists)
3. Check for adversarial collapse indicators:
   - If collapse warning is active, explicitly diversify agent focus areas
   - If easy rate (ρ_e) < 0.7, ensure agents cover all basic bug classes before deep probing
4. Inject audit skills into:
   - Phase 3 Codex design review prompt (append vulnerability detection skills as context)
   - Phase 4b agent prompts (each agent receives relevant VD-{id} skills for its subsystem)
   - Phase 4a Codex bug-finding prompt (append full skill set)
5. Record which skill version is being used for this round

**Agent Prompt Modification:**

Current agent prompts receive static focus areas. New prompts will additionally receive:

```
## Evolved Audit Skills for This Subsystem

The following detection skills have been learned from {N} prior audit rounds.
Use them to guide your probing strategy, but do NOT limit yourself to these —
also look for novel bug classes.

{Relevant VD-{id} skills filtered by subsystem affinity}

## Anti-Patterns (avoid false positives)

{Relevant FP-{id} entries filtered by subsystem}

## Coverage Gaps to Explore

{Subsystem risk profile: coverage gaps and persistent weak spots}
```

### 4.2 New Phase 5.5: Audit Skill Proposer

Insert after Phase 5 (Cross-Review) and before Phase 6 (Report).

**Purpose:** Diagnose which audit skills were effective and which were not, producing a high-level skill evolution diagnosis.

**Steps:**

1. **Classify each finding by source skill:**
   - For each confirmed finding (TRUE POSITIVE from cross-review), identify which audit skill (VD-{id}) led to its discovery, OR tag as "novel" if no existing skill covers it
   - For each rejected finding (FALSE POSITIVE from cross-review), identify which skill produced it

2. **Compute per-skill metrics:**
   - True positive count per VD-{id}
   - False positive count per VD-{id}
   - Precision = TP / (TP + FP) per skill
   - Novel findings not covered by any existing skill

3. **Proposer diagnosis format:**

```markdown
## Audit Skill Proposer Diagnosis — Round {R}

### Effective Skills (high precision, TP > 0)
| Skill | TP | FP | Precision | Notes |
|-------|----|----|-----------|-------|

### Ineffective Skills (FP > TP, or 0 TP)
| Skill | TP | FP | Precision | Problem |
|-------|----|----|-----------|---------|

### Novel Findings (not covered by existing skills)
| Finding | Bug Class | Subsystem | Proposed New Skill |
|---------|-----------|-----------|-------------------|

### Subsystem Risk Profile Updates
| Subsystem | New bug density | Trend | Action |
|-----------|----------------|-------|--------|

### Adversarial Collapse Check
- Bug class distribution: {class → count}
- Top 2 classes account for: {X}% of findings
- New bug classes discovered: {count}
- Recommendation: {diversify / maintain / specialize}
```

4. **Cross-Time Replay evaluation:**
   - Re-evaluate the current audit skill set against probe sets
   - Compute ρ_h (hard probe solve rate) and ρ_e (easy probe solve rate)
   - Compute utility score = ρ_h × ρ_e
   - Compare with historical utility scores
   - If current version scores highest → keep; otherwise, flag best historical version

### 4.3 New Phase 5.6: Audit Skill Generator

Immediately follows Phase 5.5.

**Purpose:** Materialize the Proposer's diagnosis into concrete skill document updates.

**Steps:**

1. For each **effective skill with refinement opportunity**: update detection strategy, add new examples, increase confidence markers
2. For each **ineffective skill**: add false positive indicators, narrow detection criteria, or deprecate if consistently low precision across 3+ rounds
3. For each **novel finding**: create a new VD-{id} skill entry with detection strategy, probe pattern, subsystem affinity
4. Update **subsystem risk profiles** with latest bug density and trend
5. Update **anti-patterns** with any new false positive patterns discovered
6. Append entry to **probing strategy evolution log**
7. If Cross-Time Replay selected a different version, **rollback to that version** and apply only the additions from this round

**Skill Update Rules (prevent adversarial collapse):**
- Never remove a skill that had TP > 0 in the last 3 rounds
- If adding a new skill pushes total count > 30, merge the two lowest-yield skills
- If bug class concentration > 60%, add at least one skill for an underrepresented class
- Skills not yielding TP for 5+ consecutive rounds are candidates for deprecation

### 4.4 New Phase 7.5: Update Probe Sets

Insert after Phase 7 (Update Plan).

**Purpose:** Maintain the hard and easy probe sets for Cross-Time Replay.

**Steps:**

1. **Hard probe set (Qh) update:**
   - Add the finding from this round with the lowest rubric-equivalent score: the bug that was hardest to detect (most agents missed it, or it was found only via novel skill)
   - If a previously-missed bug class is finally found, add it as a hard probe

2. **Easy probe set (Qe) update:**
   - Add the finding that was found by the most agents / was most obvious
   - Ensure each major bug class has at least one representative in Qe

3. **Probe set size management:**
   - Cap each set at 20 entries (matching Ctx2Skill's per-iteration collection)
   - When full, replace the oldest entry from the same bug class
   - Ensure diversity: no bug class has > 4 entries in either set

4. **Record utility scores** for the current skill version

---

## 5. Skill Modifications — kernel-security-audit-fix

### 5.1 New Phase 0.5: Load Fix Skills

Insert before current Phase 1 (Triage).

**Purpose:** Load the evolved fix skill set to guide approach selection.

**Steps:**

1. Read `docs/skills/fix-skills.md` (if exists)
2. For each finding to be fixed, look up:
   - Matching FX-{id} pattern by bug class
   - Historical convergence data for this bug class
   - Failed approach registry (FA-{id}) — what NOT to try
3. Pre-select fix approach based on skill recommendations:
   - If the skill says "aggressive required for this pattern" → start with aggressive
   - If the skill says "targeted converges in 1 iteration" → start with targeted
   - If the failed approach registry has an entry for this bug class → avoid that approach

**Integration with Phase 1 (Triage):**

Current triage uses static severity-to-approach mapping. New triage will additionally consult:

```
## Fix Skills Context for {Bug Class}

### Recommended Pattern: FX-{id}
{Strategy, code template, convergence history}

### Known Pitfalls:
{Failed approaches for this bug class — FA-{id} entries}

### Convergence Tips:
{Convergence acceleration skills — CA-{id} entries}
```

### 5.2 New Phase 4.5: Fix Skill Proposer (on convergence failure)

Triggered when Phase 4d (Iterate Until Convergence) requires 2+ iterations.

**Purpose:** Diagnose WHY convergence was slow and extract reusable insight.

**Steps:**

1. Analyze the convergence trajectory:
   - What did Codex flag as UNSAFE/INCOMPLETE in each iteration?
   - What did Claude Code initially miss in self-review?
   - What was the root cause of each iteration (wrong abstraction? missed edge case? scope mismatch?)

2. Produce diagnosis:

```markdown
## Fix Skill Proposer — Convergence Analysis for {R###-#}

### Iterations Required: {N}
### Root Cause of Extra Iterations:
- Iter 1 → 2: {What was wrong with the first attempt}
- Iter 2 → 3: {What was still wrong}

### Generalizable Lesson:
- {e.g., "Lock-based fixes for this pattern always miss the error cleanup path — 
  need to check cleanup explicitly as a standard step"}

### Proposed Skill Update:
- Action: {add / refine / add-pitfall}
- Target: {FX-{id} or new FA-{id}}
- Content: {What to add to the skill}
```

### 5.3 New Phase 6.5: Fix Skill Generator

Insert after Phase 6 (Document Updates).

**Purpose:** Update fix skills based on round outcomes.

**Steps:**

1. For each fix that **converged in 1 iteration**: reinforce the used FX-{id} pattern (increase confidence, add as example)
2. For each fix that **required 2+ iterations**: apply the Phase 4.5 Proposer's diagnosis — add pitfalls, refine strategy, or create CA-{id} entry
3. For each fix that **escalated to user**: record as FA-{id} failure case
4. For fixes that **eliminated a vulnerability class** (not just instance): mark the FX-{id} as "class-eliminating" and note the approach
5. Update convergence statistics
6. Append to evolution log

**Skill Update Rules:**
- Never remove an FX-{id} pattern that has been successfully used in the last 5 rounds
- If a pattern's recurrence rate is > 0 (bug class comes back), flag for review — the pattern may need escalation to aggressive
- Merge similar patterns if they share >80% of their steps
- Cap total FX-{id} entries at 25; merge lowest-yield entries when exceeded

---

## 6. Skill Modifications — kernel-next-phase

### 6.1 New Phase 0.5: Load Development Skills

Insert before current Phase 0 (Baseline Check).

**Purpose:** Load evolved development skills to guide implementation approach.

**Steps:**

1. Read `docs/skills/dev-skills.md` (if exists)
2. For the current plan item, look up:
   - Matching IM-{id} skill by task type
   - Relevant AD-{id} architecture decisions
   - PE-{id} plan execution strategies for similar item patterns

### 6.2 Modified Phase 4: Cross-Time Selection for Approach Decisions

When multiple implementation approaches exist (decision point), evaluate candidates against the probe-set-equivalent for development: the set of past items that required rework.

**Steps:**

1. If multiple approaches exist, evaluate each against:
   - Historical success rate for similar approaches (from dev-skills)
   - Safety hierarchy compliance (Safety > Efficiency > Speed)
   - Convergence likelihood (from fix-skills convergence data for similar patterns)
2. Present the top approach with supporting evidence from skills
3. Only fall back to user decision when skills don't clearly favor one approach

### 6.3 New Phase 7.5: Development Skill Proposer + Generator

Insert after Phase 7 (Update Plan).

**Purpose:** Update dev skills based on implementation outcomes.

**Steps:**

**Proposer:**
1. Analyze the implementation outcome:
   - Did the approach work first try, or require iteration?
   - Did the convergence gate reveal issues?
   - Were there unexpected dependencies or complications?
2. Produce diagnosis of what was learned

**Generator:**
1. If approach was effective and non-obvious: create/update IM-{id} skill
2. If an architecture decision was validated: create/update AD-{id} skill
3. If the plan execution strategy worked well: create/update PE-{id} skill
4. If approach required rework: record the pitfall in the relevant skill
5. Append to evolution log

---

## 7. Cross-Time Replay Protocol (Detailed)

### 7.1 When to Trigger

Cross-Time Replay evaluation runs:
- **Mandatory:** At the end of every audit round (Phase 5.5)
- **Optional:** When any adversarial collapse indicator fires mid-round
- **On demand:** When the user requests a skill evaluation

### 7.2 Evaluation Procedure

Following the paper's mechanism, adapted for our system:

```
For each audit skill version v in {audit-v1, audit-v2, ..., audit-vN}:
    For each probe in Hard Probe Set (Qh):
        Simulate: would skill version v have detected this bug?
        (Use the skill's detection strategy + probe pattern match)
        Record: detected(v, probe) = 0 or 1
    
    For each probe in Easy Probe Set (Qe):
        Simulate: would skill version v still detect this canonical bug?
        Record: detected(v, probe) = 0 or 1
    
    Compute:
        ρ_h(v) = (Σ detected(v, Qh) + 1) / (|Qh| + 1)    # Laplace smoothed
        ρ_e(v) = (Σ detected(v, Qe) + 1) / (|Qe| + 1)    # Laplace smoothed
        utility(v) = ρ_h(v) × ρ_e(v)

Select v* = argmax_v utility(v)
```

**Adaptation note:** In the paper, the Reasoner actually re-answers all probe tasks under each skill set. In our system, full re-execution is expensive (10 agents per evaluation). Instead, we use a **lightweight simulation**: match each probe's bug pattern against each skill version's detection strategies. A probe is "detected" if the skill version has a VD-{id} entry whose probe pattern matches the probe's bug class AND subsystem. This is sound because our skills explicitly encode detection patterns, not implicit model behavior.

### 7.3 Adversarial Collapse Prevention

**Detection signals:**

| Signal | Threshold | Action |
|--------|-----------|--------|
| Bug class concentration | >60% findings from 1 class in last 3 rounds | Add diversification skill; penalize concentrated class in agent prompts |
| Subsystem concentration | >50% findings from 2 subsystems in last 3 rounds | Reassign agent attention weights |
| Easy rate decline | ρ_e < 0.7 | Rollback to last version with ρ_e ≥ 0.8 |
| Zero new bug classes | 0 new classes in last 5 rounds | Add explicit exploration directive to agents |
| Skill set bloat | >30 VD-{id} entries | Merge low-yield skills; prune zero-TP skills older than 5 rounds |

**Recovery actions:**

1. **Soft recovery:** Add diversification directives to agent prompts without changing skill document
2. **Medium recovery:** Rollback to Cross-Time Replay selected version + add only new-class skills from recent rounds
3. **Hard recovery:** Reset to a manually curated baseline skill set (escape hatch)

---

## 8. Bootstrap Protocol

The system starts with no skill documents. The bootstrap process:

### Round 0 (First round with Ctx2Skill integration):

1. **Audit Skills bootstrap:**
   - Mine the last 10 QA reports (most recent) for:
     - Confirmed bug classes → create initial VD-{id} skills
     - Known false positive patterns → create initial FP-{id} entries
     - Per-subsystem bug density → create initial risk profiles
   - Set version to audit-v1

2. **Fix Skills bootstrap:**
   - Mine the last 10 fix rounds for:
     - Successful fix patterns by bug class → create initial FX-{id} skills
     - Convergence history → populate convergence data
     - Known failed approaches → create initial FA-{id} entries
   - Set version to fix-v1

3. **Probe Sets bootstrap:**
   - Hard set: Select 10 bugs that were missed for 2+ rounds before discovery
   - Easy set: Select 10 canonical bugs (one per major bug class)

4. **Dev Skills bootstrap:**
   - Mine plan completion history for recurring implementation patterns
   - Set version to dev-v1

### Subsequent Rounds:

Normal Proposer → Generator → Cross-Time Replay cycle.

---

## 9. Implementation Plan

### Phase A: Create skill documents (this step)

| File | Action | Content |
|------|--------|---------|
| `docs/skills/ctx2skill-integration-spec.md` | Create | This document |
| `docs/skills/audit-skills.md` | Create (bootstrap) | Initial audit skills from last 10 reports |
| `docs/skills/fix-skills.md` | Create (bootstrap) | Initial fix skills from last 10 fix rounds |
| `docs/skills/dev-skills.md` | Create (bootstrap) | Initial dev skills from plan history |
| `docs/skills/probe-sets.md` | Create (bootstrap) | Initial hard/easy probe sets |
| `docs/skills/evolution-log.md` | Create (empty) | Evolution audit trail |

### Phase B: Add shared reference document

| File | Action |
|------|--------|
| `.claude/skills/shared/self-play-protocol.md` | Create — shared reference for all 3 skills describing the Proposer/Generator/Replay protocol |

### Phase C: Modify skill files

| Skill File | Changes |
|------------|---------|
| `kernel-security-audit/SKILL.md` | Add Phase 1.5, 5.5, 5.6, 7.5 |
| `kernel-security-audit/references/skill-evolution.md` | Create — detailed Proposer/Generator/Replay procedures for audit |
| `kernel-security-audit-fix/SKILL.md` | Add Phase 0.5, 4.5, 6.5 |
| `kernel-security-audit-fix/references/skill-evolution.md` | Create — detailed Proposer/Generator procedures for fix |
| `kernel-next-phase/SKILL.md` | Add Phase 0.5, 4 modification, 7.5 |
| `kernel-next-phase/references/skill-evolution.md` | Create — detailed Proposer/Generator procedures for dev |

### Phase D: Dual-write to remote

All files from Phases A-C uploaded to remote via SSH.

---

## 10. Success Metrics

After 5 rounds with Ctx2Skill integration, evaluate:

| Metric | Baseline (pre-integration) | Target |
|--------|--------------------------|--------|
| True positive rate per round | Measure from last 5 rounds | +15% improvement |
| False positive rate per round | Measure from last 5 rounds | -30% reduction |
| Average convergence iterations (fix) | Measure from last 5 rounds | -20% reduction |
| New bug class discovery rate | Measure from last 10 rounds | Maintain or improve |
| Bug class coverage (Gini coefficient) | Measure from last 5 rounds | More balanced (lower Gini) |
| Audit skill set utility score (ρ_h × ρ_e) | N/A (new metric) | Monotonically non-decreasing |

---

## 11. Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Skill document becomes too large (>5000 words) | Enforce caps per section; merge/prune low-yield entries |
| Bootstrap from historical data produces low-quality initial skills | Manual review of bootstrap output; iterate in first 2 rounds |
| Cross-Time Replay simulation diverges from actual detection | Periodically validate simulation against real audit results |
| Skill evolution adds overhead to each round | Budget Phase 5.5/5.6 at <10 minutes; automate metric computation |
| Over-reliance on skills suppresses novel bug discovery | Always include "explore beyond skills" directive in agent prompts |
| Dual-write failure desynchronizes local/remote skill documents | Treat remote as authoritative; verify sync at round start |

---

## Appendix A: Ctx2Skill Paper — Key Formulas Applied

### Utility Score (Cross-Time Replay selection)

```
ρ_h(v) = (Σ_{q ∈ Qh} detected(v, q) + 1) / (|Qh| + 1)
ρ_e(v) = (Σ_{q ∈ Qe} detected(v, q) + 1) / (|Qe| + 1)
v* = argmax_v [ ρ_h(v) × ρ_e(v) ]
```

The multiplicative form ensures:
- A skill set that only finds hard bugs but misses easy ones → penalized by ρ_e
- A skill set that only finds easy bugs but misses hard ones → penalized by ρ_h
- Laplace smoothing prevents division by zero with empty probe sets

### Adversarial Collapse Detection

```
concentration(class) = count(class) / total_findings
collapse_warning = any(concentration(class) > 0.6 for class in bug_classes)
```

### Skill Routing (Ctx2Skill's core mechanism)

```
For finding f in round R:
  If f is TRUE POSITIVE:
    Route to Challenger Proposer → "why was this easy to find? Tighten skills"
    (In our case: the finding was too obvious → Challenger should generate harder probes)
  If f is FALSE POSITIVE:
    Route to Reasoner Proposer → "why did we get this wrong? Update detection skills"
    (In our case: add to anti-pattern registry FP-{id})
  If f is NOVEL (no existing skill covers it):
    Route to Reasoner Generator → "create a new detection skill"
    (In our case: create new VD-{id})
```

Note: In our system, the routing is inverted from the paper because we are evolving the Challenger (audit) skills, not the Reasoner skills, when analyzing audit results. The paper's Challenger evolves from easily-solved cases; our audit evolves from both successes (to reinforce) and failures (to correct). This is because our "Challenger" IS the audit itself — we want it to get better at finding bugs, not harder at generating challenges.
