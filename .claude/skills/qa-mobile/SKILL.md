---
name: qa-mobile
description: Exploratory mobile QA via Appium MCP. Navigate the app like a real user, find bugs, fix them with atomic commits, generate regression tests. Produces health-scored report with before/after evidence. Use when asked to "qa the app", "test the app", "find mobile bugs", or "exploratory QA".
---

# Exploratory Mobile QA

Navigate the app like a real user, discover bugs organically, fix them, and prove the fixes work. Unlike the Proctor (which runs predefined test checklists), qa-mobile is EXPLORATORY — it finds bugs the test plan didn't anticipate.

## When to Use

- "QA the app" / "test the app" / "find mobile bugs" / "exploratory QA"
- After a feature branch is complete, before PR
- When you suspect regressions but don't have a specific test plan
- When the Proctor passes but you want deeper coverage

## Parameters

Parse from user request (or use defaults):

| Parameter | Options | Default |
|-----------|---------|---------|
| **Platform** | `ios-simulator`, `android-emulator`, `mac` | `ios-simulator` |
| **Tier** | `quick` (critical+high only), `standard` (+medium), `exhaustive` (+low/cosmetic) | `standard` |
| **Scope** | `full` (entire app), `diff` (auto on feature branch), or user-specified ("focus on chat screen") | `full` (or `diff` if on feature branch) |

---

## Phase 1: Setup

### 1a. Validate Working Tree

The working tree MUST be clean before starting. Fixes are atomic commits — dirty state makes reverts impossible.

```bash
if [ -n "$(git status --porcelain)" ]; then
  echo "ERROR: Working tree is dirty. Commit or stash before running /qa-mobile."
  exit 1
fi
```

Record the starting commit SHA for the report.

### 1b. Create Appium Session

**iOS Simulator:**
1. `select_platform` → iOS
2. `boot_simulator` (if not already booted)
3. `setup_wda`
4. `install_wda`
5. `create_session` with capabilities

**macOS Desktop:**
1. `create_session` with platformName: mac, automationName: mac2, bundleId

**Android Emulator:**
1. `select_platform` → Android
2. `create_session` with capabilities

Take a **baseline screenshot** after session creation. If session creation fails, report the error and STOP — do not proceed without a live session.

### 1c. Create Output Directory

```bash
mkdir -p .qa-mobile/reports/screenshots
```

### 1d. Start Timer

Record start time for duration tracking in the final report.

---

## Phase 2: Orient

Build a mental map of the app before testing anything. This phase is observation only — no judgments yet.

1. **Capture initial screen**
   - `appium_screenshot` — save as `orient_initial.png`
   - `appium_get_page_source` — understand screen structure

2. **Identify navigation structure**
   - Find tab bar items via `appium_find_element` with accessibility IDs or class names
   - Identify drawer, stack navigator, or other navigation patterns

3. **Visit each main section**
   For each tab bar item or top-level navigation element:
   - `appium_click` the navigation element
   - `appium_screenshot` — save as `orient_{section_name}.png`
   - `appium_get_page_source` — inventory all elements on the screen
   - Note: interactive elements (buttons, inputs), data displays, navigation paths

4. **Detect app framework**
   Look in page source for:
   - `RCT` prefixes → React Native
   - `expo-` identifiers → Expo
   - Native UIKit/SwiftUI patterns → native iOS
   - Jetpack Compose patterns → native Android

5. **Produce app map**
   List all discovered screens, their navigation paths, and element counts. This map drives Phase 4 exploration order.

---

## Phase 3: Diff-Aware Scoping

**Triggered automatically** when on a feature branch with no explicit scope from the user. Skip if scope is `full` or user-specified.

```bash
# Get changed files relative to main
git diff main...HEAD --name-only 2>/dev/null

# Get commit log for context
git log main..HEAD --oneline 2>/dev/null
```

Map changed files to app screens:
- **Screen/component files** → which screens they render (prioritize these)
- **API/service files** → which screens consume those services
- **Style/theme files** → which screens are visually affected
- **Navigation files** → test ALL navigation paths (high regression risk)
- **Shared utilities** → test screens that import them

