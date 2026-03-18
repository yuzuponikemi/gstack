# Pre-Landing Review Checklist

## Instructions

Review the `git diff origin/main` output for the issues listed below. Be specific — cite `file:line` and suggest fixes. Skip anything that's fine. Only flag real problems.

**Two-pass review:**
- **Pass 1 (CRITICAL):** Run Hardware Safety, Thread Safety, Resource Management, and State Machine Completeness first. Highest severity.
- **Pass 2 (INFORMATIONAL):** Run all remaining categories. Lower severity but still actioned.

All findings get action via Fix-First Review: obvious mechanical fixes are applied automatically,
genuinely ambiguous issues are batched into a single user question.

**Output format:**

```
Pre-Landing Review: N issues (X critical, Y informational)

**AUTO-FIXED:**
- [file:line] Problem → fix applied

**NEEDS INPUT:**
- [file:line] Problem description
  Recommended fix: suggested fix
```

If no issues found: `Pre-Landing Review: No issues found.`

Be terse. For each issue: one line describing the problem, one line with the fix. No preamble, no summaries, no "looks good overall."

---

## Review Categories

### Pass 1 — CRITICAL

#### Hardware Safety
- Stage move commands sent without soft-limit check — hitting hard limits causes mechanical damage
- Pneumatic valve open/close sequence differs from defined safe order (e.g., pressurizing before exhausting)
- Fluid control commands sent without flow rate or pressure upper-bound validation
- Interlock conditions (other axis state, air pressure confirmation, etc.) bypassed by a conditional branch
- `finally` block missing: on exception, device is not returned to safe state (stage stop, valve close, camera stop)
- New device command added but emergency-stop handler does not cover it

#### Thread Safety (WinForms)
- `Control.Text` / `Control.Visible` / `Control.Enabled` updated directly from a non-UI thread without `InvokeRequired` check
- Device objects (stage controller, valve driver, etc.) accessed concurrently from multiple threads without `lock`
- Exceptions inside `BackgroundWorker` / `Task` swallowed and never surfaced to the UI
- Long device-wait loop does not accept `CancellationToken` — process cannot be stopped cleanly

#### Resource Management
- `SerialPort`, USB device, or camera SDK object not wrapped in `using` or `Dispose()` not called on error path
- Camera image buffer not released — leads to memory exhaustion in long-running sessions
- Partial resource acquisition on connection failure not cleaned up (e.g., port opened but camera init fails)

#### State Machine Completeness
- New value added to device state enum (`Idle` / `Homing` / `Moving` / `Error` etc.) but not handled in all `switch` statements
- Move command accepted while device is in `Homing` or `Error` state — must be rejected
- Disconnect/reconnect path does not reset internal state to `Idle`

---

### Pass 2 — INFORMATIONAL

#### Timeout & Polling
- Device response wait loop (`while (true)`) has no timeout upper bound
- Polling interval is a magic number in `Thread.Sleep()` — should be a named constant
- Communication timeout value duplicated between `SerialPort.ReadTimeout` and application-level retry logic, risking drift

#### Coordinate & Unit Consistency
- Stage coordinate units (mm / μm / pulses) mixed across methods without explicit conversion
- Camera-to-stage coordinate transform matrix hardcoded inline instead of a named calibration constant
- Flow rate or pressure unit conversion duplicated in multiple places — single utility method preferred

#### Error Handling Quality
- `catch (Exception e)` logs the error but does not reset device state — device left in unknown state
- Error message exposes raw SDK error code with no operator-readable explanation
- Communication error retry loop has no maximum retry count — can loop indefinitely

#### Camera-Specific
- Stage move command can be issued while camera exposure is in progress — causes motion blur in acquired image
- On image acquisition timeout, camera remains in acquisition mode — subsequent captures fail silently
- `Bitmap` assigned to `PictureBox.Image` for live preview without calling `Dispose()` on the previous value — GDI resource leak

#### Testability & Diagnostics
- Device send/receive calls not abstracted behind an interface — UI cannot be tested without physical hardware
- Communication log (timestamped byte-level send/receive) not recorded — fault analysis is difficult
- Magic threshold values (pressure limits, stage speed caps, flow rate maxima) not centralized — hard to audit safety parameters

#### Dead Code & Consistency
- Variables assigned but never read
- Version mismatch between PR title and VERSION/CHANGELOG files
- Comments or docstrings describing old behavior after code changed

---

## Severity Classification

```
CRITICAL (highest severity):          INFORMATIONAL (lower severity):
├─ Hardware Safety                    ├─ Timeout & Polling
├─ Thread Safety (WinForms)           ├─ Coordinate & Unit Consistency
├─ Resource Management                ├─ Error Handling Quality
└─ State Machine Completeness         ├─ Camera-Specific
                                      ├─ Testability & Diagnostics
                                      └─ Dead Code & Consistency

All findings are actioned via Fix-First Review. Severity determines
presentation order and classification of AUTO-FIX vs ASK — critical
findings lean toward ASK (they're riskier), informational findings
lean toward AUTO-FIX (they're more mechanical).
```

---

## Fix-First Heuristic

This heuristic is referenced by both `/review` and `/ship`. It determines whether
the agent auto-fixes a finding or asks the user.

```
AUTO-FIX (agent fixes without asking):     ASK (needs human judgment):
├─ Magic numbers → named constants         ├─ Interlock condition changes
├─ Add InvokeRequired / Invoke() wrap      ├─ Soft-limit value changes
├─ Wrap resource in using block            ├─ Valve sequence changes
├─ Add timeout upper bound                 ├─ Safe-state recovery logic
├─ Add communication log call              ├─ Coordinate transform corrections
├─ Dead code / unused variables            ├─ Large fixes (>20 lines)
└─ Stale comments contradicting code       └─ Anything changing device behavior
```

**Rule of thumb:** If the fix is mechanical and a senior engineer would apply it
without discussion, it's AUTO-FIX. If reasonable engineers could disagree about
the fix — especially when physical hardware safety is involved — it's ASK.

**Critical findings default toward ASK** (hardware safety mistakes can cause
physical damage or injury).
**Informational findings default toward AUTO-FIX** (they're more mechanical).

---

## Suppressions — DO NOT flag these

- "Add a comment explaining why this threshold was chosen" — thresholds are tuned empirically and change constantly
- "This assertion could be tighter" when the assertion already covers the behavior
- Suggesting consistency-only changes that don't affect correctness or safety
- Harmless no-ops that don't affect device state
- ANYTHING already addressed in the diff you're reviewing — read the FULL diff before commenting
