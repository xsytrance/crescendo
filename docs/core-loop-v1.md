# Core Loop v1 (Deterministic Spec)

> **Source of truth note:** `docs/core-loop-v1.md` Sections **3.3** and **4** define the canonical Crescendo v1 model. Crescendo is charge-based and player-activated via `crescendo_charges`; any other doc must reuse these same state variables and transition events.

This document defines a deterministic gameplay loop using explicit state variables so design and engineering can share one source of truth.

**Terminology note:** v1 uses continuous board sessions. References to an "encounter" in adjacent docs map to one board lifecycle between seed/reset events.

## 1) Canonical State Variables

All variables below are part of the authoritative run state.

- `board`: 2D grid of face-up/face-down cards (or tiles), each with immutable `card_id` and mutable `face_state`.
- `turn_index`: Integer starting at `1`, incremented once per completed player turn.
- `score`: Integer total score for the run.
- `multiplier`: Integer score multiplier, minimum `1`.
- `resonance`: Integer meter in range `[0, resonance_cap]`.
- `resonance_cap`: Integer, default `100`.
- `crescendo_threshold`: Integer resonance needed to earn one charge, default `100`.
- `crescendo_charges`: Integer count of stored crescendo charges.
- `max_crescendo_charges`: Integer cap on stored charges, default `3`.
- `stock_count`: Integer remaining stock draws this run.
- `max_stock_count`: Integer initial stock draws, default `3`.
- `endless_mode_enabled`: Boolean toggle; default `false`.
- `last_action_result`: Struct storing resolved result of the latest player action, with fields:
  - `match_count`: Number of matched sets created by the action.
  - `base_points_gained`: Points before multiplier.
  - `resonance_delta`: Net resonance change from this action.
  - `used_stock_draw`: Boolean.
  - `consumed_crescendo`: Boolean.

## 2) Turn Phase Order

Each turn resolves in the exact sequence below:

1. **Player Input Phase**
   - Player chooses exactly one legal action:
     - `action = move` (normal board move), or
     - `action = activate_crescendo` (if `crescendo_charges > 0`), or
     - `action = draw_stock` (if `stock_count > 0` and no legal board move exists).
2. **Player Move Resolution Phase**
   - Apply the chosen action to `board`.
   - Compute immediate matches from the action and remove resolved cards.
3. **Auto-Flip Phase**
   - Any temporary face-up cards created this turn that are not part of a resolved match are flipped back face-down.
4. **Resonance Update Phase**
   - Apply deterministic resonance gain/loss rules (Section 3).
5. **Crescendo Threshold Phase**
   - Convert resonance overflow into charges (Section 3.3).
6. **Score Update Phase**
   - Add `base_points_gained * multiplier` to `score`.
7. **End-of-Turn Checks Phase**
   - Recompute legal moves.
   - Check board clear, deadlock, stock availability, and run termination rules (Section 5).
   - Increment `turn_index` by `1` if run continues.

No phase may be skipped. If a phase has no effect, it still executes as a no-op.

## 3) Deterministic Trigger Rules

## 3.1 Resonance Gain

On each turn, compute `resonance_gain` as:

- `+20` for each resolved match set this turn (`last_action_result.match_count`).
- `+10` bonus if the action caused a chain/cascade of at least 2 sequential match events in the same turn.
- `+0` otherwise.

Formula:

`resonance_gain = (20 * match_count) + chain_bonus`

Where:

- `chain_bonus = 10` if `sequential_match_events >= 2`, else `0`.

## 3.2 Resonance Loss

On each turn, compute `resonance_loss` as:

- `+15` if no match was resolved (`match_count == 0`) and action was `move`.
- `+25` if action was `draw_stock`.
- `+0` if action was `activate_crescendo` and it generated at least one match.
- `+10` if action was `activate_crescendo` and generated zero matches.

Then clamp:

`resonance = clamp(resonance + resonance_gain - resonance_loss, 0, resonance_cap)`

## 3.3 Crescendo Charge Creation Trigger

