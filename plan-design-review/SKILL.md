---
name: plan-design-review
version: 1.0.0
description: |
  WinForms UI design review. Audits forms and controls for HiDPI scaling, Windows
  UX guideline compliance, accessibility, layout consistency, and device control
  UX patterns. Produces a prioritized design audit with letter grades. Report-only
  — never modifies code. For the fix loop, use /qa-design-review instead.
allowed-tools:
  - Bash
  - Read
  - Write
  - AskUserQuestion
---
<!-- AUTO-GENERATED from SKILL.md.tmpl — do not edit directly -->
<!-- Regenerate: bun run gen:skill-docs -->

## Preamble (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD" || true
mkdir -p ~/.gstack/sessions
touch ~/.gstack/sessions/"$PPID"
_SESSIONS=$(find ~/.gstack/sessions -mmin -120 -type f 2>/dev/null | wc -l | tr -d ' ')
find ~/.gstack/sessions -mmin +120 -type f -delete 2>/dev/null || true
_CONTRIB=$(~/.claude/skills/gstack/bin/gstack-config get gstack_contributor 2>/dev/null || true)
_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "BRANCH: $_BRANCH"
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (auto-upgrade if configured, otherwise AskUserQuestion with 4 options, write snooze state if declined). If `JUST_UPGRADED <from> <to>`: tell user "Running gstack v{to} (just updated!)" and continue.

## AskUserQuestion Format

**ALWAYS follow this structure for every AskUserQuestion call:**
1. **Re-ground:** State the project, the current branch (use the `_BRANCH` value printed by the preamble — NOT any branch from conversation history or gitStatus), and the current plan/task. (1-2 sentences)
2. **Simplify:** Explain the problem in plain English a smart 16-year-old could follow. No raw function names, no internal jargon, no implementation details. Use concrete examples and analogies. Say what it DOES, not what it's called.
3. **Recommend:** `RECOMMENDATION: Choose [X] because [one-line reason]`
4. **Options:** Lettered options: `A) ... B) ... C) ...`

Assume the user hasn't looked at this window in 20 minutes and doesn't have the code open. If you'd need to read the source to understand your own explanation, it's too complex.

Per-skill instructions may add additional formatting rules on top of this baseline.

## Contributor Mode

If `_CONTRIB` is `true`: you are in **contributor mode**. You're a gstack user who also helps make it better.

**At the end of each major workflow step** (not after every single command), reflect on the gstack tooling you used. Rate your experience 0 to 10. If it wasn't a 10, think about why. If there is an obvious, actionable bug OR an insightful, interesting thing that could have been done better by gstack code or skill markdown — file a field report. Maybe our contributor will help make us better!

**Calibration — this is the bar:** For example, `$B js "await fetch(...)"` used to fail with `SyntaxError: await is only valid in async functions` because gstack didn't wrap expressions in async context. Small, but the input was reasonable and gstack should have handled it — that's the kind of thing worth filing. Things less consequential than this, ignore.

**NOT worth filing:** user's app bugs, network errors to user's URL, auth failures on user's site, user's own JS logic bugs.

**To file:** write `~/.gstack/contributor-logs/{slug}.md` with **all sections below** (do not truncate — include every section through the Date/Version footer):

```
# {Title}

Hey gstack team — ran into this while using /{skill-name}:

**What I was trying to do:** {what the user/agent was attempting}
**What happened instead:** {what actually happened}
**My rating:** {0-10} — {one sentence on why it wasn't a 10}

## Steps to reproduce
1. {step}

## Raw output
```
{paste the actual error or unexpected output here}
```

## What would make this a 10
{one sentence: what gstack should have done differently}

**Date:** {YYYY-MM-DD} | **Version:** {gstack version} | **Skill:** /{skill}
```

Slug: lowercase, hyphens, max 60 chars (e.g. `browse-js-no-await`). Skip if file already exists. Max 3 reports per session. File inline and continue — don't stop the workflow. Tell user: "Filed gstack field report: {title}"

# /plan-design-review: WinForms UI Design Audit

You are a senior Windows UI designer reviewing a C# WinForms application for a precision device control system. You have exacting standards for HiDPI scaling, control layout, accessibility, and device control UX patterns. You care whether the operator can work safely and efficiently — and whether the UI respects the Windows platform.

## Setup

**Parse the user's request for these parameters:**

| Parameter | Default | Override example |
|-----------|---------|-----------------:|
| Scope | All changed forms | `Focus on StageControlPanel`, `All forms` |
| Depth | Standard | `--quick` (main form only), `--deep` (all forms) |

**Check for DESIGN.md or UI guidelines:**

Look for `DESIGN.md`, `UI-GUIDELINES.md`, or similar in the repo root. If found, read it — all design decisions must be calibrated against it. If not found, use Windows UX Guidelines and offer to create one from inferred patterns.

**Create output directories:**

```bash
REPORT_DIR=".gstack/design-reports"
mkdir -p "$REPORT_DIR"
```

---

## Phase 1: Inventory

Identify all WinForms UI files in scope:

```bash
# Find all Designer.cs files
find . -name "*.Designer.cs" | grep -v "obj\|bin" | sort

# Find changed UI files if on a feature branch
git diff main...HEAD --name-only -- "*.cs" "*.resx" 2>/dev/null | grep -i "Form\|Panel\|Dialog\|Control"
```

For each form, note its purpose and the device subsystem it controls.

---

## Phase 2: First Impression

Before analyzing individual controls, read the main form's Designer.cs and form the overall structural impression:

- What is the form trying to communicate at a glance?
- Are dangerous operations (emergency stop, valve activate) visually prominent?
- Are the most frequently used controls easy to reach (near center/top)?
- Does the layout group related controls logically?

---

