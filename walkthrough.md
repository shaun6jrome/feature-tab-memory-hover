# Walkthrough: Tab Memory Hover Card Feature

We have successfully researched, designed, and implemented a tab memory usage indicator for Zen Browser's hover cards. The changes are structured as standard unified patches compatible with Zen Browser's build-system overlay pipeline.

---

## 1. Documentation & Investigation

We reverse-engineered both Firefox's and Zen's components, publishing the following research reports:
1. **[Hover Card Architecture](file:///Users/jerome/Documents/MY PROJECTS/feature:tab-memory-hover/docs/hover-card-architecture.md)** — Analysis of Firefox's `TabHoverPanelSet` and `TabPanel` classes.
2. **[Firefox Memory Architecture](file:///Users/jerome/Documents/MY PROJECTS/feature:tab-memory-hover/docs/memory-architecture.md)** — Analysis of Fission processes, `ChromeUtils.requestProcInfo()` WebIDL API, and mapping metrics to PIDs.
3. **[Feasibility Report](file:///Users/jerome/Documents/MY PROJECTS/feature:tab-memory-hover/docs/feasibility-report.md)** — Detailed assessment confirming implementation feasibility.
4. **[Architecture Design](file:///Users/jerome/Documents/MY PROJECTS/feature:tab-memory-hover/docs/architecture-design.md)** — Specification of data flow, DOM insertion, caching, and refresh strategies.

---

## 2. Changes Made (Patches)

We created two patch files representing the implementation phases:

### JS Behavior Patch
#### [NEW] [tab-hover-preview-mjs.patch](file:///Users/jerome/Documents/MY PROJECTS/feature:tab-memory-hover/src/browser/components/tabbrowser/content/tab-hover-preview-mjs.patch)
- **DOM Injection:** Dynamically appends a `.tab-preview-memory-container` containing a disk icon and a value placeholder into the text container inside the `TabPanel` constructor.
- **Caching:** Instantiates memory cache variables `_procInfoCache` and `_procInfoCacheTime` with a `1500ms` TTL to prevent excessive OS process requests.
- **Data Retrieval:** In `async #updatePreview()`, calls `ChromeUtils.requestProcInfo()` and correlates the tab's process IDs to sum memory `residentSetSize` (RSS).
- **Format and Render:** Formats bytes to `MB` / `GB`, handles shared process indication (`(shared)` label), and resolves discarded tabs cleanly by displaying a `Sleeping` status.

### CSS Theme Patch
#### [NEW] [tab-hover-preview-css.patch](file:///Users/jerome/Documents/MY PROJECTS/feature:tab-memory-hover/src/browser/themes/shared/tabbrowser/tab-hover-preview-css.patch)
- Injects CSS rules targeting `.tab-preview-memory-container` to configure flex layouts, gaps, margins, text colors, and font-weights conforming to Zen Browser's design variables.

---

## 3. Verification Plan

Since Zen Browser is built by applying these custom patch overlays on the standard Firefox codebase, upstream developers can verify the changes by:
1. Building the browser: `pnpm run init` followed by `pnpm run build`.
2. Verifying that the patches apply successfully without rejects.
3. Opening the compiled browser, hovering over active tabs (verifying MB/GB values), shared tabs (verifying the shared indicator), and sleeping/discarded tabs (verifying the "Sleeping" label).
