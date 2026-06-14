# Overtime Tracking Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Allow the Obsidian Pomodoro Timer to run in overtime when a session's duration is surpassed, showing a `+ MM:SS` elapsed time, updating the UI color to indicate overtime, and allowing manual completion/logging of the session.

**Architecture:** Add a settings option to enable overtime (default: true) and an `inOvertime` flag to the timer state. During clock ticks, when the timer crosses the limit, play the notification sound but keep ticking. Format the remaining time with a `+` prefix and display a completion button in the UI.

**Tech Stack:** TypeScript, Svelte, Obsidian API

---

### Task 1: Add Overtime Settings configuration

**Files:**
- Modify: `src/Settings.ts`

- [ ] **Step 1: Modify Settings interface and Default Settings**

In `src/Settings.ts`, add `enableOvertime` to the `Settings` interface and to the `DEFAULT_SETTINGS` object.

```typescript
export interface Settings {
    // ... existing settings
    enableOvertime: boolean
}

// In DEFAULT_SETTINGS:
static readonly DEFAULT_SETTINGS: Settings = {
    // ... existing defaults
    enableOvertime: true,
}
```

- [ ] **Step 2: Add setting toggle in Settings Tab**

In `src/Settings.ts` inside the `display()` method, add the toggle control for Overtime.

```typescript
new Setting(containerEl)
    .setName('Enable Overtime')
    .setDesc('Keep the timer running in overtime when the target duration is reached, and wait for manual completion.')
    .addToggle((toggle) => {
        toggle.setValue(this._settings.enableOvertime)
        toggle.onChange((value) => {
            this.updateSettings({ enableOvertime: value })
        })
    })
```

- [ ] **Step 3: Run build to verify compilation**

Run: `npm run build`
Expected output: SUCCESS (no TypeScript errors).

- [ ] **Step 4: Commit settings changes**

```bash
git add src/Settings.ts
git commit -m "feat: add enableOvertime setting and UI toggle"
```

---

### Task 2: Implement Overtime Logic in Timer class

**Files:**
- Modify: `src/Timer.ts`

- [ ] **Step 1: Modify TimerState and TimerStore types**

In `src/Timer.ts`, add `inOvertime` to `TimerState` and update the derived store.

```typescript
export type TimerState = {
    // ... existing fields
    inOvertime: boolean
}

// In the constructor, initialize s.inOvertime to false:
this.state = {
    // ... existing fields
    inOvertime: false,
}
```

- [ ] **Step 2: Update `remain` to support counting up in overtime**

In `src/Timer.ts`, modify `remain(count, elapsed)` to check if `elapsed > count`, compute the difference, and format it with a `+` prefix.

```typescript
    private remain(count: number, elapsed: number): TimerRemained {
        let remained = count - elapsed
        let isOvertime = remained < 0
        let absRemained = Math.abs(remained)
        let min = Math.floor(absRemained / 60000)
        let sec = Math.floor((absRemained % 60000) / 1000)
        let minStr = min < 10 ? `0${min}` : min.toString()
        let secStr = sec < 10 ? `0${sec}` : sec.toString()
        return {
            millis: remained,
            human: `${isOvertime ? '+ ' : ''}${minStr} : ${secStr}`,
        }
    }
```

- [ ] **Step 3: Update `tick` logic for overtime triggering**

Update `tick(t)` to handle the transition into overtime and play the notification without stopping or transitioning automatically.

```typescript
    private tick(t: number) {
        let justFinished: boolean = false
        let pause: boolean = false
        this.update((s) => {
            if (s.running) {
                let prevElapsed = s.elapsed
                s.elapsed += t
                
                const enableOvertime = this.plugin.getSettings().enableOvertime
                if (enableOvertime) {
                    if (prevElapsed < s.count && s.elapsed >= s.count) {
                        s.inOvertime = true
                        justFinished = true
                    }
                } else {
                    if (s.elapsed >= s.count) {
                        s.elapsed = s.count
                        justFinished = true
                    }
                }
            } else {
                pause = true
            }
            return s
        })
        
        if (!pause && justFinished) {
            const enableOvertime = this.plugin.getSettings().enableOvertime
            if (enableOvertime) {
                // Play notification, but do not transition or write log file yet.
                this.update((s) => {
                    const ctx = this.createLogContext(s)
                    this.notify(ctx, undefined, true) // Play sound, no logFile yet
                    return s
                })
            } else {
                this.timeup()
            }
        }
    }
```

