# Tutorial Level v1 — `tutorial_01: First Resonance`

This file defines an implementation-ready tutorial level that is consistent with the current v1 docs and is immediately playable as a deterministic scripted board.

## Design goals

- Teach the canonical turn loop (`move`, `activate_crescendo`, `draw_stock`) in a low-variance board.  
- Demonstrate multiplier rise/fall and resonance gain/loss using the exact v1 rules.  
- Force one clear Crescendo earn-and-spend sequence so players understand that Crescendo is charge-based and player-activated.  
- Introduce motif awareness and tooltip literacy inside the first ten turns, aligned with first-run teach moments.

## Assets

- Level data: `levels/tutorial_level_01.json`
- Recommended HUD coachmark copy is embedded per scripted turn.

## Why this is playable now

- The board is fully specified (`rows`, `cols`, card identities, and initial face states).
- Initial run state matches canonical state variables from `core-loop-v1`.
- The turn script enumerates legal player inputs and expected state deltas after each turn.
- Completion and failure conditions are explicit.

## Integration notes

1. **Deterministic mode toggle**: lock the board seed and enforce `scripted_turns.required_action` only for this tutorial level.
2. **Validation mode**: after each turn, compare runtime state against `scripted_turns.expected` and log any mismatch.
3. **Coachmarks**: render one prompt per turn and auto-dismiss on successful action.
4. **Motif tutorial override**: tutorial grants one controlled motif offer on turn 7, even though baseline motif grants are encounter/score based.

## QA checklist for this level

- Turn 6 creates exactly one `crescendo_charge_gained` event.
- Turn 7 consumes exactly one charge via `activate_crescendo`.
- No `draw_stock` action is legal before deadlock.
- Board clears by turn 10 with score above `minimum_score`.
- Objective copy in Standard mode remains unchanged while tutorial prompts are active.