After resonance is updated, perform threshold conversion in a loop. This emits event `crescendo_charge_gained` for each successful conversion:

```
while resonance >= crescendo_threshold and crescendo_charges < max_crescendo_charges:
    resonance -= crescendo_threshold
    crescendo_charges += 1
```

If `crescendo_charges == max_crescendo_charges`, resonance may still accumulate up to `resonance_cap`, but no additional charges are created.

## 4) Crescendo Usage Rules

- Crescendo is **player-activated**, not automatic.
- A crescendo may be consumed only during **Player Input Phase** by choosing `action = activate_crescendo`.
- There is **no timed auto-trigger window** and **no cooldown state** in v1.
- Consumption rule:
  - If chosen while `crescendo_charges > 0`, decrement `crescendo_charges` by `1` immediately.
  - Emit transition event `crescendo_activated` when the decrement occurs.
  - Apply the crescendo effect to the current turn's move resolution (implementation-defined effect payload, e.g., guaranteed wild match, board pulse, etc.).
- Failed activation guard:
  - If no valid crescendo effect target exists, action is illegal and cannot be selected.

Backend-readable readiness mapping for v1:

- `Charging`: `crescendo_charges == 0`
- `Ready`: `crescendo_charges > 0`
- `ActivatedThisTurn` (transient): `last_action_result.consumed_crescendo == true`

## 5) Stock Draw and Hard Reset Rules

## 5.1 Stock Draw Eligibility

`draw_stock` is legal only when:

- `stock_count > 0`, and
- current legal move count is `0` (board deadlock state).

When executed:

- decrement `stock_count` by `1`,
- refill or inject cards according to stock implementation,
- apply resonance loss from Section 3.2.

## 5.2 Hard Reset Behavior

When `stock_count` reaches `0` and a deadlock occurs again, perform hard reset behavior immediately:

- `multiplier = 1`
- `resonance = 0`
- `crescendo_charges = 0`
- board is re-seeded to a fresh solvable layout.

This v1 spec defines hard reset as resetting **both** momentum systems (`multiplier` and `resonance`, including `crescendo_charges`).

Optional future flag (not active in v1):

- `hard_reset_mode in {"both", "multiplier_only", "resonance_only"}`

For v1, enforce `hard_reset_mode = "both"`.

## 5.3 Shared Reset Taxonomy (Canonical Event Names)

To keep core-loop, mode, and scoring specs aligned, all reset or recovery flows must use the
same event names.

- `stock_draw`: Player spends one stock draw to reseed/refill while deadlocked.
- `hard_reset`: Full momentum reset (`multiplier`, `resonance`, `crescendo_charges`) plus fresh board reseed.
- `redeal`: Board-only reseed that does **not** reset momentum systems. Use only in modes that allow non-punitive recovery (e.g., Zen).
- `run_fail`: Run terminates as failure.
- `run_success`: Run terminates as success.

If an implementation does not support board-only reseeds, map all redeal behavior to `hard_reset`
and do not emit `redeal`.

## 5.4 Cross-Doc Event Effects Table

| Event | Combo effect | Resonance effect | Run state effect |
|---|---|---|---|
| `stock_draw` | Reset combo to `1` | Apply stock-draw loss from Section 3.2 (`-25`, clamped) | Run continues |
| `hard_reset` | Reset combo to `1` (by resetting multiplier context) | Set to `0`; set `crescendo_charges = 0` | Run continues in endless contexts; otherwise typically followed by `run_fail` |
| `redeal` | Reset combo to `1` | No mandatory reset; mode-specific adjustment allowed | Run continues |
| `run_fail` | N/A (run ends) | N/A (run ends) | Run ends as failure |
| `run_success` | N/A (run ends) | N/A (run ends) | Run ends as success |

## 6) End-of-Run Conditions

At End-of-Turn Checks, evaluate in this order:

1. **Board Clear Win**
   - If all objective cards are cleared: run ends with success.
