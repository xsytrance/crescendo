# Scoring Model v1

This document defines a concrete, tunable scoring system for a run-based solitaire variant with combo multipliers and a "crescendo" spike state.

---

## 1) Base Score Per Move Type

Let each scored event be assigned a base value `B(event)` before multipliers, bonuses, and safeguards.

| Event type | Symbol | Base score `B` | Notes |
|---|---:|---:|---|
| Valid tableau move (opens tactical space, no reveal) | `T_move` | `8` | Core low-value repeat action |
| Tableau move that reveals a facedown card | `T_reveal` | `14` | Includes value for information unlock |
| Flip reveal from stock/waste cycle | `Flip` | `5` | Keeps flow rewarded but low farming value |
| Move card to foundation | `F_move` | `20` | Primary progress driver |
| Move sequence to tableau (length >= 3) | `Seq_move` | `12 + 2*(len-3)` | Capped at `len=7` for scoring |
| Rescue move (frees blocked key rank: A,2,K) | `Rescue` | `16` | Optional classifier from rules engine |
| Undo / reverse-equivalent action | `Undo_like` | `0` | Never directly scores |

### Composite event handling

If one action satisfies multiple categories, use:

\[
B = \max(B_i) + 0.35 \cdot \sum_{j \neq i} B_j
\]

where `i` is the highest base component. This avoids double-count abuse while preserving hybrid value.

---

## 2) Multiplier Growth Function

Use a **piecewise linear + step + capped exponential tail** model for smooth early growth and exciting spikes.

Let:
- `c` = current combo chain length (consecutive scoring actions without reset)
- `M(c)` = score multiplier

\[
M(c)=
\begin{cases}
1 + 0.12(c-1), & 1 \le c \le 6 \\
1.60 + 0.08(c-6), & 7 \le c \le 15 \\
2.32 + 0.18\left(1 - e^{-0.22(c-15)}\right), & c \ge 16
\end{cases}
\]

Hard cap:

\[
M(c) \le M_{\max}=2.50
\]

### Step breakpoints (feel accents)
Apply additive bumps:

\[
M \leftarrow M + 0.05\cdot \mathbf{1}_{c\in\{5,10,15\}}
\]

Final multiplier cap after bumps remains `2.50`.

---

## 3) Reset Rules

Two stateful systems are tracked independently:
- `combo` (drives `M(c)`)
- `resonance` (drives crescendo readiness; defined below)

### 3.1 Combo reset rules

Combo **continues** on any scored forward-progress action: `T_move`, `T_reveal`, `F_move`, `Seq_move`, `Rescue`.

Combo **resets to 1** on:
1. Dead action (no board delta) or invalid move attempt.
2. Undo-like reversal that recreates a prior board hash within last `N=8` turns.
3. `stock_draw`, `hard_reset`, or `redeal`.
4. Timeout between scored actions greater than `t_idle = 12s` (if real-time mode enabled).

### 3.2 Resonance reset/decay rules

Let resonance be `R in [0, 100]`.

Per scored action gain:
\[
\Delta R = g(event) \cdot (1 + 0.15(M-1))
\]

with defaults:
- `g(T_move)=2`
- `g(T_reveal)=4`
- `g(Flip)=1`
- `g(F_move)=6`
- `g(Seq_move)=5`
- `g(Rescue)=5`

Decay/reset:
- Passive decay each non-scoring turn: `R <- max(0, R-7)`
- `stock_draw`: apply normal non-scoring decay/loss only (no forced hard reset)
- `hard_reset`: set `R <- 0`
- `redeal`: no mandatory reset (mode-controlled; Zen keeps resonance valid)
- Soft setback on loop suspicion: `R <- 0.65R`

---

### 3.3 Shared reset-event mapping

Use the canonical event names from `core-loop-v1.md` to avoid terminology drift:

| Event | Combo effect | Resonance effect | Run state effect |
|---|---|---|---|
| `stock_draw` | Reset to `1` | No forced reset; apply stock loss/decay rules | Continue run |
| `hard_reset` | Reset to `1` | Set to `0` | Continue in endless/recovery contexts; otherwise may precede `run_fail` |
| `redeal` | Reset to `1` | Mode-defined (non-punitive modes keep resonance) | Continue run |
| `run_fail` | N/A | N/A | End run (failure) |
| `run_success` | N/A | N/A | End run (success) |

## 4) Crescendo Bonus Formula

Use a **temporary state window** + additive multiplier bonus.

### Trigger
Crescendo activates when:
\[
R \ge 100 \land c \ge 8
\]

On trigger:
- Set `state = Crescendo`
- Duration `d = 6` scored actions
- Spend resonance: `R <- 25`

### Effect
For actions during crescendo window:
\[
Score_{event} = B(event) \cdot M(c) \cdot \left(1 + C_b\right) + C_f
\]

Defaults:
- `C_b = 0.30` (30% temporary multiplier add)
- `C_f = 6` flat per scored action

### Finisher payout
If window ends naturally with at least `3` foundation moves during crescendo:
\[
Bonus_{finish} = 40 + 12 \cdot \min(4, F_{cres})
\]
where `F_cres` = foundation moves made during crescendo.

