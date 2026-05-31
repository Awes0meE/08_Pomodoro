---
date: 2026-05-31
topic: digit-particle-trigger
---

# Digit Particle Trigger & Transition — Requirements

## Summary

Make digit shatter particles predictable whenever the main timer `MM:SS` changes outside the countdown tick, especially when adjusting focus/short/long duration in Settings. Refactor the display transition path for reliability, clearer structure, and lighter DOM/layout work. Rapid stepper input may coalesce multiple intermediate values into one morph to the final time (e.g. 25 → 26 → 27 shows a single morph to `27:00`). Running countdown ticks do not trigger particles.

## Problem Frame

Users report inconsistent particle feedback when changing durations in Settings—sometimes the effect plays, sometimes it does not. Today, animation is tied to a narrow set of call paths (`updateDisplay(true)` via mode switch and reset) and to per-character diff inside `setTimerText`, with transitions that can be interrupted when inputs change quickly. That mismatch between user expectation (“I changed the time on screen”) and implementation (“only certain updates animate, and races can drop the effect”) is the core problem. A separate pass (`docs/brainstorms/2026-05-31-settings-mode-preview-requirements.md`) addresses settings preview and mode switch; this document defines when particles must fire and how transitions must behave so preview and particles stay aligned.

## Key Decisions

- **Trigger policy:** If the main display `MM:SS` string changes, particles are on, and the change is not from a one-second countdown tick, play the digit morph + particles for all changed digit positions.
- **Tick exclusion:** `updateDisplay` during the running interval does not request digit animation, even when seconds change.
- **Settings coverage:** Every successful commit of focus/short/long duration (stepper, blur, Enter) that updates the main display must use the animated path when the displayed time changes—including after auto-switch to the matching mode tab per settings-mode-preview rules.
- **No-op exclusion:** If the committed setting value is unchanged, or the user cancels a duration-reset confirmation, the main display does not change and no particle morph runs.
- **Rapid stepper coalescing:** While the user holds or rapidly taps +/−, intermediate display values may be skipped; when input pauses, one morph runs from the last stable `MM:SS` to the final `MM:SS` (e.g. only `27:00`, not separate morphs for `26:00`).
- **Implementation priority:** Reliability and clear separation of concerns first (when to animate vs how to animate), then performance (fewer redundant layout reads and DOM nodes). Visual timing polish is secondary unless needed to fix obvious desync.
- **Approach shape:** Explicit change-source classification for display updates plus a single transition scheduler that owns diff, morph class, and particle spawn—rather than ad hoc flags scattered at each call site.

## Requirements

**Trigger rules**

- R1. When `state.settings.particles` is enabled and the user has not requested reduced motion, any main-display `MM:SS` update that is not driven by the one-second running countdown must use the animated digit transition when at least one digit character differs from the previous displayed string.
- R2. Running countdown ticks must update the display without digit particles or digit-morph animation.
- R3. Successful focus, short break, or long break duration commits in Settings must trigger the animated path whenever the resulting main `MM:SS` differs from what was shown immediately before the commit (after mode switch and full-duration reset per settings-mode-preview).
- R4. Mode tab switches, timer reset, and other non-tick display updates that change `MM:SS` must follow the same animated path as R1.
- R5. If the displayed `MM:SS` is unchanged (including unchanged duration setting, canceled confirmation, or clamp-only toast with no display change), no particle morph runs.
- R6. When particles are disabled in Settings, use the existing non-particle fallback (simple blur or equivalent) for display changes that would otherwise animate—behavior unchanged except for reliability of when that branch runs.

**Rapid input coalescing**

- R7. During rapid stepper (+/−) input on a duration control, the app may defer starting a morph until input pauses; the morph must reflect the final committed duration and final `MM:SS`, not every intermediate minute value.
- R8. Coalescing must not leave the display stuck on an intermediate value: when coalescing ends, the visible time and ring progress must match the final setting.
- R9. A single coalesced morph still applies particle + morph rules to all digit positions that differ between the last stable displayed string and the final string.

**Transition reliability & structure**

- R10. Only one digit transition may be active at a time; a new requested transition cancels or replaces the pending one deterministically (no silent drop where the user sees an instant jump with no effect when animation was expected).
- R11. Display refresh logic must distinguish change source at least: countdown tick, settings duration commit, mode change, reset, and other non-tick updates—so tick exclusion cannot be accidental.
- R12. Particle spawn, digit morph CSS, and text swap timing must be owned by one scheduler module (functions or cohesive block in `index.html`), not duplicated across callers.
- R13. After a morph completes or is superseded, transient particle DOM and morph classes must be cleaned up so later updates are not blocked.

