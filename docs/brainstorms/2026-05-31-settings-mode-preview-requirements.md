---
date: 2026-05-31
topic: settings-mode-preview
---

# Settings Mode Preview — Requirements

## Summary

When a user changes focus, short break, or long break duration in Settings, the app switches to the matching main mode tab and shows the new full duration. Settings cannot be opened while the timer is running; if the timer has already consumed time, changing a duration requires confirmation and resets that mode's timer from the beginning. Canceling the confirmation restores the previous value without saving.

## Problem Frame

Today, duration changes only update the main timer when the user is already on the matching mode tab. Changing focus duration while on short break saves the value but gives no visible feedback on the main screen. Users who adjust settings to preview or verify a duration cannot see the result without manually switching tabs. Additionally, changing duration mid-session can silently alter remaining time in ways that are hard to predict.

## Key Decisions

- **Settings access while running:** Block opening Settings when the timer is actively running. User must pause first.
- **Duration change with progress:** If the timer has progress (paused mid-session or any state where elapsed time exists), changing a duration triggers a confirmation that the timer will restart from the new full duration.
- **Cancel behavior:** If the user cancels the confirmation, do not save the new value; restore the input and setting to the previous value.
- **After save — timer display:** Always show the new full duration for the target mode (not a proportional adjustment of remaining time).
- **Mode navigation:** Every successful duration change for focus, short break, or long break automatically switches to that mode's main tab so the user sees the updated time immediately.
- **Trigger moments:** Auto-switch applies on every effective change via stepper (+/−), direct input blur, and Enter — whenever the new value is actually applied.
- **Pomodoro count setting:** Changing "pomodoros before long break" does not trigger mode switch or timer reset confirmation.

## Requirements

**Settings access**

- R1. When the timer is running (`state.running === true`), tapping the Settings control must not open the Settings panel. Show a brief message instructing the user to pause first.
- R2. When the timer is paused or idle, Settings opens normally.

**Duration change — confirmation**

- R3. When the user applies a new value for focus, short break, or long break duration, and the timer has progress (`timeLeft !== totalTime` or equivalent "has run" signal), show a confirmation: modifying the duration will reset the timer to the new full duration from the beginning.
- R4. If the user confirms, save the new duration, switch to the corresponding mode tab, and display the full new duration.
- R5. If the user cancels, do not save. Restore the input field and stored setting to the value before the edit attempt.
- R6. When there is no timer progress (full duration, never started or reset to start), apply the new duration without confirmation, switch to the corresponding mode tab, and show the full new duration.

**Mode mapping**

- R7. Focus duration change → switch to Focus tab.
- R8. Short break duration change → switch to Short Break tab.
- R9. Long break duration change → switch to Long Break tab.

**Apply triggers**

- R10. Mode switch and duration apply occur whenever a duration value is successfully committed via stepper (+/−), input blur, or Enter — not when the value is unchanged.

**Out of scope for this change**

- R11. Changing pomodoro count before long break does not auto-switch modes and does not trigger the duration-reset confirmation flow.

## Key Flows

- F1. Open Settings while timer running
  - **Trigger:** User taps Settings while `state.running === true`
  - **Steps:** Reject open; show "请先暂停计时器" (or equivalent) message
  - **Outcome:** Settings panel stays closed
  - **Covered by:** R1

- F2. Change duration with no progress
  - **Trigger:** User commits a new focus/short/long duration; timer at full length with no elapsed time
  - **Steps:** Save value → switch to matching tab → show full new duration on ring and display
  - **Outcome:** User immediately sees the new time on the correct mode
  - **Covered by:** R6, R7–R10

- F3. Change duration with progress — confirm
  - **Trigger:** User commits a new duration while timer has progress
  - **Steps:** Show confirm dialog → on OK: save, switch tab, reset to full new duration → on Cancel: revert input and setting to previous value
  - **Outcome:** Confirmed changes are visible on the correct tab; canceled changes leave state unchanged
  - **Covered by:** R3–R5, R7–R10

## Acceptance Examples

- AE1. **Covers R1.**
  - **Given:** Focus timer running at 18:00
  - **When:** User taps Settings
  - **Then:** Panel does not open; user sees pause-first message

- AE2. **Covers R6, R8.**
  - **Given:** User on Focus tab, timer at 25:00 idle; short break is 5 min
  - **When:** User opens Settings, changes short break to 8 via + button
  - **Then:** App switches to Short Break tab; display shows 08:00

- AE3. **Covers R3–R5.**
  - **Given:** Focus paused at 18:00; focus duration was 25
  - **When:** User changes focus to 30 and confirms the reset dialog
  - **Then:** Focus tab active; display shows 30:00

- AE4. **Covers R5.**
  - **Given:** Focus paused at 18:00; focus duration was 25
  - **When:** User changes focus to 30 and cancels the reset dialog
  - **Then:** Setting remains 25; input shows 25; timer still shows 18:00 on Focus tab

- AE5. **Covers R11.**
  - **Given:** User on any tab
  - **When:** User changes "pomodoros before long break" from 4 to 6
  - **Then:** No mode switch; no duration-reset confirmation

## Scope Boundaries

**In scope**

- Gating Settings open while running
- Confirm + reset on duration change when timer has progress
- Auto-switch to matching mode on duration change
- Cancel restores previous value

**Deferred for later**

- Visual disabled state on Settings button while running (toast-only is sufficient for v1)
- Escape-to-close Settings while input focused (pre-existing behavior)

**Outside this product's identity**

- Multi-user or cloud-synced settings
- Separate "preview mode" that doesn't affect saved settings

## Dependencies / Assumptions

- Assumes existing mode tabs (Focus / Short Break / Long Break) and Settings stepper inputs remain the primary UI.
- Assumes `hasTimerProgress()` or equivalent logic correctly detects "timer has run" including paused mid-session.
- Single-file app constraint per `AGENTS.md` — all behavior changes stay in `index.html`.

## Outstanding Questions

None — product decisions resolved in brainstorm dialogue.