---

## 5) Anti-Infinite Safeguards

Layer multiple protections to suppress farming loops without punishing natural play.

### 5.1 Local repetition diminishing returns

Track canonical move signature `s` over rolling window `W=20` actions.
If signature count in window is `k>=1`, apply:
\[
D_{rep}(k)=\frac{1}{1+0.22(k-1)}
\]

### 5.2 Board-state loop detection

Hash board state after every turn. If current hash reappears with period `p <= 6` and at least `3` cycles observed:
- zero score for repeated loop actions until state diverges,
- force combo reset,
- apply resonance setback (`R <- 0.65R`).

### 5.3 Soft cap on long-run efficiency

Let total raw score this run be `S_raw`. Effective score gain multiplier soft-cap:
\[
D_{soft}(S_{raw}) =
\begin{cases}
1, & S_{raw} \le 2500 \\
\left(\frac{2500}{S_{raw}}\right)^{0.18}, & S_{raw} > 2500
\end{cases}
\]

Final per-event score:
\[
Score_{final} = \left(B \cdot M \cdot Crescendo\_mods + Finisher\_if\_any\right)\cdot D_{rep}\cdot D_{soft}
\]

Round to nearest integer at commit to UI.

---

## 6) Target Pacing Bands

These bands are balancing targets, not strict guarantees.

### Early game (first 20 scored actions)
- Target cumulative score: **220–420**
- Typical multiplier range: **1.0–1.5**
- Crescendo probability: **<15%**

### Mid-run spike (actions 21–60)
- Target cumulative score: **900–1700** by action 60
- Multiplier range: **1.4–2.2**
- 1–2 crescendos expected on strong lines
- Most of skill expression should live here

### Late-run ceiling (61+)
- Target end-run totals:
  - Average clear/fail: **1800–2600**
  - Strong run: **2600–3600**
  - High-skill peak: **3600–4800**
- Soft cap should make gains beyond ~4800 asymptotically harder.

---

## Sample Expected Score Curves (for tuning anchors)

Values below are expected cumulative score by scored action index.

### A) Average run (few long chains, 0–1 crescendo)

| Action # | 10 | 20 | 30 | 40 | 50 | 60 | 75 |
|---:|---:|---:|---:|---:|---:|---:|---:|
| Cumulative score | 110 | 290 | 520 | 760 | 1020 | 1290 | 1680 |

Shape: mostly linear with mild mid-run lift.

### B) Strong run (stable combos, 1–2 crescendos)

| Action # | 10 | 20 | 30 | 40 | 50 | 60 | 75 |
|---:|---:|---:|---:|---:|---:|---:|---:|
| Cumulative score | 140 | 360 | 690 | 1140 | 1560 | 2020 | 2780 |

Shape: visible convexity beginning around action 25 due to sustained multiplier + crescendo windows.

### C) High-skill run (tight optimization, 2–3 crescendos, low repetition penalties)

| Action # | 10 | 20 | 30 | 40 | 50 | 60 | 75 |
|---:|---:|---:|---:|---:|---:|---:|---:|
| Cumulative score | 165 | 430 | 860 | 1460 | 2160 | 2990 | 4180 |

Shape: strong mid-run acceleration, then tempered late by repetition and soft-cap damping.

---

## Default Parameter Block (implementation-ready)

```yaml
scoring_v1:
  base:
    T_move: 8
    T_reveal: 14
    Flip: 5
    F_move: 20
    Seq_move_base: 12
    Seq_move_per_extra: 2
    Seq_move_len_cap: 7
    Rescue: 16
  multiplier:
    linear_1_slope: 0.12      # c=1..6
    linear_2_slope: 0.08      # c=7..15
    exp_tail_amp: 0.18        # c>=16
    exp_tail_rate: 0.22
    step_bonuses:
      5: 0.05
      10: 0.05
      15: 0.05
    hard_cap: 2.50
  combo:
    idle_reset_seconds: 12
    loop_hash_window: 8
  resonance:
    max: 100
    gains:
      T_move: 2
      T_reveal: 4
      Flip: 1
      F_move: 6
      Seq_move: 5
      Rescue: 5
    multiplier_coupling: 0.15
    passive_decay_non_scoring_turn: 7
    loop_setback_factor: 0.65
  crescendo:
    trigger_resonance: 100
    trigger_combo_min: 8
    duration_scored_actions: 6
    post_trigger_resonance: 25
    bonus_multiplier_add: 0.30
    bonus_flat_per_action: 6
    finisher_base: 40
    finisher_per_foundation: 12
    finisher_foundation_cap: 4
    finisher_min_foundation_for_award: 3
  anti_infinite:
    repetition_window_actions: 20
    repetition_penalty_alpha: 0.22
    loop_period_max: 6
    loop_cycles_required: 3
    soft_cap_start_score: 2500
    soft_cap_exponent: 0.18
```

This baseline is intentionally conservative in the first 20 actions, explosive in the mid-run, and self-damping at the high end for leaderboard health.