- [ ] **Step 4: Update `processLog`, `notify`, `completeSession`, and `endSession`**

Update `processLog` and `notify` to support disabling the notification sound when completing the session. Add `completeSession` to handle manual completion. Reset `inOvertime` in `endSession`.

```typescript
    // Update processLog signature:
    private async processLog(ctx: LogContext, playSound: boolean = true) {
        if (ctx.mode == 'WORK') {
            await this.plugin.tracker?.updateActual()
        }
        const logFile = await this.logger.log(ctx)
        this.notify(ctx, logFile, playSound)
    }

    // Update notify signature and check playSound:
    private notify(state: TimerState, logFile: TFile | void, playSound: boolean = true) {
        // ... existing logic
        if (playSound && this.plugin.getSettings().notificationSound) {
            this.playAudio()
        }
    }

    // Add completeSession method:
    public completeSession() {
        let autostart = false
        this.update((state) => {
            const ctx = this.createLogContext(state)
            this.processLog(ctx, false) // Log and update actual, do not play notification sound again
            autostart = state.autostart
            return this.endSession(state)
        })
        if (autostart) {
            this.start()
        }
    }

    // In endSession, reset state.inOvertime:
    private endSession(state: TimerState) {
        // ... existing transitions
        state.inOvertime = false
        return state
    }
```

- [ ] **Step 5: Run build to verify compilation**

Run: `npm run build`
Expected output: SUCCESS (no TypeScript errors).

- [ ] **Step 6: Commit timer logic changes**

```bash
git add src/Timer.ts
git commit -m "feat: implement timer overtime logic and counting up"
```

---

### Task 3: Implement Svelte UI Updates

**Files:**
- Modify: `src/TimerViewComponent.svelte`
- Modify: `styles.css`

- [ ] **Step 1: Cap Svelte circular progress bar strokeOffset**

In `src/TimerViewComponent.svelte`, cap `strokeOffset` at a minimum of 0 when in overtime so the circle stays fully filled.

```typescript
$: strokeOffset = Math.max(0, $timer.remained.millis / $timer.count * offset)
```

- [ ] **Step 2: Add conditional class `in-overtime` and Complete Button**

Apply `in-overtime` class conditionally on the main container. Add a checkmark complete button next to the controls when `$timer.inOvertime` is true.

```svelte
<!-- In TimerViewComponent.svelte container div -->
<div class="container" class:in-overtime={$timer.inOvertime}>
```

In the `.btn-group` controls list:
```svelte
            {#if $timer.inOvertime}
                <span on:click={() => timer.completeSession()} class="control" title="Complete Session">
                    <svg xmlns="http://www.w3.org/2000/svg" width="16" height="16" viewBox="0 0 24 24" fill="none" stroke="currentColor" stroke-width="2" stroke-linecap="round" stroke-linejoin="round" class="lucide lucide-check-check"><path d="m12 15 2 2 4-4"/><path d="m4 12 2 2 4-4"/><path d="m14 4-2 2-4-4"/></svg>
                </span>
            {/if}
```

- [ ] **Step 3: Add Overtime styling**

In `styles.css`, add the styling for the `.in-overtime` selector:

```css
.in-overtime .circle_animation {
    stroke: var(--text-error);
}

.in-overtime .timer-text {
    color: var(--text-error);
}
```

- [ ] **Step 4: Run build to verify compilation**

Run: `npm run build`
Expected output: SUCCESS (no build or CSS compilation errors).

- [ ] **Step 5: Commit UI and CSS changes**

```bash
git add src/TimerViewComponent.svelte styles.css
git commit -m "feat: add complete button and color styling for overtime timer"
```