If no obvious screen mapping can be determined, fall back to full app exploration. Always include the home/landing screen regardless.

---

## Phase 4: Explore

Visit screens systematically based on the app map (Phase 2) and scope (Phase 3). At each screen, run the **Per-Screen Exploration Checklist**.

### Per-Screen Exploration Checklist

#### 1. Visual Scan
- `appium_screenshot` — capture the screen
- Look for:
  - Layout issues (overlapping elements, clipped text, content overflow)
  - Alignment problems (off-grid, uneven spacing)
  - Missing or broken images/icons
  - Font/color inconsistencies
  - Dark mode issues (if applicable — check via system settings)

#### 2. Interactive Elements
- `appium_get_page_source` — discover all tappable elements
- For each button, pressable, link:
  - `appium_find_element` to locate it
  - `appium_click` to activate it
  - Verify response: did it do what its label says?
  - `appium_screenshot` after each interaction
  - Navigate back to continue testing remaining elements

#### 3. Forms
Find text inputs and test:
- **Empty submission** — tap submit without filling any fields. Expected: validation errors.
- **Valid input** — use the RN TextInput workaround:
  1. `appium_find_element` → `appium_click` (focus the input)
  2. `appium_set_value` with the text (sets native value)
  3. `appium_click` the submit button immediately after
- **Edge cases** — very long text (200+ chars), special characters (`<>'"&`), emoji
- If `appium_set_value` fails to populate the field, mark as `blocked:appium-typing` and continue. **Never block the entire session on a typing failure.**

#### 4. Navigation
- **Tab bar:** each tab navigates to the correct screen and back
- **Back button:** returns to the previous screen with correct state
- **Deep links:** `appium_deep_link` if URL scheme is known
- **Swipe gestures:** `appium_swipe` for drawer navigation, stack pop, carousel

#### 5. States
Check for proper handling of:
- **Empty state:** helpful message + action when no data exists (not blank screen)
- **Loading state:** loading indicator during async operations (not frozen UI)
- **Error state:** meaningful error message when things go wrong (not silent failure)

#### 6. Error Detection
- `appium_get_page_source` — scan for error elements (error messages, crash screens, fallback UI, "Something went wrong" text)
- Note any red/warning elements visible in screenshots

#### 7. Orientation
- `appium_set_orientation("LANDSCAPE")` → `appium_screenshot`
- Verify layout adapts (no clipping, no overlap, content reflows)
- `appium_set_orientation("PORTRAIT")` → restore

#### 8. Scroll
- `appium_scroll` to find off-screen content
- Verify:
  - FlatList/ScrollView renders correctly
  - Content isn't cut off at the bottom
  - Pull-to-refresh works (if applicable)
  - No duplicate items after scroll

### Depth Judgment

- **Core features** (home, chat, main workflow, primary actions) → exhaustive exploration (all 8 checklist items)
- **Secondary screens** (settings, profile, about, help) → quick visual scan + basic interaction test (items 1-2 only)
- **Tertiary screens** (legal, terms of service, licenses) → visual scan only (item 1)

### Quick Tier Override

When tier is `quick`: only visit main tab screens + test top 3 interactive elements each. Skip form testing, orientation, and scroll checks.

---

## Phase 5: Document Issues

Document each issue **IMMEDIATELY when found** — do not batch them until the end.

### Interactive Bugs (broken flows, dead buttons, form failures)
1. `appium_screenshot` — before state (the setup)
2. Perform the action that triggers the bug
3. `appium_screenshot` — after state (showing the problem)
4. Write repro steps with screenshot file references

### Static Bugs (layout issues, missing images, text problems)
1. `appium_screenshot` showing the problem
2. Describe what's wrong with specific element references

### Issue Format

```markdown
### ISSUE-NNN: {Short title}

| Field | Value |
|-------|-------|
| **Severity** | critical / high / medium / low |
| **Category** | navigation / visual / functional / ux / performance / accessibility / forms / stability |
| **Screen** | {screen name from app map} |

**Description:** {What is wrong. Expected behavior vs actual behavior.}

**Repro Steps:**
1. Navigate to {screen} via {path}
2. {Action — e.g., "Tap the Submit button without filling any fields"}
3. **Observe:** {what goes wrong — e.g., "App crashes to home screen instead of showing validation errors"}

**Screenshots:** before_{NNN}.png, after_{NNN}.png
```

