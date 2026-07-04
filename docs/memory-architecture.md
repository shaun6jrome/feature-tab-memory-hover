# Firefox Memory Architecture Investigation

> Phase 3: Research Gecko internals for per-tab memory reporting.

## How `about:processes` Obtains Memory Information

### Primary API: `ChromeUtils.requestProcInfo()`

Source: [`toolkit/components/aboutprocesses/content/aboutProcesses.js`](https://searchfox.org/mozilla-central/source/toolkit/components/aboutprocesses/content/aboutProcesses.js)

The `about:processes` page (Firefox's built-in Task Manager, `Shift+Esc`) calls:

```javascript
const procInfo = await ChromeUtils.requestProcInfo();
```

This is a **`[ChromeOnly]`** privileged API defined in WebIDL:

Source: [`dom/chrome-webidl/ChromeUtils.webidl`](https://searchfox.org/mozilla-central/source/dom/chrome-webidl/ChromeUtils.webidl)

```webidl
[ChromeOnly]
static Promise<ParentProcInfoDictionary> requestProcInfo();
```

### Data Structure Returned

Source: [`widget/ProcInfo.h`](https://searchfox.org/mozilla-central/source/widget/ProcInfo.h)

```
ParentProcInfoDictionary {
    pid: long long
    threads: ThreadInfoDictionary[]
    residentSetSize: unsigned long long     // RSS in bytes
    residentUniqueSize: unsigned long long  // USS in bytes (Linux/macOS)
    cpuUser: double                         // CPU time (user, ms)
    cpuKernel: double                       // CPU time (kernel, ms)
    children: ChildProcInfoDictionary[]
}

ChildProcInfoDictionary {
    pid: long long
    childID: unsigned long long             // Firefox internal child ID
    type: WebIDLProcType                    // "web", "webIsolated", "extension", "gpu", etc.
    threads: ThreadInfoDictionary[]
    residentSetSize: unsigned long long     // RSS in bytes
    residentUniqueSize: unsigned long long  // USS in bytes
    cpuUser: double
    cpuKernel: double
    windows: WindowInfoDictionary[]         // BrowsingContexts in this process
    origin: UTF8String                      // Site origin (for webIsolated)
}
```

### OS-Level Implementation

The `ProcInfo.h` native struct is populated by OS-specific code:
- **macOS:** `mach_task_basic_info`
- **Linux:** `/proc/[pid]/statm`
- **Windows:** `GetProcessMemoryInfo`

## Does Firefox Expose Per-Tab Memory?

### Answer: **No. Firefox exposes per-PROCESS memory only.**

`ChromeUtils.requestProcInfo()` returns memory metrics (`residentSetSize`, `residentUniqueSize`)
at the **OS process level**. There is no built-in API that directly returns "this specific tab
uses X MB."

### Why This Is Complex Under Fission

Under Firefox's **Fission** (Site Isolation) architecture:

1. **Multiple tabs from the same origin can share a content process**
2. **A single tab can span multiple processes** (cross-origin iframes in separate processes)
3. **The memory figure for a process is the total for ALL documents hosted in that process**

### What `about:processes` Does as a Workaround

The `about:processes` code:
1. Calls `ChromeUtils.requestProcInfo()` for process-level memory
2. Iterates over `BrowsingContext` trees to map processes → tabs
3. If a process hosts only one tab, tab memory ≈ process RSS
4. If a process hosts multiple tabs, it shows combined memory for all tabs in that process
5. Labels memory as an **estimate**

### Alternative: `about:memory` / `nsIMemoryReporterManager`

```javascript
const mgr = Cc["@mozilla.org/memory-reporter-manager;1"]
              .getService(Ci.nsIMemoryReporterManager);
```

This provides extremely granular per-window/per-DOM breakdown, but:
- **Forces a full GC + memory report** (heavyweight)
- **Not suitable for real-time tooltip display**
- Too slow for hover card use

### Alternative: `performance.measureUserAgentSpecificMemory()`

**NOT available in Firefox** — this is a Chromium-only API (Chrome 89+). Cannot be used.

## How Tabs Map to Processes Under Fission

### Process Mapping Chain

```
Tab → BrowsingContext (top-level) → WindowGlobalParent → ContentParent → pid/childID
```

### Privileged JS Code to Map Tab → PID

```javascript
// From a tab element in browser chrome:
let browser = tab.linkedBrowser;
let browsingContext = browser.browsingContext;            // CanonicalBrowsingContext
let windowGlobal = browsingContext.currentWindowGlobal;   // WindowGlobalParent
let osPid = windowGlobal?.osPid;                          // OS process ID
```

### Firefox Already Has This Mapping

The `tab-hover-preview.mjs` module already maps tabs to PIDs:

```javascript
// From tab-hover-preview.mjs, line ~557-565:
get #displayPids() {
    const pids = this.win.gBrowser.getTabPids(this.#tab);
    if (!pids.length) {
        return "";
    }
    let pidLabel = pids.length > 1 ? "pids" : "pid";
    return `${pidLabel}: ${pids.join(", ")}`;
}
```

The `gBrowser.getTabPids(tab)` method returns an array of PIDs associated with a tab
(multiple PIDs if the tab has cross-origin iframes under Fission).

### Source References

| Component | Source | Role |
|-----------|--------|------|
| `WindowGlobalParent::OsPid()` | `dom/ipc/WindowGlobalParent.cpp` | Maps BrowsingContext → OS PID |
| `nsIRemoteTab.osPid` | `dom/ipc/nsIRemoteTab.h` | Read-only PID attribute |
| `E10SUtils.sys.mjs` | `toolkit/modules/E10SUtils.sys.mjs` | Process selection logic |
| `ProcessIsolation.cpp` | `dom/ipc/ProcessIsolation.cpp` | Fission isolation logic |

### Important Caveats

1. `browsingContext.currentWindowGlobal` can be **`null`** for unloaded/discarded tabs
2. Cross-origin iframes within a tab live in **different** processes
3. `gBrowser.getTabPids(tab)` already handles the multi-PID case
4. A process shared by multiple tabs means memory is **combined, not split**

## Can We Access Memory Data from Zen's Overlay Scripts?

### Answer: **YES — absolutely.**

Zen Browser's overlay scripts run in the **browser chrome context**, which has the same
privilege level as `about:processes` itself.

### Proof: Available APIs from Chrome JS

```javascript
// ✅ ChromeUtils.requestProcInfo() — the main memory API
const procInfo = await ChromeUtils.requestProcInfo();

// ✅ Tab → Process mapping
const tab = gBrowser.selectedTab;
const browser = tab.linkedBrowser;
const bc = browser.browsingContext;
const wg = bc.currentWindowGlobal;
const tabPid = wg?.osPid;

// ✅ Find matching child process
const childProc = procInfo.children.find(c => c.pid === tabPid);
const memoryBytes = childProc?.residentSetSize || 0;
const memoryMB = (memoryBytes / (1024 * 1024)).toFixed(1);

// ✅ Services object for preferences
Services.prefs.getBoolPref("browser.tabs.hoverPreview.enabled");
```

### Performance Characteristics

| API | Overhead | Suitable for Hover? |
|-----|----------|-------------------|
| `ChromeUtils.requestProcInfo()` | **Low** — reads OS stats, no GC | ✅ Yes |
| `nsIMemoryReporterManager` | **Very High** — forces full GC | ❌ No |
| `performance.measureUserAgentSpecificMemory()` | N/A | ❌ Not in Firefox |

`requestProcInfo()` is lightweight — it reads OS process statistics without forcing garbage
collection. Suitable for real-time updates on a ~1-2 second cache interval.

## Summary Table

| Question | Answer |
|----------|--------|
| How does `about:processes` get memory? | `ChromeUtils.requestProcInfo()` |
| Per-tab or per-process? | **Per-process only** |
| Can we map tab → process? | Yes, via `tab.linkedBrowser.browsingContext.currentWindowGlobal?.osPid` |
| Firefox already does tab→PID mapping? | Yes, `gBrowser.getTabPids(tab)` exists |
| Can Zen's overlay JS access this? | **Yes** — same privilege level as `about:processes` |
| Is `requestProcInfo()` fast enough for tooltips? | **Yes** — low overhead, OS-level stats |
| Shared process issue? | Memory is combined for all tabs in a process |