**Performance**

- R14. Avoid redundant layout measurement during a single morph (e.g. batch `getBoundingClientRect` per character field, not per particle).
- R15. Coalescing and superseding transitions must not accumulate orphaned timers or particle containers.

**Accessibility & settings**

- R16. `prefers-reduced-motion: reduce` continues to bypass particles and morph; display updates immediately to the final text.
- R17. Toggling “数字粒子效果” off continues to use the non-particle transition path for all R1-eligible updates.

## Key Flows

- F1. Settings duration change — single step
  - **Trigger:** User commits a new focus/short/long minute value; display will change (e.g. `25:00` → `26:00`)
  - **Steps:** Apply setting → switch to matching mode if required → schedule morph from old `MM:SS` to new → spawn particles on changed digit indices → swap text mid-transition → cleanup
  - **Outcome:** User always sees particle feedback when the main time string changes
  - **Covered by:** R1, R3, R4, R9

- F2. Settings duration change — rapid stepper
  - **Trigger:** User holds or rapidly taps +/− through several values within a short window
  - **Steps:** Update stored setting and target display as commits occur; defer morph start until pause; run one morph from last stable `MM:SS` to final `MM:SS`
  - **Outcome:** One coherent shatter to `27:00`, not three partial animations
  - **Covered by:** R7–R9

- F3. Settings change — no display change
  - **Trigger:** User commits same value, or cancels reset confirmation
  - **Steps:** Revert or skip apply; do not schedule morph
  - **Outcome:** No particles; no false “sometimes works” flicker
  - **Covered by:** R5

- F4. Running countdown tick
  - **Trigger:** Timer interval fires each second while running
  - **Steps:** Update `MM:SS` without animation flag
  - **Outcome:** Digits update plainly; no particles
  - **Covered by:** R2

- F5. Reset or mode switch (non-settings)
  - **Trigger:** User resets timer or switches focus/short/long tab with a different displayed time
  - **Steps:** Same scheduler as F1 when `MM:SS` changes
  - **Outcome:** Consistent with settings-driven changes
  - **Covered by:** R1, R4

## Acceptance Examples

- AE1. **Covers R3, R1.**
  - **Given:** Particles on; user on Short Break showing `05:00`; focus duration is 25
  - **When:** User commits focus duration 25 → 30 in Settings
  - **Then:** App shows Focus tab at `30:00` with digit particle morph on changed positions

- AE2. **Covers R7–R9.**
  - **Given:** Particles on; Focus tab at `25:00`
  - **When:** User rapidly steps 25 → 26 → 27 within one continuous press, then releases
  - **Then:** One morph to `27:00`; no requirement to animate through `26:00`

- AE3. **Covers R5.**
  - **Given:** Focus paused at `18:00`; focus duration 25
  - **When:** User changes focus to 30 and cancels the reset dialog
  - **Then:** Display remains `18:00`; no particle morph

- AE4. **Covers R2.**
  - **Given:** Particles on; timer running at `24:59`
  - **When:** One second elapses
  - **Then:** Display shows `24:58` without particle morph

- AE5. **Covers R10.**
  - **Given:** Particles on; user triggers a morph, then immediately commits another duration change before the first morph finishes
  - **Then:** User sees a single coherent transition to the latest `MM:SS`, not a frozen or half-cleared effect

- AE6. **Covers R6, R17.**
  - **Given:** Particles toggle off
  - **When:** User commits a duration change that updates the main display
  - **Then:** Non-particle fallback runs; no digit shatter particles

## Scope Boundaries

**In scope**

- Trigger rules for non-tick `MM:SS` changes (settings, mode, reset)
- Rapid stepper coalescing to final morph
- Transition scheduler refactor in `index.html`
- Reliability, structure, and performance constraints above

**Deferred for later**

- Particle animation on every countdown second
- Redesign of ambient background particles (`#particles`)
- Fine-grained visual tuning (easing curves, color gradients) beyond fixing obvious desync

**Outside this product's identity**

- Build tools, bundlers, or multi-file runtime splits (per `AGENTS.md`)

## Dependencies / Assumptions

- Assumes `docs/brainstorms/2026-05-31-settings-mode-preview-requirements.md` behaviors (auto mode switch, full duration after commit, confirmation on progress) are implemented or implemented together with this work so settings commits always reach a stable final `MM:SS`.
- Assumes digit particles remain a user-toggleable setting stored in `state.settings.particles`.
- All implementation stays in `index.html` per `AGENTS.md`.

## Outstanding Questions

None — product decisions resolved in brainstorm dialogue.
