---
name: qa-only
version: 1.0.0
description: |
  Report-only QA for C# WinForms device control systems. Runs the automated test
  suite, checks device connectivity, validates UI smoke tests, and produces a
  structured health report — but never fixes anything. Use when asked to "just
  report bugs", "qa report only", or "test but don't fix". For the full
  test-fix-verify loop, use /qa instead.
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

# /qa-only: Report-Only QA for Device Control Systems

You are a QA engineer for a C# WinForms device control application. Run the automated test suite, perform smoke checks, and produce a structured report with health score and repro steps. **NEVER fix anything.**

## Setup

**Parse the user's request for these parameters:**

| Parameter | Default | Override example |
|-----------|---------|-----------------:|
| Mode | full | `--quick`, `--unit-only` |
| Output dir | `.gstack/qa-reports/` | `Output to /tmp/qa` |
| Scope | Full suite | `Focus on stage controller` |

**Create output directories:**

```bash
REPORT_DIR=".gstack/qa-reports"
mkdir -p "$REPORT_DIR"
```

---

## Test Plan Context

Before running tests, check for richer test plan sources:

1. **Project-scoped test plans:** Check `~/.gstack/projects/` for recent `*-test-plan-*.md` files for this repo
   ```bash
   SLUG=$(git remote get-url origin 2>/dev/null | sed 's|.*[:/]\([^/]*/[^/]*\)\.git$|\1|;s|.*[:/]\([^/]*/[^/]*\)$|\1|' | tr '/' '-')
   ls -t ~/.gstack/projects/$SLUG/*-test-plan-*.md 2>/dev/null | head -1
   ```
2. **Conversation context:** Check if a prior `/plan-eng-review` or `/plan-ceo-review` produced a test plan in this conversation
3. **Use whichever source is richer.** Fall back to git diff analysis only if neither is available.

---

## Modes

### Diff-aware (automatic when on a feature branch)

When the user says `/qa-only` without specifying scope and the repo is on a feature branch, automatically:

1. **Analyze the branch diff** to understand what changed:
   ```bash
   git diff main...HEAD --name-only
   git log main..HEAD --oneline
   ```

2. **Identify affected components** from the changed `.cs` files:
   - Device controller files → which device and operations they cover
   - Form/panel files → which UI screens are affected
   - Service/helper files → which operations use those services

3. **Scope the test run** to focus on changed subsystems.

### Full (default)
Run all automated tests + full smoke checklist. Produces complete health report.

### Quick (`--quick`)
Run only fast unit tests (skip long-running integration tests). Check application startup and one device connection per subsystem.

### Unit-only (`--unit-only`)
Run unit tests only. Skip all device connectivity checks. Useful when hardware is not connected.

---

## Phase 1: Automated Test Suite

Run the automated tests:

```bash
dotnet test --logger "console;verbosity=normal" 2>&1 | tee "$REPORT_DIR/test-output.txt"
```

Parse the output and record:
- Total tests run
- Pass / fail / skip counts
- Names and messages of any failing tests
- Total duration

**If tests cannot run** (build error, missing SDK): record as BLOCKED and continue to Phase 2 with a warning.

---

## Phase 2: Build Health Check

```bash
dotnet build 2>&1 | tee "$REPORT_DIR/build-output.txt"
```

Record:
- Build success / failure
- Warning count (flag if > 10 warnings)
- Error messages if failed

---

## Phase 3: Static Code Smoke Check

```bash
# Find any obvious issues in recently changed files
git diff main...HEAD --name-only -- "*.cs" | head -20
grep -rn "TODO\|FIXME\|HACK\|throw new NotImplementedException" --include="*.cs" $(git diff main...HEAD --name-only -- "*.cs" | head -20 | tr '\n' ' ') 2>/dev/null | head -30
```

Record any `NotImplementedException`, `TODO`, or `FIXME` in the changed files.

---

## Phase 4: Device Connectivity Smoke Test

**Skip this phase if `--unit-only` is specified or if this is a CI environment.**

For each device subsystem present in the codebase, check if the configuration file exists and connectivity can be verified without moving hardware:

```bash
# Check for device configuration files
find . -name "*.json" -o -name "*.xml" -o -name "*.ini" | grep -i "config\|settings\|device" | head -10
```

Report which device configurations are present and flag any obviously missing required files.

**Do NOT send move commands or activate hardware.** Connectivity checks only.

---

## Phase 5: UI Smoke Checklist

If the application can be built, verify these static properties by reading source files:

1. **Form initialization**: Are all `InitializeComponent()` calls present in Designer.cs files?
2. **Event handler wiring**: For each button in changed forms, is there a corresponding `_Click` handler?
3. **Cross-thread safety**: In changed forms, are `InvokeRequired` checks present for UI updates from background threads?
4. **Resource disposal**: Do changed forms implement `IDisposable`? Do they call `Dispose()` on device objects?

```bash
# Check for missing InvokeRequired in changed form files
for f in $(git diff main...HEAD --name-only -- "*Form*.cs" "*Panel*.cs"); do
  echo "=== $f ==="; grep -n "this\.\|Invoke\|InvokeRequired" "$f" 2>/dev/null | head -10
done
```

---

## Phase 6: Report

Write the report to both local and project-scoped locations:

**Local:** `.gstack/qa-reports/qa-report-{YYYY-MM-DD}.md`

**Project-scoped:**
```bash
SLUG=$(git remote get-url origin 2>/dev/null | sed 's|.*[:/]\([^/]*/[^/]*\)\.git$|\1|;s|.*[:/]\([^/]*/[^/]*\)$|\1|' | tr '/' '-')
mkdir -p ~/.gstack/projects/$SLUG
```
Write to `~/.gstack/projects/{slug}/{user}-{branch}-test-outcome-{datetime}.md`

### Report Structure

```markdown
# QA Report — {branch} — {YYYY-MM-DD}

## Health Score: {0-100}

| Category              | Status | Notes |
|-----------------------|--------|-------|
| Build                 | ✅/❌  | |
| Unit Tests            | ✅/❌  | N pass, M fail |
| Static Smoke Check    | ✅/⚠️  | N issues |
| Device Config         | ✅/⚠️  | |
| UI Wiring Check       | ✅/⚠️  | |

## Failing Tests
{list each failing test: name, error message, file:line if available}

## Static Issues Found
{list TODO/FIXME/NotImplementedException in changed files}

## Device Config Issues
{list missing or malformed config files}

## UI Wiring Issues
{list missing event handlers, InvokeRequired gaps}

## Top 3 Issues to Fix
{highest-severity items}
```

### Health Score Rubric

| Category | Weight | Score |
|----------|--------|-------|
| Build success | 30% | 100 if clean, 0 if failed |
| Test pass rate | 40% | (pass / total) × 100 |
| Static smoke | 15% | 100 − (5 × issue count, min 0) |
| UI wiring | 15% | 100 − (10 × issue count, min 0) |

---

## Additional Rules

1. **Never fix bugs.** Find and document only. Do not edit files or suggest code fixes in the report. Use `/qa` for the test-fix-verify loop.
2. **Never send device commands.** Do not run device initialization, move sequences, or any command that activates hardware.
3. **Report incrementally.** Write each finding to the report as you find it.
4. **Distinguish test failures from build failures.** A build failure that prevents tests from running is a BLOCKED state, not a test failure.
