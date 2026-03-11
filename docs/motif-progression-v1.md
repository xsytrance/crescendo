# Motif Progression v1

This document defines how motif choices are awarded, selected, and resolved during a run.

## 1) Trigger moments that award motif choices

Motif choices are granted by two trigger types.

### 1.1 Fixed interval triggers
- **Interval**: every **4 completed encounters**.
- **First grant**: after encounter 4.
- **Subsequent grants**: encounter 8, 12, 16, etc.
- **Cap interaction**: if motif slots are full, the trigger still occurs and follows overflow behavior (see section 4).

### 1.2 Event-based triggers
- **Boss clear trigger**: grant 1 motif choice after each boss encounter completion.
- **Milestone trigger**: grant 1 motif choice when the run score first reaches each multiple of **1000** (1000, 2000, 3000, ...).
- **Single-fire rule**: each specific milestone value can trigger at most once per run.
- **Stacking rule**: if multiple triggers happen at the same moment, queue and resolve each motif choice independently in trigger order: fixed interval -> boss clear -> score milestone.

---

## 2) Choice model and reroll rules

### 2.1 Base choice model
- Each motif choice presents **3 distinct motifs**.
- Player must select exactly **1 motif** or use a reroll if available.
- Offered motifs are sampled without replacement from the currently eligible motif pool.

### 2.2 Rerolls
- **Starting rerolls per run**: 1.
- **Max stored rerolls**: 3.
- **Reroll action**: replaces all 3 offered motifs with a fresh set of 3 eligible motifs.
- **Reroll cost**: consumes 1 reroll.
- **Reroll limit per choice screen**: 1 reroll maximum.
- **Post-reroll guarantee**: at least 1 option must differ from the pre-reroll set.
- **No-reroll state**: if rerolls are 0, player must pick from the current 3 options.

---

## 3) Rarity table with drop weights and unlock conditions

Rarity applies when sampling offered motifs.

| Rarity | Weight | Base unlock condition | Additional condition |
|---|---:|---|---|
| Common | 70 | Always unlocked | None |
| Rare | 25 | Reach encounter 4 in current run | None |
| Epic | 5 | Reach encounter 8 in current run | None |

### 3.1 Sampling notes
- Roll rarity first by normalized weight across currently unlocked rarities.
- If no eligible motif exists in rolled rarity, fallback to next lower rarity with eligible motifs (Epic -> Rare -> Common).
- Duplicate motifs already owned are ineligible unless motif explicitly supports stacking (none do in v1 starter pool).

---

## 4) Maximum motif slots and overflow behavior

- **Maximum active motif slots**: 6.
- If active slots < 6, selected motif is added to the next free slot.
- If active slots = 6, selected motif goes to overflow resolution:
  1. Prompt player to select exactly 1 currently active motif to replace.
  2. Remove selected old motif.
  3. Add newly selected motif in the vacated slot.
- Overflow replacement is mandatory; skipping is not allowed.

---

## 5) Conflict rules for contradictory motifs

### 5.1 Contradiction definition
Two motifs are contradictory if one requires a state/value that the other forbids or negates for the same target and timing window.

### 5.2 v1 contradiction handling
- **Paradox system**: **inactive in v1**.
- Contradictory motifs cannot coexist.
- During offer generation, motifs that would contradict any currently active motif are removed from eligibility.
- During overflow replacement, contradiction is checked only against the post-replacement set.
- If all 3 options would be contradictory, regenerate offer once with contradiction filtering; if still empty, offer 3 Common motifs ignoring rarity weights but still enforcing contradiction filtering.

### 5.3 Conflict-resolution precedence for shared variables

When multiple active motifs modify the same target variable in the same hook window, resolve in this strict order:

1. **Aggregate additive terms** (`+x`, `-x`) by summation.
2. **Apply multiplicative terms** (`*k`) to the post-additive value.
3. **Apply clamps** in order: first minimum clamps (`max(value, min_i)`), then maximum clamps (`min(value, max_j)`).
4. **Tie-break deterministic ordering**: if motifs still conflict with non-commutative operations in the same tier, apply by ascending motif ID (`M001` < `M002` < ...).

This precedence is global for v1 and applies to score, attack, defense/shield, cost, healing, and resonance-adjacent variables.


---

## 6) Starter pool (10 motifs)

Effect text below is authoritative and implementation-oriented.

| ID | Name | Rarity | Tags | Timing hooks | Stacking behavior | Effect (machine-testable) |
|---|---|---|---|---|---|---|
| M001 | Sharp Opening | Common | damage, start-of-encounter | `on_turn_start` (encounter turns 1-3) | Additive to base attack; stacks additively with other attack buffs, then multiplicative effects | `At encounter start, add +2 to base attack for the first 3 turns of that encounter.` |
| M002 | Guarded Entry | Common | defense, start-of-encounter | `on_turn_start` (encounter turn 1) | Additive flat shield grant; merges additively with same-hook shield grants | `At encounter start, grant 12 shield. Shield expires at encounter end.` |
| M003 | Rhythm Step | Common | tempo | `on_match` / `on_action_resolved` (every 4th player action) | Additive action-point grant; multiple triggers in same window sum before AP cap checks | `Every 4th player action in an encounter, gain +1 action point immediately after that action resolves.` |
| M004 | Clean Finish | Common | execute | `on_match` (hit resolution) | Additive true-damage rider applied before hit-result clamps | `When hitting an enemy with HP <= 15, deal +8 bonus true damage on that hit.` |
| M005 | Echo Strike | Rare | damage, repeat | `on_match` (after single-target attack commit) | Multiplicative repeat scalar (`0.5x`) on copied damage instance; repeats of repeats are disallowed in v1 | `After using a single-target attack card, repeat that card's damage effect at 50% rounded down on the same target once.` |
| M006 | Adaptive Guard | Rare | defense, scaling | `on_match` (when shield is gained), `on_score_commit` (persist stack) | Additive per-stack bonus to shield gain; persistent run-scoped stack increments additively | `Each time shield is gained, increase that shield gain by +2. This bonus stacks additively and persists for the run.` |
| M007 | Focused Draw | Rare | draw, resource | `on_turn_start` | Additive draw-to-threshold effect; does not multiply draw count | `At turn start, if hand size < 3, draw until hand size = 3.` |
| M008 | Frugal Burst | Rare | cost, burst | `on_turn_start` (flag), `on_match` (first attack spend) | Additive cost reduction (`-1`) with minimum clamp at `0`; clamp occurs after additive/multiplicative cost mods | `The first attack card played each turn costs 1 less energy (minimum cost 0).` |
| M009 | Crescendo Core | Epic | scaling, damage | `on_match` (every 6 cards played), `on_score_commit` (persist for encounter) | Additive permanent attack increments within encounter; cumulative additive stacks | `After every 6 cards played in a single encounter, gain +1 permanent attack for the remainder of that encounter.` |
| M010 | Lasting Pulse | Epic | sustain, recovery | `on_turn_end` (every 3rd turn), `on_score_commit` | Additive heal event with max-HP clamp applied in clamp tier | `At end of every 3rd turn, restore 5 HP. This cannot raise HP above max HP.` |

## 6.1 Explicit contradiction pairs in starter pool
- M001 **Sharp Opening** contradicts any motif with effect clause `At encounter start, set base attack to 0 for N turns.` (none in starter pool).
- M008 **Frugal Burst** contradicts any motif with effect clause `The first attack card played each turn costs 1 more energy.` (none in starter pool).

In v1 starter pool, no pair among M001-M010 is mutually contradictory.