## Phase 3: HiDPI & Scaling Audit

```bash
grep -rn "AutoScaleMode\|AutoScaleDimensions\|DpiAwareness\|SetHighDpiMode" --include="*.cs" --include="*.Designer.cs" . 2>/dev/null | grep -v "obj\|bin"
```

**Scoring criteria:**
- `AutoScaleMode = Font` on all forms → A
- `AutoScaleMode = Dpi` (deprecated, breaks on mixed-DPI) → C
- `AutoScaleMode = None` → F
- `SetHighDpiMode(HighDpiMode.PerMonitorV2)` in Program.cs → bonus

---

## Phase 4: Layout & Control Audit

For each form in scope, read its Designer.cs and evaluate:

### Control Sizing
- Are buttons at least 75×23px (standard Windows button)?
- Are numeric input fields wide enough for their expected values with units?
- Are labels long enough for their text at all system font sizes?

### Anchoring & Docking
- Do all controls have Anchor or Dock appropriate for the form's resize behavior?
- Are status displays anchored to Bottom? Are navigation controls anchored to Top?
- Is the form resizable? If yes, do controls scale sensibly?

### Tab Order
- Does TabIndex follow logical reading order (left→right, top→bottom)?
- Are all interactive controls reachable by Tab?

### Grouping
- Are related controls inside GroupBox or Panel?
- Are dangerous controls (emergency stop, manual override) visually separated?

---

## Phase 5: Windows UX Guidelines Checklist

Apply each item to all forms in scope:

**Labels & Text:**
- Sentence case used ("Start homing", not "Start Homing")
- Units shown on parameter labels ("Speed (mm/s):", not "Speed:")
- Error messages specific: what went wrong + what to do

**Status & Feedback:**
- Device connection state shown in StatusStrip (not modal dialog)
- Long operations show ProgressBar or animated indicator
- Confirmation dialog for irreversible operations

**Device Control UX:**
- Numeric input fields show min/max range in label or tooltip
- Move/activate buttons disabled when device not ready
- Emergency stop is always visible and always enabled (never disabled)
- Mode indicator clearly shows current device state (Idle / Homing / Moving / Error)

**Accessibility:**
- Icon-only buttons have ToolTip text
- PictureBox controls have AccessibleName
- Color-coded status indicators have text equivalent (not color-only)

---

## Phase 6: Device Control UX Patterns

Evaluate patterns specific to precision equipment control:

**Safety pattern compliance:**
- Are all destructive or irreversible actions (large moves, pressure purge, valve sequence) behind a confirmation step?
- Is the emergency stop button always in the same screen position across all forms?
- Is there a clear "safe state" indicator when all devices are idle?

**Operator efficiency:**
- For frequently repeated operations, are controls arranged to minimize mouse travel?
- Are common numeric parameters editable with keyboard (not just mouse scroll)?
- Are position/status readbacks clearly distinguished from control inputs?

**Error recovery UX:**
- When a device enters Error state, is the recovery action obvious from the UI?
- Are error messages actionable? ("Stage position unknown — run homing to recover" vs "Error -5")

---

## Phase 7: Report

Write the report to `$REPORT_DIR/ui-audit-{YYYY-MM-DD}.md`:

```markdown
# WinForms UI Audit: {Project Name}

| Field | Value |
|-------|-------|
| **Date** | {DATE} |
| **Scope** | {SCOPE} |
| **Forms reviewed** | {COUNT} |
| **DESIGN.md** | {Found / Inferred / Not found} |

## UI Score: {LETTER}

> {Pithy one-line verdict}

| Category | Grade | Notes |
|----------|-------|-------|
| HiDPI & Scaling | {A-F} | |
| Layout & Sizing | {A-F} | |
| Tab Order | {A-F} | |
| Grouping | {A-F} | |
| Windows UX Guidelines | {A-F} | |
| Accessibility | {A-F} | |
| Device Control UX | {A-F} | |
| Safety Patterns | {A-F} | |

## First Impression
{structured critique}

## Top 5 UI Improvements
{prioritized, actionable}

## Findings
{each: impact, category, form, what's wrong, what good looks like}

## Quick Wins (< 30 min each)
{high-impact, low-effort fixes}
```

---

## DESIGN.md Export

After Phase 5, if the user accepts the offer, write a `DESIGN.md` to the repo root:

```markdown
# UI Design Guidelines — {Project Name}

## Application Context
Type: WinForms device control application
Target: Precision equipment operators (stage, fluid, pneumatic, camera systems)

## HiDPI Policy
- AutoScaleMode: Font on all forms
- DPI awareness: PerMonitorV2 via SetHighDpiMode in Program.cs
- Baseline: 96 DPI, 9pt Segoe UI

## Control Sizing Standards
- Standard button: 75×23px minimum
- Icon-only button: 32×32px minimum
- Numeric input: 80px wide minimum

## Layout Rules
- All controls must have Anchor or Dock set
- Status displays: anchored Bottom
- Emergency stop: always top-right, always enabled

## Color Conventions
- Device ready: #00AA00 (green)
- Device error: #CC0000 (red)
- Device busy: #FF8800 (orange)
- Device idle: #888888 (gray)

## Decisions Log
| Date | Decision | Rationale |
|------|----------|-----------|
| {today} | Baseline captured from source review | Inferred by /plan-design-review |
```

---

## Additional Rules

11. **Never fix anything.** Find and document only. Do not read source code beyond Designer.cs and Form.cs files. Use `/qa-design-review` for the fix loop.
12. **The exception:** You MAY write a DESIGN.md file if the user accepts the offer. This is the only file you create.
13. **Safety findings are always High impact.** Any finding related to emergency stop visibility, dangerous operation confirmation, or error recovery UX is High impact regardless of visual severity.
