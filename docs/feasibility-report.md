# Technical Feasibility Report: Tab Memory Usage in Hover Card

> Phase 4: Assessment of implementability, blockers, APIs, and risks.

After reverse-engineering Zen Browser's layout patches and Firefox's underlying Gecko memory and hover subsystems, we have evaluated the feasibility of displaying per-tab memory usage inside the tab hover card.

---

## Feasibility Status: **FULLY FEASIBLE**

The proposed feature is technically feasible and can be implemented with minimal overhead. It does not require any binary/C++ changes or new XPCOM components. It can be implemented entirely in the JavaScript frontend layer using existing Mozilla chrome APIs and applied as a patch within Zen Browser's build/source structure.

---

## Required APIs

### 1. Process Info Retrieval: `ChromeUtils.requestProcInfo()`
- **Availability:** Privileged Chrome JS context (`[ChromeOnly]`).
- **Function:** Asynchronously fetches CPU, memory (RSS/USS), and thread details for all processes from the OS.
- **Overhead:** Extremely low. It reads OS metrics directly (e.g., from `/proc` on Linux, task APIs on macOS/Windows) without triggering expensive garbage collection or memory reporting sweeps.

### 2. Tab-to-Process Mapping: `canonicalBrowsingContext.currentWindowGlobal.osPid`
- **Availability:** Via `tab.linkedBrowser.browsingContext.currentWindowGlobal?.osPid`.
- **Function:** Matches the target tab to its content process OS PID. Firefox already implements this under the hood in `tab-hover-preview.mjs` via `gBrowser.getTabPids(tab)`.

---

## Required Files to Modify/Patch

To implement this feature in Zen Browser, we need to modify:

### 1. `src/browser/components/tabbrowser/content/tab-hover-preview-mjs.patch` [NEW PATCH]
- We will patch `browser/components/tabbrowser/content/tab-hover-preview.mjs` to:
  - In `TabPanel.constructor()`, programmatically inject a new DOM container (`.tab-preview-memory-container`) into the hover panel's text container.
  - In `TabPanel.#updatePreview()`, call `ChromeUtils.requestProcInfo()`.
  - Resolve the tab's process ID(s) and match them against the process information.
  - Render the formatted memory size (e.g., `132 MB`) in the injected container.

### 2. CSS Overrides / Custom Patch
- Style the new memory container using Zen Browser's existing CSS styling system (such as adding CSS rules under `src/browser/themes/shared/tabbrowser/tabs-css.patch` or as a new CSS override).

---

## Data Retrieval & Caching Strategy

Since `ChromeUtils.requestProcInfo()` is asynchronous, we must handle the asynchronous updates correctly:
1. When a tab is hovered, `TabPanel.activate(tab)` triggers `#updatePreview()`.
2. We invoke `await ChromeUtils.requestProcInfo()`.
3. While the promise is resolving, we display a placeholder or hide the memory label.
4. Once the promise resolves, we update the text content.
5. **Caching:** To avoid hammering the OS with process statistics requests during rapid mouse movements, we can cache the process info data for `1500ms`. If `#updatePreview()` is called within the cache window, we reuse the cached data.

---

## Technical Blockers & Fission Architecture

There are no hard blockers, but Fission (Site Isolation) introduces two architectural nuances:

1. **Shared Content Processes:** Under Fission, multiple tabs of the same origin can share a single content process.
   - *Impact:* The reported memory is the total memory of that content process.
   - *Mitigation:* We will check if a process hosts multiple tabs. If it does, we can display a shared indicator next to the memory value (e.g., `132 MB (shared)`), or divide the process RSS by the number of active tabs in that process to show an estimate. Chromium's implementation displays the process total. We will display the process RSS with an indicator when shared, ensuring 100% accuracy.
2. **Discarded / Unloaded Tabs:** Tabs that are unloaded or in a discarded state do not have an active process.
   - *Impact:* `osPid` is null/undefined.
   - *Mitigation:* We will detect this state and display a clear label (e.g., `Sleeping` or `Unloaded`) or hide the memory section entirely.

---

## Estimated Complexity & Performance

- **Complexity:** **Low-Medium**. The implementation is self-contained within the hover preview module and requires patching only 1 JS file and adding a few CSS rules.
- **CPU Overhead:** **Negligible**. Caching the process info ensures `requestProcInfo()` is called at most once per 1.5 seconds.
- **Memory Overhead:** **None**. No heavy polling loop runs while the hover card is closed. Data is only requested on demand when the hover card is active.

---

## Risks

- **Upstream Changes:** Firefox central updates might refactor `tab-hover-preview.mjs`. Since Zen Browser tracks Firefox releases, patches must be kept small and focused so they apply cleanly. Programmatic DOM manipulation within the JS file minimizes conflicts with static XUL markup changes.
- **Race Conditions:** If the user moves the mouse rapidly, `requestProcInfo` promises could resolve out of order.
  - *Mitigation:* The update function checks if the active tab has changed (`this.#tab == tab`) before applying the resolved memory value.
