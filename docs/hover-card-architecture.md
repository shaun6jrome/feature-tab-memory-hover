# Hover Card Architecture — Zen Browser Investigation

> Phase 2: Reverse engineering the tab hover card system.

## Summary

Zen Browser inherits Firefox's **Tab Hover Preview** system without modification.
There is **no `tab-hover-preview-mjs.patch`** in the Zen Browser source tree, meaning the
entire hover preview panel operates using upstream Firefox code.

## Firefox Tab Hover Preview Architecture

### Source Files

| Component | Path (mozilla-central) | Purpose |
|-----------|----------------------|---------|
| **Main module** | `browser/components/tabbrowser/content/tab-hover-preview.mjs` | Core logic for hover preview panels |
| **CSS** | `browser/themes/shared/tabbrowser/tab-hover-preview.css` | Visual styling |
| **Panel markup** | `#tab-preview-panel` element in `browser.xhtml` | DOM structure (XUL panel) |

### Key Classes

The `tab-hover-preview.mjs` module defines three classes:

1. **`TabHoverPanelSet`** — Orchestrator that manages multiple hover panels:
   - `this.tabPanel` → `TabPanel` (single-tab preview)
   - `this.tabGroupPanel` → `TabGroupPanel` (group preview)
   - `this.tabNotePanel` → `TabNotePanel` (note preview)

2. **`TabPanel extends HoverPanel`** — The primary tab hover card:
   - Displays title (`.tab-preview-title`)
   - Displays URL (`.tab-preview-uri`)
   - Displays PIDs (`.tab-preview-pid`) — already shows process IDs when debugging
   - Displays thumbnail (`.tab-preview-thumbnail-container`)
   - Displays tab note button (`.tab-preview-add-note`)

3. **`HoverPanel`** — Base class for all hover panels.

### Hover Lifecycle

```
User hovers over tab
    ↓
tabs.js fires mouseenter → gBrowser.tabContainer
    ↓
TabHoverPanelSet.activate(tab)
    ↓
TabPanel.activate(tab) {
    panelElement.openPopup(tab, popupOptions)  // XUL popup
    ↓
    addEventListener("popupshowing")
    ↓
    #updatePreview() {
        .tab-preview-title ← tab.textLabel.textContent
        .tab-preview-uri   ← getPrettyURI(browser.currentURI)
        .tab-preview-pid   ← getTabPids(tab)       // ALREADY EXISTS
        .tab-preview-activeness ← docShellIsActive  // ALREADY EXISTS
        thumbnail ← PageThumbs.captureTabPreviewThumbnail()
    }
}
```

### DOM Structure of `#tab-preview-panel`

```xml
<panel id="tab-preview-panel">
  <div class="tab-preview-content-interactive">
    <!-- Add Note button injected here -->
  </div>
  <div class="tab-preview-content-main">
    <div class="tab-preview-text-container">
      <div class="tab-preview-title">Page Title</div>
      <div class="tab-preview-uri">example.com</div>
    </div>
    <div class="tab-preview-pid-activeness">
      <span class="tab-preview-pid">pid: 12345</span>
      <span class="tab-preview-activeness">[A]</span>
    </div>
    <div class="tab-preview-thumbnail-container">
      <canvas><!-- thumbnail --></canvas>
    </div>
  </div>
</panel>
```

### Configuration Preferences

| Preference | Default | Purpose |
|-----------|---------|---------|
| `browser.tabs.hoverPreview.enabled` | `true` | Enable/disable hover preview |
| `browser.tabs.hoverPreview.showThumbnails` | `false` | Show page thumbnail |
| `browser.tabs.cardPreview.delayMs` | (varies) | Delay before showing |
| `ui.tooltip.delay_ms` | (varies) | System tooltip delay |

### Popup Positioning (Vertical Tabs)

The `popupOptions` getter in `TabPanel` handles both horizontal and vertical tab modes:
- **Horizontal:** Panel anchors below the tab (`bottomleft topleft`)
- **Vertical:** Panel anchors to the side of the tab, respecting sidebar position
- **Pinned grid:** Panel anchors at tab corner with offset

## Zen Browser Specifics

### No Hover Preview Patches

Confirmed: The Zen Browser `src/` tree contains **zero patches** to `tab-hover-preview.mjs`.
Searched the full directory listing and found no files matching `hover-preview` or `hover-card`.

### Zen's Tab-Related Patches

| File | Purpose |
|------|---------|
| `tab-js.patch` | Modifies tab element: adds `tab-reset-pin-button`, `zen-tab-sublabel`, `tab-reset-button` |
| `tabbrowser-js.patch` | Modifies tab management: workspace integration, glance tabs, essential tabs |
| `tabs-js.patch` | Tab container behavior modifications |

### Zen's Glance Feature (Different from Hover Preview)

The `ZenGlanceManager.mjs` implements a **click-triggered overlay preview** (not hover).
This is unrelated to the tab hover card — it's activated by modifier+click on links.

### Key Zen Tab Modules

| Module | Path | Purpose |
|--------|------|---------|
| `ZenPinnedTabManager.mjs` | `src/zen/tabs/` | Pinned tab management |
| `ZenUIManager.mjs` | `src/zen/common/modules/` | General UI orchestration |
| `ZenGlanceManager.mjs` | `src/zen/glance/` | Link preview overlay |
| `ZenViewSplitter.mjs` | `src/zen/split-view/` | Split view management |

## Integration Points for Memory Feature

The cleanest integration points are:

1. **`tab-hover-preview.mjs` patch** — Add a memory display to `#updatePreview()` alongside the existing PID display
2. **New CSS** — Add styling for the memory label in the hover preview
3. **No new XUL needed** — We can create DOM elements programmatically in the patch, similar to how `#displayPids` works

### Existing Infrastructure We Can Reuse

- **PID display mechanism** — The `#displayPids` getter already maps tabs to processes via `gBrowser.getTabPids(tab)`. We can follow the same pattern for memory.
- **Preference system** — Add a new pref like `zen.tabs.hoverPreview.showMemory` to control the feature.
- **CSS classes** — Follow the existing `.tab-preview-*` naming convention.

## References

- Firefox source (mozilla-central): `browser/components/tabbrowser/content/tab-hover-preview.mjs`
- Firefox CSS: `browser/themes/shared/tabbrowser/tab-hover-preview.css`
- Zen Browser repo: `zen-browser/desktop` (dev branch)
- Zen tab patches: `src/browser/components/tabbrowser/content/*.patch`
