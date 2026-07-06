# Contribution 2: Add Data Correction Section to Settings Page

**Contribution Number:** 2  
**Student:** Hui Hwoo  
**Issue:** [npmx.dev #2536 — Download charts: facilitate access to data correction settings](https://github.com/npmx-dev/npmx.dev/issues/2536)  
**Status:** Phase II [In Progress]

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

## Reproduction Process

### Environment Setup

The project uses **pnpm** as its package manager and **Nuxt 3** as the framework.

```bash
pnpm install
pnpm dev
```

Node 20+ is required. The dev server starts at `http://localhost:3000`.

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

The fix does not require changing the chart modal or the `useSettings` composable. It only requires:
1. Adding a new section to `settings.vue` that reads `settings.value.chartFilter`
2. Computing whether any value differs from the defaults
3. Rendering a reset button (disabled when all values are at defaults)
4. Wiring the reset button to restore defaults
5. Adding i18n strings

### Proposed Solution

Add a **"DATA CORRECTION"** section to `settings.vue` that:
- Displays the current values of all four `chartFilter` fields as read-only labels (matching the informational style of other settings sections)
- Shows a "Reset to defaults" button that is disabled when all values are already at their defaults
- On click, sets all four fields back to their defaults via the `useSettings` composable

### Implementation Plan

Using UMPIRE framework:

**Understand:** The settings page needs a new section exposing `chartFilter` state and a conditional reset button. The data lives in `useSettings().settings.value.chartFilter` and is already reactive and persisted.

**Match:** 
- Section structure: mirrors existing sections in `settings.vue` (uppercase heading, `bg-bg-subtle border border-border rounded-lg` container)
- Reset button: follows the `ColumnPicker.vue` pattern — `ButtonBase` component, placed after a `border-t` separator, disabled when already at defaults
- i18n: reuse existing `package.trends.data_correction`, `package.trends.average_window`, etc. for field labels; add a new key for the reset button label

**Plan:**
1. In `app/pages/settings.vue`: add a new `<section>` block after the existing sections with:
   - Section heading: reuse `package.trends.data_correction` i18n key
   - Four rows showing current values: Average window, Smoothing, Prediction, Known anomalies
   - A `computed` `isDefaultChartFilter` that returns `true` when all four values equal the defaults
   - A `resetChartFilter()` function that writes the defaults back into `settings.value.chartFilter`
   - A `ButtonBase` reset button bound to `resetChartFilter`, disabled when `isDefaultChartFilter`
2. In `i18n/locales/en.json`: add `settings.data_correction.reset` = `"Reset to defaults"`
3. Mirror the new i18n key across all other locale files (set to the English string as a fallback)

**Implement:** [Link to branch/commits — to be added]

**Review:**
- [ ] Does the section follow the same visual language as other settings sections?
- [ ] Is the reset button correctly disabled when values are at defaults?
- [ ] Does the reset actually clear localStorage (verified by reopening the page)?
- [ ] Are i18n keys added for all locales?
- [ ] Does `pnpm lint` pass?
- [ ] Does `pnpm test` pass?

**Evaluate:** 
- Manually modify a chartFilter value via the TrendsChart modal, navigate to `/settings`, confirm the reset button is enabled
- Click reset, navigate back to the chart — confirm the slider is back at 0
- Reload the page and verify the reset value persisted to localStorage

---

## Testing Strategy

### Unit Tests

- [ ] `isDefaultChartFilter` returns `true` when all values equal defaults
- [ ] `isDefaultChartFilter` returns `false` when any single value differs
- [ ] `resetChartFilter()` sets all four fields to their default values

### Integration Tests

- [ ] Reset button is disabled on initial page load (all defaults)
- [ ] Reset button becomes enabled after modifying any chartFilter value
- [ ] After clicking reset, `settings.value.chartFilter` equals the default object

### Manual Testing

- Navigate to a package page, modify correction sliders, go to `/settings`, confirm reset button is enabled, click it, go back to the chart and confirm sliders reset to 0/default

---

## Implementation Notes

### Week 1 Progress

Completed Phase I (Discovery) and Phase II (Solution Planning). Explored the codebase to locate `useSettings.ts`, `settings.vue`, and the `TrendsChart` modal. Identified the existing reset button pattern from `ColumnPicker.vue` and confirmed all needed i18n keys. Posted a comment on the issue claiming the work. No code changes yet.

---

## Pull Request

**PR Link:** [To be added]

**PR Description:** [To be drafted]

**Maintainer Feedback:**
- [Pending first submission]

**Status:** Awaiting implementation

---

## Learnings & Reflections

### Technical Skills Gained

[To be filled after implementation]

### Challenges Overcome

[To be filled after implementation]

### What I'd Do Differently Next Time

[To be filled after implementation]

---

## Resources Used

- [Issue #2536](https://github.com/npmx-dev/npmx.dev/issues/2536)
- [Issue #2510 — related context on smoothing discrepancies](https://github.com/npmx-dev/npmx.dev/issues/2510)
- [Nuxt 3 docs — useLocalStorage / Vueuse](https://vueuse.org/core/useLocalStorage/)
- [npmx.dev CONTRIBUTING.md](https://github.com/npmx-dev/npmx.dev/blob/main/CONTRIBUTING.md)
