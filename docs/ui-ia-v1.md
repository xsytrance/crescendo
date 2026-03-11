# UI IA v1 — HUD, Tooltips, Onboarding, and Accessibility

## Purpose and scope
This document defines the information architecture (IA) and interaction rules for the in-game HUD and mechanic tooltips. It is intended to align design, implementation, and QA for a consistent first-release player experience.

---

## 1) HUD elements and hierarchy

### 1.1 Priority model
Use this hierarchy to prioritize visual prominence, placement, and update cadence:

1. **Primary (always visible, high salience)**
   - Score
   - Multiplier
   - Resonance meter
2. **Secondary (always visible, medium salience)**
   - Crescendo readiness
3. **Tertiary (contextual, compact by default)**
   - Motif list

### 1.2 Recommended layout zones
- **Top-left:** Score (largest numeric anchor)
- **Top-center or top-right:** Multiplier (paired with score but visually distinct)
- **Bottom-center:** Resonance meter (continuous fill + threshold markers)
- **Adjacent to resonance meter:** Crescendo readiness state chip/button
- **Right side panel or collapsible lower-right block:** Motif list

> Note: Exact coordinates can vary by aspect ratio; relative ordering and visibility priority must stay consistent.

### 1.3 Element definitions and acceptance criteria

#### A) Score
**Definition:** Running points total earned by player actions.

**Behavior:**
- Updates immediately when scoring events resolve.
- Uses an animated delta pulse (+X) near the score value for positive gains.

**Acceptance criteria:**
- Score is visible at all gameplay times (except explicitly full-screen modal overlays).
- Numeric value updates within **100 ms** of event resolution.
- Delta pulse appears for gains and fades within **600–1000 ms**.
- At no time may score text clip or overflow container at supported resolutions.

#### B) Multiplier
**Definition:** Current score multiplier applied to eligible scoring actions.

**Behavior:**
- Displayed as `xN` (e.g., `x1`, `x2.5`, `x4`).
- Brief highlight on increase; warning tint/animation on imminent decay.

**Acceptance criteria:**
- Multiplier is always visible during gameplay.
- Format remains `xN` with max 1 decimal place unless value is integer.
- Increase feedback appears within **100 ms** of change.
- Decay warning appears at configured threshold (default: final 20% of decay timer).
- Warning state must remain legible and distinct under reduced-motion mode.

#### C) Resonance meter
**Definition:** Resource/progress meter that fills from gameplay actions and enables crescendo.

**Behavior:**
- Horizontal or radial meter with current fill and one or more threshold markers.
- Fill increases/decreases smoothly with bounded animation.

**Acceptance criteria:**
- Meter is always visible during gameplay.
- Fill percentage reflects internal state with max visual error of ±1%.
- Threshold markers are visible at all scales where HUD is shown.
- Fill transition animation duration is **100–250 ms** per update (or instant in reduced motion).

#### D) Crescendo readiness
**Definition:** Explicit state indicator showing whether crescendo can be activated.

**States:**
- `Charging` (not ready)
- `Ready` (activation available)
- `Active` (currently in crescendo)
- `Cooldown` (temporarily unavailable)

**Behavior:**
- Shown as badge/chip/button adjacent to resonance meter.
- State label and icon change by state.

**Acceptance criteria:**
- Exactly one state is displayed at any given moment.
- State transition appears within **100 ms** of state changes.
- `Ready` state includes non-color cue (icon or text emphasis), not color alone.
- If activatable by input, control includes keybinding hint on desktop and touch affordance on mobile.

#### E) Motif list
**Definition:** Active or recently collected motif set relevant to strategic play.

**Behavior:**
- Compact by default (e.g., top 3–5 items), expandable for full list.
- Items include icon + label + optional progress or stack indicator.

**Acceptance criteria:**
- List is available from gameplay HUD without navigating away.
- Default compact view shows at least 3 motifs on 16:9 at baseline HUD scale.
- Expanded state persists only for current session unless configured otherwise.
- Item labels truncate gracefully with tooltip or expand-on-focus behavior.

---

## 2) Tooltip trigger, dismissal, and mobile fallback

### 2.1 Trigger behavior (desktop)
- **Hover trigger:**
  - On pointer hover over a tooltip-enabled element for **300 ms dwell**, show tooltip.
  - Cancel if pointer leaves target before dwell completes.
- **Click trigger (sticky mode):**
  - Click/tap on tooltip-enabled info icon toggles pinned tooltip.
  - Pinned tooltip remains visible until dismissed.

### 2.2 Dismissal behavior
- Non-pinned hover tooltip dismisses on:
  - Pointer leave from target + tooltip hit area for **150 ms**, or
  - `Esc` key.
- Pinned tooltip dismisses on:
  - Close button,
  - `Esc` key,
  - Clicking/tapping outside,
  - Re-clicking source icon.

### 2.3 Mobile fallback
- Hover is unavailable; use tap-only interaction:
  - First tap opens tooltip as a bottom sheet or anchored popover.
  - Second tap on source or close action dismisses.
- Ensure tooltip content is scrollable if content exceeds available height.

### 2.4 Tooltip acceptance criteria
- Tooltip never renders off-screen; auto-flips or repositions.
- Tooltip appearance latency:
  - Hover mode: 300 ms dwell ±50 ms.
  - Tap/click mode: visible within 100 ms of input.
