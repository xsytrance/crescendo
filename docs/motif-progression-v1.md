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

---

## 6) Starter pool (10 motifs)

Effect text below is authoritative and implementation-oriented.

| ID | Name | Rarity | Tags | Effect (machine-testable) |
|---|---|---|---|---|
| M001 | Sharp Opening | Common | opener, multiplier | `For the first 3 turns after board seed/reset, if match_count > 0 then multiplier gains an additional +1 (net multiplier change for a matching turn becomes +2 instead of +1).` |
| M002 | Guarded Entry | Common | opener, resonance | `For turns 1-3 after board seed/reset, reduce resonance_loss from a miss by 5 (move miss: 15->10; failed activate_crescendo: 10->5).` |
| M003 | Rhythm Step | Common | tempo, chain | `On every 4th turn (turn_index % 4 == 0), if match_count >= 1 then add +10 bonus resonance_gain after normal resonance calculation.` |
| M004 | Clean Finish | Common | execute, board-control | `If a turn resolves match_count >= 2, add +40 flat base_points_gained before multiplier is applied.` |
| M005 | Echo Strike | Rare | repeat, resonance | `After any turn with match_count >= 1, grant +5 resonance per matched set (bonus_resonance = 5 * match_count).` |
| M006 | Adaptive Guard | Rare | scaling, recovery | `Each time draw_stock is used, permanently reduce that action's resonance_loss by 5 for the rest of the run (minimum draw_stock loss floor = 10).` |
| M007 | Focused Draw | Rare | deadlock, stock | `When legal_move_count becomes 0, gain one-time deadlock guard: the next draw_stock in this deadlock does not consume stock_count.` |
| M008 | Frugal Burst | Rare | deadlock, recovery | `After draw_stock resolves, if legal_move_count > 0 then grant +1 multiplier immediately.` |
| M009 | Crescendo Core | Epic | scaling, resonance | `Whenever resonance_gain is applied, also add floor(multiplier / 2) resonance (e.g., multiplier 5 grants +2 extra resonance on that turn).` |
| M010 | Lasting Pulse | Epic | sustain, deadlock | `After any deadlock recovery that restores legal_move_count from 0 to >=1, set resonance to max(current resonance, 30).` |

## 6.1 Explicit contradiction pairs in starter pool
- M001 **Sharp Opening** contradicts any motif with effect clause `For the first N turns after board seed/reset, matching turns cannot increase multiplier.` (none in starter pool).
- M008 **Frugal Burst** contradicts any motif with effect clause `After draw_stock resolves, multiplier cannot increase this turn.` (none in starter pool).

In v1 starter pool, no pair among M001-M010 is mutually contradictory.

## 6.2 Deterministic motif-to-core-loop hook examples

These examples are normative implementation hooks that compose with `core-loop-v1` phase order.

1. **Resonance gain modification (M009 + M005):**
   - Base turn result: `match_count = 2`, `multiplier = 5`, chain bonus active.
   - Core loop resonance gain: `(20 * 2) + 10 = 50`.
   - Apply M005 bonus: `+ (5 * 2) = +10`.
   - Apply M009 bonus: `+ floor(5 / 2) = +2`.
   - Final `resonance_gain = 62` before resonance loss/clamp.

2. **Stock penalty reduction and deadlock guard (M006 + M007):**
   - `legal_move_count` becomes `0`; M007 arms guard.
   - Player uses `draw_stock`: `stock_count` is unchanged (guard consumed), and M006 applies reduced loss.
   - If M006 has stacked twice already, draw penalty is `25 - (5 * 2) = 15`.

3. **Chain-to-score conversion (M004 with baseline scoring):**
   - Turn resolves with `match_count = 3`, `base_points_gained = 180`, `multiplier = 4`.
   - M004 condition passes (`match_count >= 2`) and adds `+40` base points.
   - Score phase uses modified base: `(180 + 40) * 4 = 880` turn points.
