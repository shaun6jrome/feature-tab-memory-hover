# Tab Memory Hover — Zen Browser Feature

> Display per-tab memory usage inside Zen Browser's tab hover card.

## Overview

This repository contains a proposed feature for [Zen Browser](https://zen-browser.app) — a Firefox-based browser — that adds a memory usage indicator to the tab hover card, similar to Google Chrome's implementation.

### Example

When hovering over a tab, the hover card would display:

```
GitHub
github.com

💾 Memory Usage
132 MB
```

## Status

🔬 **Investigation Phase** — Researching technical feasibility.

## Repository Structure

```
docs/                    # Architecture investigation and design documents
  hover-card-architecture.md    # Phase 2: Hover card reverse engineering
  memory-architecture.md        # Phase 3: Firefox memory API investigation
  feasibility-report.md         # Phase 4: Technical feasibility assessment
  architecture-design.md        # Phase 5: Architecture design (if feasible)
src/                     # Feature implementation (Phase 6)
```

## Development Context

Zen Browser is built on top of Mozilla Firefox (Gecko engine). It applies custom patches and overlay modules from a `src/` directory on top of the Firefox source tree. Key architectural facts:

- **Zen-specific modules** live under `src/zen/` (e.g., tabs, glance, workspaces)
- **Firefox patches** are `.patch` files under `src/browser/`
- **Build system** uses `pnpm` with scripts that initialize Firefox source, apply patches, and build

This feature must integrate with both Zen's overlay architecture and Firefox's underlying Gecko APIs.

## Engineering Standards

- Follow Conventional Commits
- Prefer existing Firefox/Gecko infrastructure over new systems
- Keep changes minimal and suitable for upstream review
- Every conclusion must be backed by actual code references

## License

Following Zen Browser's licensing (MPL-2.0).