- All tooltips support keyboard focus and dismissal with `Esc`.
- On mobile, tooltip open/close is reachable with one-handed tap targets (minimum 44x44 px).
- At most one pinned tooltip may be open at a time.

---

## 3) First-run teach moments (first 3 minutes)

### 3.1 Progressive disclosure principles
- Teach only mechanics that are immediately actionable.
- Use short, plain-language prompts (1 sentence + optional “Why it matters”).
- Do not block gameplay unless the action is required to proceed.
- Never show more than one coachmark at once.

### 3.2 Timeline sequence

#### 0:00–0:20 — Orientation
- Highlight Score + Multiplier.
- Message: “Score goes up when you land actions; multiplier boosts each gain.”
- Trigger: first successful scoring event.

#### 0:20–1:00 — Resource awareness
- Highlight Resonance meter.
- Message: “Your resonance builds from consistent play.”
- Trigger: meter reaches first visible threshold (e.g., 25%).

#### 1:00–1:45 — Crescendo readiness
- Highlight Crescendo readiness indicator when entering `Ready` for first time.
- Message: “Crescendo is ready—activate now for a power window.”
- Trigger: first transition to `Ready`.

#### 1:45–2:30 — Motif strategy
- Highlight Motif list area.
- Message: “Motifs track your current patterns—combine them for better outcomes.”
- Trigger: first motif acquisition or second motif interaction.

#### 2:30–3:00 — Tooltip literacy
- Prompt on info icon behavior.
- Message: “Need a refresher? Hover or tap info icons any time.”
- Trigger: no tooltip opened yet by 2:30.

### 3.3 Teach moments acceptance criteria
- Each teach moment appears at most once per account/profile unless reset.
- If player performs the taught action before prompt, skip or auto-complete that moment.
- Prompt text max length: 120 characters on default locale baseline.
- Prompts never overlap critical gameplay alerts.
- Completion of all first-run teach moments is logged for analytics/QA verification.

---

## 4) Terminology glossary (plain language)

| Term | Plain-language definition |
|---|---|
| Score | Your total points in the current run/session. |
| Multiplier | A bonus factor that makes each point event worth more. |
| Resonance | A build-up resource earned by consistent successful play. |
| Resonance meter | The visual bar/gauge showing how much resonance you currently have. |
| Crescendo | A temporary high-power state you can trigger when ready. |
| Crescendo readiness | The current availability state of crescendo (charging/ready/active/cooldown). |
| Motif | A gameplay pattern, trait, or token that influences strategy and scoring. |
| Motif list | The HUD panel showing currently relevant motifs and their status. |
| Tooltip | Small contextual help text shown near an element. |
| Coachmark / Teach moment | A short onboarding prompt that explains a feature in context. |
| Cooldown | A wait period before a mechanic can be used again. |
| Threshold | A marked point on a meter that signals a new state or benefit. |
| Encounter | In v1, this means one continuous board session from seed/reset until run end or next reset event (not a separate combat room). |

### Glossary acceptance criteria
- Every core HUD mechanic has one glossary entry.
- Definitions are readable at approximately grade 8 level or simpler.
- No glossary entry uses an undefined term unless linked to another glossary term.
- Glossary is accessible from help/settings and referenced by tooltip system.

---

## 5) Accessibility baselines

### 5.1 Color and contrast
- Minimum contrast ratios:
  - Body text/UI labels: **4.5:1**
  - Large text (>=18 pt or >=14 pt bold): **3:1**
  - Essential icons and state indicators: **3:1** against background

**Acceptance criteria:**
- HUD labels and values meet or exceed required contrast in all supported themes.
- Critical state changes are never conveyed by color alone.

### 5.2 Colorblind-safe suit indicators
- Suit/category indicators must combine:
  - Distinct hue,
  - Distinct shape/icon,
  - Optional pattern/texture cue.

**Acceptance criteria:**
- Any two suit indicators remain distinguishable in simulated protanopia, deuteranopia, and tritanopia.
- Indicator meaning is preserved in grayscale.

### 5.3 Motion and animation reduction
- Provide **Reduce Motion** toggle in Accessibility settings.
- In reduced mode:
  - Replace non-essential motion with fades or instant state changes.
  - Keep timing and feedback clarity via text/icon emphasis.

**Acceptance criteria:**
- All HUD feedback remains functional with reduced motion enabled.
- No essential gameplay information depends on animation presence.
- Toggle applies live without requiring restart.

### 5.4 Input and focus
- Keyboard navigability for tooltip triggers, pinned tooltip controls, and help links.
- Visible focus indicators for all interactive HUD/help elements.

**Acceptance criteria:**
- Full tooltip workflow (open/read/dismiss) is keyboard-operable.
- Focus order follows visual reading order and has no traps.

---

## QA checklist summary (cross-cutting)
- HUD element ordering matches hierarchy specification.
- State changes propagate to UI within stated latency bounds.
- Tooltip behavior matches desktop and mobile definitions.
- First-run sequence respects timing, triggers, and skip logic.
- Accessibility baselines pass contrast, non-color cues, reduced-motion, and keyboard checks.

