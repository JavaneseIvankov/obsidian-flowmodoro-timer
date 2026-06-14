# Overtime Tracking Design Spec

This document specifies the design for adding an overtime tracking feature to the Obsidian Pomodoro Timer plugin. Currently, the timer stops abruptly and transitions automatically when a session's duration is surpassed. This feature will allow the timer to continue running in overtime, visually indicating it, and logging the total elapsed duration.

## 1. Requirements

### Functional Requirements
- **Configurability**: Users can toggle overtime tracking via a settings switch. It should be enabled by default.
- **Overtime Behavior**: When a work or break session exceeds the target duration, the timer should play a sound and show a notification alert, but continue running (incrementing elapsed time).
- **Time Display**: When in overtime, the elapsed time should count upwards from the target duration, formatted as `+ MM:SS` (e.g. `+ 02:45`).
- **Visual Alert**: The timer UI should visually change (e.g., color-shift the timer text and progress ring to red) when in overtime.
- **Manual Completion**: A dedicated completion button (checkmark icon) will appear during overtime. Clicking this will log the session and transition to the next mode.
- **Session Logging**: The logged session duration in daily/weekly notes or files should reflect the actual elapsed time (including the overtime duration).

---

## 2. Architecture & State Design

### 2.1 Settings Changes (`src/Settings.ts`)
We will add `enableOvertime` to the settings schema.
- **Default value**: `true`
- **UI Element**: A toggle switch under the general settings section.

### 2.2 Timer State Changes (`src/Timer.ts`)
We will add `inOvertime` to the state to track whether the timer has crossed its target duration.

```typescript
export type TimerState = {
    // Existing fields
    autostart: boolean
    running: boolean
    mode: Mode
    elapsed: number
    startTime: number | null
    inSession: boolean
    workLen: number
    breakLen: number
    count: number
    duration: number
    
    // New fields
    inOvertime: boolean
}
```

### 2.3 Timer Logic Flow
1. **Clock Tick**:
    - The elapsed milliseconds `elapsed` continue to increase.
    - If `enableOvertime` is active:
        - When `elapsed` crosses `count`, set `inOvertime = true`.
        - Trigger the timeup notification (alert sound, system/notice notification) exactly once at the boundary.
        - Do not reset `elapsed` or transition to the next mode.
    - If `enableOvertime` is disabled:
        - Behave as before: cap `elapsed` at `count` and execute transition immediately.
2. **Time Rendering (`remain` method)**:
    - If `elapsed > count`, compute `diff = elapsed - count`.
    - Format as `+ MM:SS`.
3. **Session Completion**:
    - Triggered via user interaction.
    - Write the log (using full `elapsed` duration).
    - Increment actual task count (using existing task tracking code).
    - Transition to the next mode (`WORK` or `BREAK`) and reset `inOvertime` to `false`.

---

## 3. UI Implementation Details

### 3.1 Svelte Component Updates (`src/TimerViewComponent.svelte`)
- Bind the class `in-overtime` to the container element when `$timer.inOvertime` is true.
- Add a new "Complete Session" button rendering a checkmark SVG. This button will only be visible when `$timer.inOvertime` is true, and will call `timer.completeSession()`.

### 3.2 Svelte Component Updates (`src/StatusBarComponent.svelte`)
- Display the formatted overtime string (e.g., `+ 01:30`) in the status bar if enabled.
- Add a context menu item or handle status bar updates appropriately to show overtime.

### 3.3 Style Updates (`styles.css` / Svelte style block)
- Add styling for the `.in-overtime` state.
- Change the timer elapsed stroke to a crimson/red accent (`var(--text-error)` or similar).
- Change the timer text color to match.

---

## 4. Testing Plan
- **Standard Run**: Run a pomodoro with `enableOvertime` off; verify it transitions automatically.
- **Overtime Run**: Run a pomodoro with `enableOvertime` on; verify it plays sound/notice at the threshold, keeps counting up, shows `+ MM:SS`, and turns red.
- **Pause/Resume Overtime**: Verify the overtime count can be paused and resumed correctly.
- **Complete Overtime**: Click completion button; verify it transitions, logs correct duration, and increments actual task pomodoro counts in markdown notes.
