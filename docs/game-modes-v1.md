# Game Modes v1

## Purpose
This document defines the player-facing identity, objective hierarchy, and validation criteria for the initial game mode set. It is intended to align design, implementation, UI copy, and QA.

---

## Shared Core Rules (All Modes Unless Overridden)

These rules define the baseline game contract.

### Core Loop
- Players place/resolve pieces on the board according to normal placement and resolution rules.
- Successful actions can generate **resonance** and progress **motif** state.
- The board can enter constrained states (low space, no legal actions, etc.) depending on mode.

### Shared Systems
- **Resonance**
  - Resonance is a core resource/state system available in all modes by default.
  - Resonance gains/spends follow the standard economy unless a mode override disables or simplifies it.
- **Motifs**
  - Motif discovery/progression remains part of normal play by default.
  - Motif effects and tracking follow base rules unless mode overrides replace or suppress them.
- **Scoring**
  - Score is always computed.
  - Whether score is a win condition or only an informational metric depends on mode.

### Shared Failure-State Vocabulary
Use these consistent terms in code, UI, and QA:
- **Deadlock**: no legal moves/actions remain.
- **Board Failure**: board reaches an unrecoverable state defined by core rules (e.g., overflow/blocked condition).
- **Run End**: session termination condition (can be failure-based or player-initiated depending on mode).

> Note: each mode below specifies whether these states end the run, reduce reward, or are ignored.

---

## Mode Definitions

## 1) Standard Mode

### Identity
The default, balanced mode focused on board mastery and completion quality.

### Objective Priority
1. **Primary:** Complete the board objective.
2. **Secondary:** Maximize score within successful board completion.

### Success Criteria
- A run is a **success** only when the board-completion objective is achieved.
- Score determines post-run ranking/reward granularity among successful runs.

### Failure Criteria
- Deadlock or board failure before board-completion objective is met results in run failure.

### System Behavior
- Resonance: full baseline behavior.
- Motifs: full baseline behavior.
- Roguelike layer: fully enabled.

### UI Objective Copy
**Your goal is to complete the board objective; score is a secondary measure of performance.**

---

## 2) Score Attack Mode

### Identity
High-intensity scoring mode where optimization and endurance drive outcomes.

### Objective Priority
1. **Primary:** Reach the highest possible score.
2. **Constraint:** Survive and avoid deadlock/board-failure states for as long as possible.

### Success Criteria
- A run is evaluated primarily by final score.
- Longer survival is expected because deadlock/board failure ends scoring opportunity.

### Failure Criteria
- Deadlock or board failure immediately ends the run.
- Ended runs are still valid attempts; ranking is by score.

### System Behavior
- Resonance: full baseline behavior (to preserve optimization depth).
- Motifs: full baseline behavior (high-skill scoring interactions remain available).
- Roguelike layer: enabled unless separately tuned for pacing.

### UI Objective Copy
**Your goal is to earn the highest score possible before you are forced into deadlock or failure.**

---

## 3) Zen Mode

### Identity
Low-pressure, player-paced mode emphasizing expression and flow over challenge.

### Objective Priority
1. **Primary:** Continue play freely at your own pace.
2. **Secondary:** Optional score tracking for personal interest only.

### Success Criteria
- No strict win requirement.
- Session is considered complete when the player chooses to end.

### Failure Criteria
- No punitive fail state by default.
- Deadlock and board failure should be prevented, auto-resolved, or treated non-punitively according to implementation.

### System Behavior
- Resonance: simplified or disabled (implementation choice must avoid cognitive pressure).
- Motifs: simplified or cosmetic-only tracking.
- Roguelike layer: removed or heavily simplified (no escalating punishment loop).

### UI Objective Copy
**Your goal is to play freely and enjoy the board—there is no fail pressure in Zen Mode.**

---

## Shared Rules vs Mode Overrides

| System / Rule | Shared Baseline | Standard Mode | Score Attack Mode | Zen Mode |
|---|---|---|---|---|
| Board objective exists | Yes | Required for success | Optional/ignored for ranking | Optional/ambient |
| Score computed | Yes | Secondary metric | Primary metric | Informational only |
| Resonance | Enabled by default | Full | Full | Simplified or off |
| Motifs | Enabled by default | Full | Full | Simplified/cosmetic |
| Deadlock handling | Defined globally | Immediate run failure if objective incomplete | Immediate run end; submit score | Non-punitive (prevent/resolve/ignore) |
| Board failure handling | Defined globally | Run failure | Run end; submit score | Non-punitive (prevent/resolve/ignore) |
| Roguelike escalation | Baseline available | Enabled | Enabled (pace-tuned allowed) | Disabled or heavily reduced |
| Run completion trigger | Mode-defined | Board objective achieved (success) | Failure/end-state reached (score submitted) | Player exits session |

---

## QA Validation Matrix (v1)

Use this matrix for fast mode-regression checks.

| Test Case | Standard Expected | Score Attack Expected | Zen Expected |
|---|---|---|---|
| UI objective line shown on mode start | Matches Standard copy | Matches Score Attack copy | Matches Zen copy |
| Board completed with moderate score | **Success** | Continue or end per rules; score is main output | Allowed; no special success gate required |
| High score but board not "completed" | Not a completion success | Valid result; ranked by score | Valid session; informational only |
| Deadlock occurs | Run fails | Run ends; score finalized | No punitive fail (recover/continue/end gracefully) |
| Board failure occurs | Run fails | Run ends; score finalized | No punitive fail (recover/continue/end gracefully) |
| Resonance interactions available | Yes | Yes | Simplified or unavailable |
| Motif progression impacts gameplay | Yes | Yes | Simplified or cosmetic-only |
| Roguelike pressure/escalation present | Yes | Yes (possibly tuned) | No / significantly reduced |
| Manual player exit | Ends run (typically abandon if objective unmet) | Ends run and submits current score | Primary intended way to end session |

---

## Canonical UI Objective Lines

For consistency across menus, loading cards, and in-run HUD:

- **Standard Mode:** “Your goal is to complete the board objective; score is a secondary measure of performance.”
- **Score Attack Mode:** “Your goal is to earn the highest score possible before you are forced into deadlock or failure.”
- **Zen Mode:** “Your goal is to play freely and enjoy the board—there is no fail pressure in Zen Mode.”

---

## Versioning
- Version: **v1**
- Intent: establish stable mode identity and QA test surface before numerical balancing.
- Any future changes to objective hierarchy or fail-state handling should increment document version and include migration notes.