2. **Deadlock With No Stock**
   - If no legal moves and `stock_count == 0`:
     - If `endless_mode_enabled == false`: run ends with failure.
     - If `endless_mode_enabled == true`: apply hard reset and continue run.
3. **Otherwise Continue**
   - Proceed to next turn.

## 7) Scoring Rule (v1 Normative Multiplier Model)

Per turn:

- `turn_points = base_points_gained * multiplier_start_of_turn`
- `score += turn_points`

v1 uses the **simple integer step multiplier** from the core loop (not the combo-function model):

- If `match_count > 0`, `multiplier_next_turn = multiplier_current_turn + 1`.
- If `match_count == 0`, `multiplier_next_turn = max(1, multiplier_current_turn - 1)`.

This is equivalent to:

`multiplier_next_turn = max(1, multiplier_current_turn + (1 if match_count > 0 else -1))`

No additional curve, breakpoint bonus, or exponential tail applies in v1. Multiplier transitions occur at end-of-turn and are visible starting on the next turn.

## 8) Worked Example (12 Turns)

Assume initial state:

- `turn_index=1`, `score=0`, `multiplier=1`, `resonance=0`, `crescendo_charges=0`, `stock_count=2`, `endless_mode_enabled=false`.
- `resonance_cap=100`, `crescendo_threshold=100`, `max_crescendo_charges=3`.

| Turn | Action | match_count | base_points_gained | resonance change | resonance (end) | crescendo_charges | multiplier (end) | score (end) | Notes |
|---|---|---:|---:|---:|---:|---:|---:|---:|---|
| 1 | `move` | 1 | 100 | +20 | 20 | 0 | 2 | 100 | basic match |
| 2 | `move` | 1 | 100 | +20 | 40 | 0 | 3 | 300 | combo building |
| 3 | `move` | 0 | 0 | -15 | 25 | 0 | 2 | 300 | miss |
| 4 | `move` | 2 | 220 | +50 (`+40` + chain `+10`) | 75 | 0 | 3 | 740 | cascade |
| 5 | `move` | 1 | 120 | +20 | 95 | 0 | 4 | 1220 | near threshold |
| 6 | `move` | 1 | 120 | +20 => 115 then charge conversion | 15 | 1 | 5 | 1700 | earned 1 crescendo |
| 7 | `activate_crescendo` | 2 | 260 | +50 | 65 | 0 | 6 | 3000 | consumed charge |
| 8 | `move` | 0 | 0 | -15 | 50 | 0 | 5 | 3000 | miss |
| 9 | `move` | 0 | 0 | -15 | 35 | 0 | 4 | 3000 | deadlock forms |
| 10 | `draw_stock` | 0 | 0 | -25 | 10 | 0 | 3 | 3000 | `stock_count` 2->1 |
| 11 | `move` | 1 | 140 | +20 | 30 | 0 | 4 | 3420 | recovered |
| 12 | `move` | 0 | 0 | -15 | 15 | 0 | 3 | 3420 | continue run |

This sequence demonstrates deterministic phase resolution, resonance threshold conversion into `crescendo_charges`, explicit player-activated crescendo use, and stock draw penalties without hard reset (since stock remains).


## 9) Mini Example: Exact Multiplier Transitions (6 Turns)

Initial state for this mini-example: `multiplier=1`, `score=0`.

| Turn | match_count | base_points_gained | multiplier at start | turn_points | multiplier at end (next turn value) | score (end) |
|---|---:|---:|---:|---:|---:|---:|
| 1 | 1 | 80 | 1 | 80 | 2 | 80 |
| 2 | 2 | 120 | 2 | 240 | 3 | 320 |
| 3 | 0 | 0 | 3 | 0 | 2 | 320 |
| 4 | 1 | 100 | 2 | 200 | 3 | 520 |
| 5 | 0 | 0 | 3 | 0 | 2 | 520 |
| 6 | 0 | 0 | 2 | 0 | 1 | 520 |

Transition trace: `1 -> 2 -> 3 -> 2 -> 3 -> 2 -> 1`. This demonstrates the exact +1/-1 step behavior with floor at `1`.