### Severity Definitions

| Severity | Definition | Examples |
|----------|------------|----------|
| **critical** | Blocks a core workflow, causes data loss, or crashes the app | Form submit crashes, checkout flow broken, data deleted without confirmation |
| **high** | Major feature broken or unusable, no workaround | Search returns wrong results, file upload silently fails, auth redirect loop |
| **medium** | Feature works but with noticeable problems, workaround exists | Slow load (>5s), missing validation but submit works, layout broken in landscape only |
| **low** | Minor cosmetic or polish issue | Typo, 1px alignment, hover state inconsistent, minor spacing |

---

## Phase 6: Health Score

Compute a health score across 8 categories. Each category starts at 100 and takes deductions per finding. The final score is a weighted average.

### Deduction Table

| Severity | Points Deducted |
|----------|----------------|
| Critical | -25 |
| High | -15 |
| Medium | -8 |
| Low | -3 |

### Category Weights

| Category | Weight | What It Covers |
|----------|--------|----------------|
| Navigation (15%) | Tab bar, back button, deep links, swipe gestures | All navigation paths work correctly |
| Visual (10%) | Layout, alignment, dark mode, images, icons | No visual defects |
| Functional (20%) | Core features, data persistence, state management | Features do what they claim |
| UX (15%) | Loading indicators, empty states, error messages, flow completion | User can understand and complete tasks |
| Performance (10%) | Screen transitions, scroll smoothness, load times | App feels responsive |
| Accessibility (15%) | Touch targets >= 44px, labels present, VoiceOver-compatible | App is usable by all users |
| Forms & Input (10%) | Text input works, validation present, keyboard avoidance | Forms submit correctly |
| Stability (5%) | No crashes, session intact, background/foreground survives | App doesn't crash |

### Computation

```
category_score = max(0, 100 - sum_of_deductions)
final_score = sum(category_score * weight for each category)
```

Floor at 0 per category — negative scores aren't meaningful.

Record this as the **baseline health score**.

---

## Phase 7: Triage

Sort all issues by severity (critical first, then high, medium, low).

Select which to fix based on the tier:
- **Quick:** critical + high only
- **Standard:** critical + high + medium (default)
- **Exhaustive:** all severities including low/cosmetic

Mark these as **deferred** regardless of tier:
- Appium infrastructure limitations (e.g., `blocked:appium-typing`)
- Issues requiring backend changes or external service fixes
- Issues requiring design decisions (ambiguous expected behavior)

---

## Phase 8: Fix Loop

For each fixable issue, in severity order:

### 8a. Locate Source

- Grep for component names, error messages, accessibility IDs found during exploration
- Glob for file patterns matching the affected screen name
- Read the source code to understand the root cause

### 8b. Fix

Apply the **minimal change** that resolves the issue:
- Smallest diff possible
- Do NOT refactor surrounding code
- Do NOT add features
- Do NOT "improve" unrelated code
- If the fix would touch more than 3 files, pause and reconsider

### 8c. Commit

```bash
git add <only-changed-files>
git commit -m "fix(qa-mobile): ISSUE-NNN — short description"
git push
```

**One commit per fix. Never bundle multiple fixes.**

### 8d. Re-test via Appium

The app may need to be relaunched to pick up code changes:
```
appium_terminate_app with bundleId
appium_activate_app with bundleId
```

Then:
1. Navigate back to the affected screen
2. `appium_screenshot` — capture the current state (should reflect the fix)
3. Perform the action that previously triggered the bug
4. `appium_screenshot` — capture the result (should show correct behavior)
5. Check for regressions on the same screen and adjacent screens

### 8e. Classify the Fix

- **verified**: re-test confirms the fix works, no regressions detected
- **best-effort**: fix applied but couldn't fully verify (needs backend, auth state, real data, etc.)
- **reverted**: regression detected → immediately run `git revert HEAD` → mark issue as "deferred"

