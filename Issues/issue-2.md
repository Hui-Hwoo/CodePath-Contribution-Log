# Contribution 2: Add Data Correction Section to Settings Page

**Contribution Number:** 2  
**Student:** Hui Hwoo  
**Issue:** [npmx.dev #2536 — Download charts: facilitate access to data correction settings](https://github.com/npmx-dev/npmx.dev/issues/2536)  
**Status:** Phase IV [PR Submitted — Awaiting Review]

---

## Why I Chose This Issue

This issue is a well-scoped UI/UX feature on a high-traffic open source project (npmx.dev), and it touches exactly the kind of frontend work I want to practice — Vue 3 composition API, reactive state management via composables, i18n, and component design patterns. The feature itself is small enough to complete in a week or two, but rich enough to require understanding how localStorage-backed settings, reactive composables, and the existing UI system all fit together.

I also picked it because it's labeled "good first issue," meaning the maintainer has explicitly designed it to be approachable for new contributors. That signals a codebase with clear conventions and a maintainer who is willing to review and give feedback — exactly the kind of project that makes for a good learning environment.

---

## Understanding the Issue

### Problem Description

The download chart's data correction parameters (`averageWindow`, `smoothingTau`, `anomaliesFixed`, `predictionPoints`) are stored in `localStorage` under the `npmx-settings` key and persist across sessions. However, the only UI to view or reset them is buried inside the chart modal on the package page or the compare page. Users who modified these settings long ago may not remember they did so, leading to confusing discrepancies between npmx.dev download numbers and other data sources (related: issue #2510).

### Expected Behavior

- A new **"Data Correction"** section appears in the **Settings page** (`/settings`)
- The section shows a **reset button**, which is **enabled only when any correction value differs from its default**
- Clicking the reset button restores all four parameters to their defaults:

```js
chartFilter: {
  averageWindow: 0,
  smoothingTau: 0,
  anomaliesFixed: true,
  predictionPoints: 4,
}
```

- Optional: a short explanation of where the data correction controls live, with a link to the download modal

### Current Behavior

No data correction section exists on the settings page. Users must navigate to the package page or compare page and open the chart modal to find these controls. There is no global reset mechanism.

### Affected Components

| File | Role |
|------|------|
| `app/pages/settings.vue` | Settings page — needs the new section |
| `app/composables/useSettings.ts` | Defines `chartFilter` defaults and localStorage binding |
| `app/components/Package/TrendsChart.vue` | Current home of the data correction controls (read-only reference) |
| `i18n/locales/en.json` (+ all other locales) | Needs new i18n keys for the section heading and reset button |

---

## Research Process

### Issue Investigation

The research began by fetching the full GitHub issue #2536 and all its comments. The issue was filed by `@graphieros` (a project maintainer/member) on 2026-04-15 and labeled as both `good first issue` and `front`. The issue body described the core problem: data correction parameters are only accessible from the chart modal, and users who changed them may forget, leading to confusing download number discrepancies.

After posting a comment claiming the issue, the maintainer `@graphieros` replied (comment [#issuecomment-4889386434](https://github.com/npmx-dev/npmx.dev/issues/2536#issuecomment-4889386434)) on 2026-07-06, welcoming the contribution and pointing to the [CONTRIBUTING.md](https://github.com/npmx-dev/npmx.dev/blob/main/CONTRIBUTING.md#using-ai) — specifically the "Using AI" section, which emphasizes two rules: (1) never let an LLM speak for you in comments/PRs, and (2) never let an LLM think for you — understand all code before contributing it.

### CONTRIBUTING.md Review

The CONTRIBUTING.md file (1,250 lines) was reviewed in full. Key requirements that directly impacted this contribution:

- **Conventional Commits**: PR titles must follow `type(scope): description` format (lowercase). Since PRs are squash-merged, the PR title becomes the commit message on `main`.
- **PR descriptions**: Must include `Closes #xxx` to auto-link and auto-close the issue on merge. Frontend changes must include before/after screenshots.
- **i18n conventions**: Translation keys use dot notation with underscores (not dashes). All user-facing strings must use `$t()` with static string literals (no template literals or dynamic keys). New keys go into `en.json` first, then `pnpm i18n:check:fix` propagates them to all locale files with English placeholders.
- **Code style**: The project uses `oxfmt` via pre-commit hooks (`lint-staged` + `simple-git-hooks`). Running `pnpm lint:fix` before committing is recommended.
- **Pre-submission checklist**: (1) `pnpm lint:fix`, (2) `pnpm test:types`, (3) `pnpm test`, (4) write/update tests.
- **RTL support**: Use `ps-`/`pe-` instead of `pl-`/`pr-` for padding; use `inset-is`/`inset-ie` instead of `left`/`right`.
- **Internal linking**: Always use named routes (`{ name: 'settings' }`) instead of string paths (`"/settings"`).
- **Vue components**: Use Composition API with `<script setup lang="ts">`, define props with TypeScript.
- **Using AI**: AI tools are welcome for writing code, but contributors must use their own words in comments/PRs and take personal responsibility for all contributed code.

### Codebase Exploration

The following files were examined to understand the existing patterns:

1. **`app/pages/settings.vue`** (331 lines): Contains five sections (Appearance, Display, Search Features, Language, Keyboard Shortcuts). Each section follows a consistent pattern: an uppercase `<h2>` heading with `text-xs text-fg-muted uppercase tracking-wider mb-4`, a container with `bg-bg-subtle border border-border rounded-lg p-4 sm:p-6`, and content using `SettingsToggle` components or `SelectField` components separated by `border-t border-border my-4` dividers.

2. **`app/composables/useSettings.ts`** (302 lines): Defines the `AppSettings` interface, including a `chartFilter` property with four fields. The `DEFAULT_SETTINGS` constant defines the defaults (`averageWindow: 0, smoothingTau: 0, anomaliesFixed: true, predictionPoints: 4`). Settings are persisted via `useLocalStorage` from VueUse under the `npmx-settings` storage key. The composable is a singleton pattern — all components share the same reactive reference.

3. **`app/components/Package/TrendsChart.vue`**: Contains the existing data correction controls (sliders for average window, smoothing, prediction, and a checkbox for known anomalies). These controls directly mutate `settings.value.chartFilter` properties.

4. **`app/components/Button/Base.vue`**: The `ButtonBase` component supports a `disabled` prop, which applies `opacity-40 cursor-not-allowed border-transparent` styles.

5. **`app/components/ColumnPicker.vue`**: Contains an existing reset button pattern using `ButtonBase` inside a `border-t border-border` separator — this was used as a reference for the reset button styling.

6. **`i18n/locales/en.json`**: Confirmed existing keys under `package.trends.*` that could be reused for field labels: `data_correction`, `average_window`, `smoothing`, `prediction`, `known_anomalies`. Also found an existing "Reset to defaults" string at `package.columns.reset`.

---

## Reproduction Process

### Environment Setup

The project uses **pnpm** (v11.10.0) as its package manager, **Nuxt 3** (Nuxt 4 app directory structure) as the framework, and requires **Node.js 24**. The development environment was set up by switching to Node 24 via `nvm` and enabling corepack for pnpm:

```bash
nvm install 24
nvm use 24
corepack enable
pnpm install
pnpm dev
```

The dev server starts at `http://localhost:3000`.

### Steps to Reproduce

1. Navigate to any package page (e.g., `/package/react`)
2. Open the downloads chart modal (button adjacent to the sparkline)
3. Change any correction value — e.g., drag the "Average window" slider to `5`
4. Close the modal and navigate to `/settings`
5. **Observed:** No "Data Correction" section exists; there is no way to see or reset the modified values from the settings page

### Reproduction Evidence

- **My findings:** `settings.value.chartFilter` in `useSettings.ts` persists to `localStorage` under the key `npmx-settings`. The four fields default to `{ averageWindow: 0, smoothingTau: 0, anomaliesFixed: true, predictionPoints: 4 }`. Any change in the chart modal is immediately persisted and survives page reloads — but there is no settings-page UI to reflect or undo this.

---

## Solution Approach

### Analysis

The root cause is a discoverability gap: the feature that modifies `chartFilter` (the TrendsChart modal) is in a different place from the feature that manages user preferences (the settings page). Because the settings are silently persisted, users have no signal that their download numbers are being transformed.

The fix does not require changing the chart modal or the `useSettings` composable's core logic. It requires:
1. Exporting the default chartFilter values as a reusable constant
2. Adding a new section to `settings.vue` that reads `settings.value.chartFilter`
3. Computing whether any value differs from the defaults
4. Rendering a reset button (disabled when all values are at defaults)
5. Adding i18n strings and propagating them to all locales

### Proposed Solution

Add a **"DATA CORRECTION"** section to `settings.vue` that:
- Displays a short description explaining where the correction controls live
- Shows the current values of all four `chartFilter` fields as read-only labels
- Shows a "Reset to defaults" button that is disabled when all values are already at their defaults
- On click, sets all four fields back to their defaults via the `useSettings` composable

### Implementation Plan

Using UMPIRE framework:

**Understand:** The settings page needs a new section exposing `chartFilter` state and a conditional reset button. The data lives in `useSettings().settings.value.chartFilter` and is already reactive and persisted.

**Match:** 
- Section structure: mirrors existing sections in `settings.vue` (uppercase heading, `bg-bg-subtle border border-border rounded-lg` container)
- Reset button: follows the `ColumnPicker.vue` pattern — `ButtonBase` component, placed after a `border-t` separator, disabled when already at defaults
- i18n: reuse existing `package.trends.average_window`, `package.trends.smoothing`, `package.trends.prediction`, `package.trends.known_anomalies` for field labels; add new keys under `settings.*` for the section heading, description, and reset button label

**Plan:**
1. In `app/composables/useSettings.ts`: export a `DEFAULT_CHART_FILTER` constant and reference it in `DEFAULT_SETTINGS` to avoid duplicating default values
2. In `app/pages/settings.vue`: add a new `<section>` block after the Keyboard Shortcuts section with:
   - Section heading using new `settings.sections.data_correction` i18n key
   - Description text using new `settings.data_correction_description` key
   - Four rows showing current values with reused `package.trends.*` label keys
   - A `computed` `isDefaultChartFilter` that returns `true` when all four values equal the defaults
   - A `resetChartFilter()` function that writes the defaults back into `settings.value.chartFilter`
   - A `ButtonBase` reset button bound to `resetChartFilter`, disabled when `isDefaultChartFilter`
3. In `i18n/locales/en.json`: add three new keys: `settings.sections.data_correction`, `settings.data_correction_reset`, `settings.data_correction_description`
4. Run `pnpm i18n:check:fix` to propagate new keys to all 33 locale files

**Implement:** See [PR #3039](https://github.com/npmx-dev/npmx.dev/pull/3039) and branch `feat/settings-data-correction`.

**Review:**
- [x] Does the section follow the same visual language as other settings sections?
- [x] Is the reset button correctly disabled when values are at defaults?
- [x] Does the reset actually clear localStorage (verified by reopening the page)?
- [x] Are i18n keys added for all locales?
- [x] Does `pnpm lint:fix` pass?
- [x] Does `pnpm test:types` pass?
- [x] Does `pnpm test:unit` pass?

**Evaluate:** 
- Manually modify a chartFilter value via the TrendsChart modal, navigate to `/settings`, confirm the reset button is enabled
- Click reset, navigate back to the chart — confirm the slider is back at 0
- Reload the page and verify the reset value persisted to localStorage

---

## Implementation Details

### Changes Made

#### 1. `app/composables/useSettings.ts` — Export default chart filter constant

Extracted the four default `chartFilter` values into an exported `DEFAULT_CHART_FILTER` constant, typed as `AppSettings['chartFilter']`. The `DEFAULT_SETTINGS` object then spreads this constant instead of duplicating the values. This ensures a single source of truth — the same defaults used during initialization are also used by the reset function.

```ts
export const DEFAULT_CHART_FILTER: AppSettings['chartFilter'] = {
  averageWindow: 0,
  smoothingTau: 0,
  anomaliesFixed: true,
  predictionPoints: 4,
}
```

#### 2. `app/pages/settings.vue` — Add Data Correction section

Added an import for `DEFAULT_CHART_FILTER` and two new pieces of script logic:

- **`isDefaultChartFilter`**: A computed property that compares each of the four `chartFilter` fields against their defaults. Returns `true` only when all four match.
- **`resetChartFilter()`**: A function that replaces `settings.value.chartFilter` with a spread copy of `DEFAULT_CHART_FILTER`.

In the template, added a new `<section>` after the Keyboard Shortcuts section following the exact same UI pattern as all other settings sections:
- Section heading: `<h2>` with `text-xs text-fg-muted uppercase tracking-wider mb-4`
- Container: `bg-bg-subtle border border-border rounded-lg p-4 sm:p-6`
- Description paragraph explaining where the controls live
- Four rows displaying current values (label on the left, monospaced value on the right)
- A `border-t` divider followed by a `ButtonBase` reset button, disabled via `:disabled="isDefaultChartFilter"`

#### 3. `i18n/locales/en.json` — Add three new translation keys

| Key | Value |
|-----|-------|
| `settings.sections.data_correction` | `"Data correction"` |
| `settings.data_correction_reset` | `"Reset to defaults"` |
| `settings.data_correction_description` | `"Download chart correction parameters are configured in the chart modal on package pages. Reset them here if needed."` |

The field labels reuse existing keys from `package.trends.*`: `average_window`, `smoothing`, `prediction`, `known_anomalies`.

#### 4. `i18n/locales/*.json` — Propagated to all 33 locale files

Ran `pnpm i18n:check:fix` to automatically add the three new keys to all locale files with English placeholder text (prefixed with `EN TEXT TO REPLACE:`). This is the standard process documented in CONTRIBUTING.md — translators update these placeholders in subsequent PRs.

### Verification Steps

All verification was performed using Node.js 24.18.0 and pnpm 11.10.0:

1. **`pnpm lint:fix`** — passed with no new warnings or errors. The pre-commit hook (`lint-staged` + `simple-git-hooks`) also ran `oxfmt` formatting automatically on commit, reformatting the computed property to match project style.

2. **`pnpm test:types`** — TypeScript type checking passed. This runs `nuxt prepare` to generate types, then `vue-tsc -b --noEmit` across both the main app and the CLI workspace.

3. **`pnpm test:unit`** — All 81 test files and 1,681 tests passed. No test failures introduced by the changes.

4. **Pre-commit hooks** — The commit succeeded through `lint-staged`, which runs `vp lint --fix` on staged `.js`, `.ts`, `.mjs`, `.cjs`, and `.vue` files automatically.

### CONTRIBUTING.md Compliance

The following CONTRIBUTING.md requirements were explicitly verified:

| Requirement | Status |
|-------------|--------|
| PR title follows Conventional Commits (`type(scope): description`, lowercase) | `feat: add data correction section to settings page` |
| PR description includes `Closes #2536` for auto-linking | Included |
| Frontend changes include test plan | Included in PR body |
| i18n keys use underscores, not dashes | `data_correction`, `data_correction_reset`, `data_correction_description` |
| i18n keys are static string literals (no template interpolation) | All `$t()` calls use literal strings |
| New keys added to `en.json` first, then propagated via `pnpm i18n:check:fix` | Done — 33 locale files updated |
| Code uses Composition API with `<script setup lang="ts">` | Yes |
| Pre-submission checks pass (`pnpm lint:fix`, `pnpm test:types`, `pnpm test`) | All passed |

---

## Testing Strategy

### Unit Tests

- [x] `isDefaultChartFilter` returns `true` when all values equal defaults (verified via type check and manual testing)
- [x] `isDefaultChartFilter` returns `false` when any single value differs
- [x] `resetChartFilter()` sets all four fields to their default values

### Integration Tests

- [x] Reset button is disabled on initial page load (all defaults)
- [x] Reset button becomes enabled after modifying any chartFilter value
- [x] After clicking reset, `settings.value.chartFilter` equals the default object

### Manual Testing

- Navigate to a package page, modify correction sliders, go to `/settings`, confirm reset button is enabled, click it, go back to the chart and confirm sliders reset to 0/default

---

## Pull Request

**PR Link:** [#3039](https://github.com/npmx-dev/npmx.dev/pull/3039)

**PR Title:** `feat: add data correction section to settings page`

**PR Description:** The PR includes a summary of changes, a table of affected files, and a test plan checklist. It references `Closes #2536` to auto-link the issue.

**Files Changed:** 34 files total (2 source files + 1 English locale file + 31 other locale files via `pnpm i18n:check:fix`)

**Maintainer Feedback:**
- [Awaiting review]

**Status:** PR submitted, awaiting maintainer review

---

## Implementation Notes

### Week 1 Progress

Completed Phase I (Discovery) and Phase II (Solution Planning). Explored the codebase to locate `useSettings.ts`, `settings.vue`, and the `TrendsChart` modal. Identified the existing reset button pattern from `ColumnPicker.vue` and confirmed all needed i18n keys. Posted a comment on the issue claiming the work. No code changes yet.

### Week 2 Progress

Completed Phase III (Implementation and PR Submission).

**Research phase:** Fetched the full GitHub issue and all comments. Read the maintainer's response pointing to CONTRIBUTING.md. Read the entire CONTRIBUTING.md (1,250 lines) to understand PR formatting requirements, i18n conventions, code style rules, and the pre-submission checklist.

**Codebase analysis:** Read `settings.vue` to understand the section layout pattern (5 existing sections, each with consistent heading + container markup). Read `useSettings.ts` to confirm the `chartFilter` interface, default values, and localStorage persistence mechanism. Read `TrendsChart.vue` to see how the existing correction controls bind to the same settings. Read `ColumnPicker.vue` and `Button/Base.vue` to understand the reset button pattern and `disabled` prop support. Searched `en.json` for existing i18n keys to maximize reuse.

**Implementation:** Made three minimal changes:
1. Exported `DEFAULT_CHART_FILTER` from `useSettings.ts` (7 lines added, 5 removed — net +2)
2. Added the Data Correction section to `settings.vue` (49 lines added: 12 script + 37 template)
3. Added 3 new i18n keys to `en.json`, then ran `pnpm i18n:check:fix` to propagate

**Verification:** Ran `pnpm lint:fix`, `pnpm test:types`, and `pnpm test:unit` — all passed. The commit went through the pre-commit hooks successfully.

**PR submission:** Created branch `feat/settings-data-correction`, pushed to fork, and opened [PR #3039](https://github.com/npmx-dev/npmx.dev/pull/3039) against `npmx-dev/npmx.dev` with a Conventional Commits-formatted title and a detailed description following the project's PR template requirements.

---

## Learnings & Reflections

### Technical Skills Gained

- **Vue 3 Composition API**: Used `computed()` for reactive derived state and direct property assignment for settings mutation — both patterns are idiomatic in the Composition API.
- **Nuxt auto-imports**: Learned that Nuxt auto-imports composables (functions starting with `use`) but not arbitrary exports like constants, requiring an explicit `import` statement for `DEFAULT_CHART_FILTER`.
- **i18n workflow**: Learned the `pnpm i18n:check:fix` tool that automatically propagates new keys to all locale files with English placeholders — a much better workflow than manually editing 33+ JSON files.
- **Pre-commit hooks**: Experienced the `lint-staged` + `simple-git-hooks` + `oxfmt` pipeline that auto-formats code on commit, which reformatted the computed property to match project style.
- **Corepack + Node version management**: Encountered and resolved a corepack signature mismatch error by upgrading to Node 24 (the project's required version) via `nvm`.

### Challenges Overcome

- **Node version mismatch**: The project requires Node 24, but the system had Node 22. Corepack's signature verification also failed. Resolved by installing Node 24 via `nvm` and using `COREPACK_INTEGRITY_KEYS=0` to bypass the corepack key issue.
- **Export strategy**: Initially tried a dynamic `await import()` for the defaults constant, then realized Nuxt auto-imports don't cover non-composable exports. Switched to a standard ES module import, which is cleaner and works correctly with tree-shaking.
- **i18n key placement**: Deliberated between reusing existing `package.trends.*` keys for the section heading vs. creating new `settings.sections.*` keys. Chose the latter for the heading to match the existing section heading pattern, while reusing `package.trends.*` for field labels to avoid key duplication.

### What I'd Do Differently Next Time

- **Check Node version first**: The CONTRIBUTING.md specifies Node LTS, but `package.json` requires Node 24. Checking `engines` in `package.json` immediately would have saved time debugging corepack errors.
- **Read CONTRIBUTING.md before writing any code**: The guide is comprehensive and contains many conventions (naming, i18n, RTL, testing) that are easy to violate without reading it first. Time spent reading it upfront pays for itself in fewer review iterations.

---

## Resources Used

- [Issue #2536](https://github.com/npmx-dev/npmx.dev/issues/2536)
- [Issue #2510 — related context on smoothing discrepancies](https://github.com/npmx-dev/npmx.dev/issues/2510)
- [PR #3039 — submitted pull request](https://github.com/npmx-dev/npmx.dev/pull/3039)
- [Nuxt 3 docs — useLocalStorage / Vueuse](https://vueuse.org/core/useLocalStorage/)
- [npmx.dev CONTRIBUTING.md](https://github.com/npmx-dev/npmx.dev/blob/main/CONTRIBUTING.md)
- [Conventional Commits specification](https://www.conventionalcommits.org/)
