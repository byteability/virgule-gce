# Contributing to Virgule

Thank you for your interest in contributing to Virgule! This document covers everything you need to know to set up a local development environment, understand the project structure, and publish a release.

---

## Table of Contents

- [Prerequisites](#prerequisites)
- [Project Structure](#project-structure)
- [Development Setup](#development-setup)
- [Working with the Extension](#working-with-the-extension)
- [Code Style Guidelines](#code-style-guidelines)
- [Submitting a Pull Request](#submitting-a-pull-request)
- [Build Process](#build-process)
- [Release Process](#release-process)

---

## Prerequisites

| Tool | Minimum Version | Notes |
|------|----------------|-------|
| Node.js | 20+ (24 recommended) | Matches the CI environment |
| npm | 10+ | Bundled with Node 20+ |
| Google Chrome | 114+ | Required for Manifest V3 and the File System Access API |
| Git | Any recent version | |

---

## Project Structure

```
develop/
├── app.html              # Single-page UI (source of truth for markup)
├── src/
│   ├── app.js            # Core application logic (~3 800 lines)
│   ├── app.css           # Application styles (Tailwind v4 source)
│   ├── background.js     # MV3 service worker
│   └── main.js           # Vite entry shim
├── public/
│   └── manifest.json     # Chrome Extension manifest (MV3)
├── dist/                 # Built output – gitignored, loaded into Chrome
├── .github/
│   └── workflows/
│       └── release.yml   # Automated release pipeline
├── vite.config.js        # Vite build configuration
├── tailwind.config.js    # Tailwind CSS v4 configuration
├── postcss.config.js     # PostCSS (autoprefixer)
└── package.json          # Scripts and dev dependencies
```

### Key entry points

| File | Role |
|------|------|
| `app.html` | Full application shell; all UI panels are defined here |
| `src/app.js` | All UI logic, file system operations, state management |
| `src/background.js` | Minimal service worker (MV3 requirement) |
| `public/manifest.json` | Extension permissions, `newtab` override, background worker |

---

## Development Setup

### 1. Clone and install

```bash
git clone https://github.com/Frank-Sheppard-Ltd/clio-notes-chrome-ext.git
cd clio-notes-chrome-ext/develop
npm install
```

### 2. Start the dev server

```bash
npm run dev
```

Vite starts a local server (default: `http://localhost:5173`) and automatically redirects `/` to `/app.html`. Hot Module Replacement (HMR) is active — most changes to `src/app.js` and `src/app.css` will reload instantly without a full page refresh.

### 3. Load the extension in Chrome (for extension-specific testing)

Because some APIs (`chrome.tabs`, `chrome.tabGroups`, `chrome.tabs.create`, etc.) are only available inside a real Chrome Extension context, you will need to load a built copy for those features:

```bash
npm run build
```

Then:

1. Open Chrome and navigate to `chrome://extensions`.
2. Enable **Developer mode** (top-right toggle).
3. Click **Load unpacked**.
4. Select the `dist/` folder inside this repository.
5. Open a new tab — Virgule overrides the New Tab page.

> **Tip:** After each code change, run `npm run build` again and click the **↺ Reload** icon next to the extension on `chrome://extensions`.

---

## Working with the Extension

### Permissions

The extension declares the following permissions in [`public/manifest.json`](public/manifest.json):

| Permission | Purpose |
|------------|---------|
| `tabs` | Query open tabs to save as bookmarks |
| `tabGroups` | Read tab group names to organise saved bookmarks |
| `bookmarks` | Save, list, and delete saved tab sessions as native Chrome bookmark folders under "Virgule Sessions" |
| `storage` | Track whether the one-time legacy markdown-to-bookmarks migration has run |

Any new permission must be added to `manifest.json` **and** documented here.

### File System Access API

Virgule uses the browser's **File System Access API** (`showDirectoryPicker`, `getFileHandle`, `createWritable`, etc.) to read and write files directly on the user's machine. No server or cloud storage is involved.

- Library folders are persisted in **IndexedDB** across sessions.
- Permissions for each folder handle are re-requested via `queryPermission` / `requestPermission` on every page load.

### Mock mode

For rapid UI development without a real filesystem, append `?mock=true` to the dev server URL:

```
http://localhost:5173/app.html?mock=true
```

This activates an in-memory `MockFileHandle` so you can exercise UI flows without picking a real directory.

---

## Code Style Guidelines

- **Vanilla JS only** — no frameworks, no TypeScript. Keep it that way unless there is a compelling reason to add a build step.
- **Tailwind CSS v4** for styling. Define design tokens in `tailwind.config.js`; avoid one-off inline styles.
- **Lucide icons** — import only the icons you use from `lucide` to keep the bundle lean.
- **`state` object** is the single source of truth for UI state. Do not store UI state in the DOM.
- Use `void someAsyncFn()` when firing an async function from an event listener without awaiting it — this is the established pattern in the codebase.
- Keep functions small and focused. If a function grows beyond ~30 lines, consider splitting it.
- Log errors with `console.error(...)` before surfacing them to the user via `setStatus(...)`.

---

## Submitting a Pull Request

1. **Fork** the repository and create a feature branch from `main`:
   ```bash
   git checkout -b feature/my-feature
   ```
2. Make your changes, keeping commits atomic and descriptive.
3. Ensure the extension builds cleanly:
   ```bash
   npm run build
   ```
4. Load the built `dist/` in Chrome and manually verify the feature works end-to-end.
5. **Push** your branch and open a Pull Request against `main`.
6. Address any review feedback; a maintainer will merge once approved.

> Please do **not** commit the `dist/` folder or any `.zip` archives — both are gitignored and generated by CI.

---

## Build Process

The build is powered by **Vite 8** with a custom Rollup configuration.

```bash
npm run build
```

### What happens during a build

1. Vite reads `vite.config.js` and processes two entry points:
   - `app.html` → outputs `dist/app.html` + `dist/assets/app.js`
   - `src/background.js` → outputs `dist/background.js` (filename is preserved — required by the manifest)
2. Tailwind CSS v4 is processed via PostCSS and autoprefixer, emitting `dist/assets/app.css`.
3. Static assets from `public/` (including `manifest.json`) are copied verbatim to `dist/`.
4. The output directory (`dist/`) is wiped clean (`emptyOutDir: true`) before every build.

### Output structure

```
dist/
├── app.html
├── background.js
├── manifest.json
└── assets/
    ├── app.js      # Bundled and minified application code
    └── app.css     # Processed Tailwind styles
```

---

## Release Process

Releases are fully automated via the [`.github/workflows/release.yml`](.github/workflows/release.yml) GitHub Actions workflow. The workflow runs on **Node.js 24** on `ubuntu-latest`.

### Triggering a release

#### Option A — Git tag (recommended)

Push a version tag that matches `v*`:

```bash
# Bump the version in public/manifest.json and package.json first
git add public/manifest.json package.json
git commit -m "chore: bump version to v1.2.0"
git tag v1.2.0
git push origin main --tags
```

The workflow triggers automatically when the tag reaches GitHub.

#### Option B — Manual dispatch

Go to **Actions → Release → Run workflow** in the GitHub UI and enter the tag name (e.g. `v1.2.0`). This is useful for re-releasing or hotfixes without pushing a new tag.

### What the workflow does

```
Checkout → Install deps (npm ci) → Build (npm run build) → Zip dist/ → Create GitHub Release
```

| Step | Command | Notes |
|------|---------|-------|
| Install | `npm ci \|\| npm install` | `ci` is preferred for reproducibility |
| Build | `npm run build` | Produces `dist/` |
| Package | `zip -r virgule.zip dist/` | Single archive attached to the release |
| Release | `softprops/action-gh-release@v3` | Creates a GitHub Release with `virgule.zip` attached |

The `GITHUB_TOKEN` secret is provided automatically by GitHub Actions — no manual secret setup is required.

### Versioning

Virgule follows [Semantic Versioning](https://semver.org/):

| Change | Version bump | Example |
|--------|-------------|---------|
| Bug fix | Patch | `1.0.0` → `1.0.1` |
| New feature (backwards-compatible) | Minor | `1.0.0` → `1.1.0` |
| Breaking change or major redesign | Major | `1.0.0` → `2.0.0` |

Before tagging, update the version string in **both** of these files to keep them in sync:

- [`public/manifest.json`](public/manifest.json) — `"version"` field (Chrome Web Store reads this)
- [`package.json`](package.json) — `"version"` field

### Installing a release build

1. Download `virgule.zip` from the [Releases](https://github.com/Frank-Sheppard-Ltd/clio-notes-chrome-ext/releases) page.
2. Unzip it to get the `dist/` folder.
3. Open `chrome://extensions`, enable **Developer mode**, click **Load unpacked**, and select the `dist/` folder.