### 8e.5. Regression Test

**Skip if:** the fix is not "verified", the issue is purely visual/layout, or no test framework is detected in the project.

1. **Learn conventions** — read 2-3 existing test files closest to the fix (same directory or nearest `__tests__/` folder)
2. **Trace the codepath** — what input/state triggered the bug, what broke, where in the code
3. **Write the regression test:**
   - Set up the precondition that triggered the bug
   - Perform the action
   - Assert the correct behavior (not just "it renders" — assert the specific fix)
   - Include attribution comment:
     ```
     // Regression: ISSUE-NNN — {what broke}
     // Found by /qa-mobile on {YYYY-MM-DD}
     ```
4. **Run only the new test file** — do not run the full suite
5. **If it passes** → commit:
   ```bash
   git add <test-file>
   git commit -m "test(qa-mobile): regression test for ISSUE-NNN"
   git push
   ```
6. **If it fails** → attempt one fix. Still fails → delete the test file, mark as deferred.

### 8f. Self-Regulation (check every 5 fixes)

Compute WTF-likelihood — a heuristic for "am I making things worse?"

```
WTF-LIKELIHOOD:
  Start at 0%
  Each revert:                 +15%
  Each fix touching >3 files:  +5%
  After fix 15:                +1% per additional fix
  All remaining are Low sev:   +10%
  Touching unrelated files:    +20%
```

**If WTF > 20%:** STOP immediately. Commit current state, push, and report what's done and what triggered the stop. Do not continue fixing.

**Hard cap:** 50 fixes maximum. After 50, stop regardless of remaining issues.

---

## Phase 9: Final QA

After all fixes are applied (or the fix loop stopped):

1. **Re-explore all affected screens** via Appium
   - Visit every screen that had a fix applied
   - `appium_screenshot` each one
   - Verify fixes are still in place
   - Check for new regressions introduced by the fix set

2. **Compute final health score** using the same methodology as Phase 6

3. **Compare to baseline:**
   - Score improved → report the delta
   - Score unchanged → note that fixes didn't move the needle (possible if issues were deferred)
   - **Score WORSE than baseline → WARN prominently** — something regressed

---

## Phase 10: Report

Write the full report to `.qa-mobile/reports/qa-mobile-{project}-{YYYY-MM-DD}.md`

```markdown
# QA-Mobile Report: {PROJECT_NAME}

| Field | Value |
|-------|-------|
| **Date** | {YYYY-MM-DD} |
| **Platform** | {ios-simulator / android-emulator / mac} |
| **Branch** | {branch name} |
| **Commit (start)** | {starting SHA} |
| **Commit (end)** | {ending SHA} |
| **Tier** | Quick / Standard / Exhaustive |
| **Scope** | {diff description / user-specified scope / "Full app"} |
| **Duration** | {HH:MM:SS} |
| **Screens visited** | {count} |
| **Screenshots taken** | {count} |

## Health Score: {FINAL_SCORE}/100

| Category | Before | After | Delta |
|----------|--------|-------|-------|
| Navigation | {score} | {score} | {+/-N} |
| Visual | {score} | {score} | {+/-N} |
| Functional | {score} | {score} | {+/-N} |
| UX | {score} | {score} | {+/-N} |
| Performance | {score} | {score} | {+/-N} |
| Accessibility | {score} | {score} | {+/-N} |
| Forms & Input | {score} | {score} | {+/-N} |
| Stability | {score} | {score} | {+/-N} |
| **Weighted Total** | **{before}** | **{after}** | **{+/-N}** |

## Top 3 Things to Fix

> The most impactful issues — either unfixed or deferred. Address these first.

1. **ISSUE-NNN: {title}** — {one-line description with severity}
2. **ISSUE-NNN: {title}** — {one-line description with severity}
3. **ISSUE-NNN: {title}** — {one-line description with severity}

## Summary

| Severity | Found | Fixed | Deferred |
|----------|-------|-------|----------|
| Critical | N | N | N |
| High | N | N | N |
| Medium | N | N | N |
| Low | N | N | N |
| **Total** | **N** | **N** | **N** |

## Issues

{All per-issue details from Phase 5, in severity order, with screenshot references}

## Fixes Applied

| Issue | Severity | Status | Commit | Files Changed |
|-------|----------|--------|--------|---------------|
| ISSUE-NNN | critical | verified | {short SHA} | {file list} |
| ISSUE-NNN | high | best-effort | {short SHA} | {file list} |
| ISSUE-NNN | medium | reverted | {revert SHA} | {file list} |
| ISSUE-NNN | high | deferred | — | — |

## Regression Tests

| Issue | Test File | Status |
|-------|-----------|--------|
| ISSUE-NNN | {relative path} | committed |
| ISSUE-NNN | — | deferred (reason) |

## Ship Readiness

| Metric | Value |
|--------|-------|
| Health score | {before} → {after} ({+/-N}) |
| Issues found | {total} |
| Fixes applied | {count} (verified: {V}, best-effort: {B}, reverted: {R}) |
| Deferred | {count} |
| Regression tests | {committed count} / {eligible count} |
| WTF-likelihood | {final %}  |

**PR Summary:** "QA-Mobile found {N} issues, fixed {M}, health score {before} → {after}."
```

---

## Phase 11: Cleanup

1. **End Appium session:** `delete_session` — always, even if earlier phases failed
2. **Deferred issues:** if deferred bugs exist and a `TODOS.md` file exists in the repo, append them with severity and category:
   ```markdown
   - [ ] ISSUE-NNN ({severity}, {category}): {title} — found by /qa-mobile on {date}
   ```
3. **Report location:** confirm the report path in output so the user can find it

---

## Recovery Patterns

### App Crash During Test
1. `appium_screenshot` (may fail — that's OK)
2. Record the crash as a **Stability** finding with **critical** severity
3. `appium_activate_app` with bundle ID to relaunch
4. Wait 5 seconds for the app to stabilize
5. Continue exploration from a known screen

### Appium Session Lost
1. Record as a **Stability** finding
2. Recreate the session (run Phase 1b setup again)
3. Navigate back to where you were and continue

### Stuck on Unexpected Screen
1. `appium_screenshot` for evidence
2. Try back button or home gesture
3. If still stuck: `appium_terminate_app` → `appium_activate_app`
4. Record as a **UX** finding (dead end, confusing navigation)

### App Won't Relaunch After Fix
1. `appium_terminate_app` with bundleId
2. Wait 3 seconds
3. `appium_activate_app` with bundleId
4. If still fails, try full session teardown and recreation (Phase 1b)
5. If THAT fails, record as **Stability/critical** and stop the fix loop

---

## Important Rules

1. **Screenshots are mandatory.** Before and after every test action. This is the evidence. A bug without a screenshot is a rumor.
2. **Don't guess — observe.** Use `appium_find_element` and `appium_get_text` to verify element state. Never assume based on what "should" happen.
3. **Test like a user.** Navigate the app naturally during exploration (Phase 4). Don't read source code to decide what to test — discover it by using the app. Source code reading happens in Phase 8 when fixing.
4. **One commit per fix.** Never bundle multiple fixes into one commit. This makes reverts surgical.
5. **Revert on regression.** If a fix breaks something else, `git revert HEAD` immediately. No exceptions. Mark the issue as deferred.
6. **RN TextInput workaround.** `appium_set_value` doesn't fire `onChangeText` — use tap to focus → `appium_set_value` → tap submit immediately. Prefer quick-reply buttons over typing when possible.
7. **Never block on typing failures.** Record as `blocked:appium-typing` and continue. Typing issues are infrastructure problems, not app bugs.
8. **Depth over breadth.** 5-10 well-documented issues with screenshots and repro steps are worth more than 20 vague descriptions. Quality of evidence matters.
9. **Self-regulate.** Follow the WTF-likelihood heuristic. When in doubt, stop. A clean partial run is better than a messy complete one.
10. **Clean up.** Always `delete_session` when done, even after errors. Leaked Appium sessions cause problems for subsequent runs.
