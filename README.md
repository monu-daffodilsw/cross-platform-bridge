# uni-bridge

Claude Code plugin that enforces cross-platform rules so your code runs on both **web** (Next.js) and **native** (React Native / Expo) from a single shared codebase.

Rules are always active — applied automatically on every file you create or modify. No slash commands. No manual steps.

---

## Install

### Option 1 — CLI commands (recommended)

**Global install** (available in all projects):

```bash
# Step 1 — Register the marketplace (one-time per machine)
claude plugin marketplace add https://github.com/monu-daffodilsw/uni-bridge.git

# Step 2 — Install the plugin
claude plugin install uni-bridge@uni-bridge
```

**Project-only install** (only active in the current project):

```bash
# Run from inside your project directory

# Step 1 — Register the marketplace (one-time per machine)
claude plugin marketplace add https://github.com/monu-daffodilsw/uni-bridge.git

# Step 2 — Install scoped to this project only
claude plugin install uni-bridge@uni-bridge --scope project
```

**Step 3 — Reopen Claude Code**

Skills are cached at startup. Close and reopen Claude Code after installation.

---

## Update

When the plugin is updated on GitHub, refresh the marketplace first to pull the latest, then update the plugin:

```bash
# Step 1 — Refresh marketplace (fetches latest from GitHub)
claude plugin marketplace refresh uni-bridge

# Step 2 — Update the plugin
claude plugin update uni-bridge@uni-bridge

# For project-only install, add --scope project to step 2
claude plugin update uni-bridge@uni-bridge --scope project

# Step 3 — Reopen Claude Code to reload updated skills
```

If `marketplace refresh` is not available, remove and re-add instead:

```bash
claude plugin marketplace remove uni-bridge
claude plugin marketplace add https://github.com/monu-daffodilsw/uni-bridge.git
claude plugin update uni-bridge@uni-bridge
```

---

### Option 2 — Manual settings

**Global** — edit `~/.claude/settings.json`:

```json
{
  "enabledPlugins": {
    "uni-bridge@uni-bridge": true
  },
  "extraKnownMarketplaces": {
    "uni-bridge": {
      "source": {
        "source": "git",
        "url": "https://github.com/monu-daffodilsw/uni-bridge.git"
      }
    }
  }
}
```

**Project-only** — create `.claude/settings.json` in your project root:

```json
{
  "enabledPlugins": {
    "uni-bridge@uni-bridge": true
  },
  "extraKnownMarketplaces": {
    "uni-bridge": {
      "source": {
        "source": "git",
        "url": "https://github.com/monu-daffodilsw/uni-bridge.git"
      }
    }
  }
}
```

Then reopen Claude Code.

---

## Converting a Next.js Project to Cross-Platform

Install the plugin first. Then paste each phase prompt into Claude Code **one at a time**. Verify it passes before moving to the next phase.

---

### Phase 1 — Router

Replace all Next.js file-based routing with `@ui-library/router`. Keeps the web app running exactly as before — just changes how routing works internally.

**Paste into Claude Code:**

> Phase 1 — Router. Read the project structure first. Replace all Next.js file-based routing with `@ui-library/router`. Keep only `app/layout.tsx` and `app/globals.css` — delete all other files inside `app/`. Create a single `app/[[...slug]]/page.tsx` catch-all. Create `router/routes.ts` with all existing routes. Replace every routing import from `next/navigation` and `next/link` across all components and hooks. Do not change any UI, styles, or dependencies. Stop when done and wait for confirmation before Phase 2.

**Verify:** `npx next build` passes and all pages open correctly in the browser.

---

### Phase 2 — Primitives

Replace all HTML elements with `@ui-library` components and fix all class names to be NativeWind-compatible. After this phase the codebase is cross-platform ready.

**Paste into Claude Code:**

> Phase 2 — Primitives. Phase 1 is complete — routing is already migrated to `@ui-library/router`. Read the project structure first. Replace every HTML element in shared components with the `@ui-library` equivalent. Audit flex direction on every `div`→`View` migration. Wrap every bare JSX string in `<Text>`. Remove all `StyleSheet.create` and static `style={{ }}` objects — use `className` instead. Prefix web-only classes with `web:`. Replace all direct `localStorage` and `AsyncStorage` calls with `Storage` from `@ui-library`. Do not touch `app/` or any `.native.tsx` file. Stop when done and wait for confirmation before Phase 3.

**Verify:** `npx tsc --noEmit` passes with zero errors.

---

### Phase 3 — Expo

Install Expo and React Native, configure the native bundler, and create platform splits for any hooks that use browser-only APIs. After this phase the app runs on both web and native.

**Paste into Claude Code:**

> Phase 3 — Expo. Phases 1 and 2 are complete — routing is migrated and all components use `@ui-library` primitives. Read the project structure first. Install Expo and all required React Native dependencies. Configure `babel.config.js` and `metro.config.js` for NativeWind. Confirm `globals.css` is imported in `app/layout.tsx`. Create `.native.ts` splits for any hook that uses browser-only APIs. Do not change any component logic or styles. Stop when done and wait for confirmation before Phase 4.

**Verify:** `npx expo start` runs with zero Metro errors and the UI renders correctly on device or simulator.

---

### Phase 4 — Audit

Scan the entire codebase for anything missed in the previous phases and fix it. This is the final check before the project is fully cross-platform.

**Paste into Claude Code:**

> Phase 4 — Audit. Phases 1, 2, and 3 are complete. Read the entire project structure. Check every shared file against the cross-platform rules enforced by the uni-bridge plugin. Find and fix anything that was missed — HTML elements, wrong imports, bare strings, static styles, missing `web:` prefixes, direct storage calls, empty catch blocks, or anything else that would fail on native. Report every issue found and what was fixed. If everything is clean, confirm the project is fully cross-platform.

**Verify:** `npx next build` passes. `npx expo start` passes. `npx tsc --noEmit` passes with zero errors.

---

## Rules Enforced Automatically

| # | Rule | What it catches |
|---|---|---|
| 1 | No HTML in shared code | `div`, `span`, `button`, etc. → use `@ui-library` |
| 2 | No react-native imports in shared code | `import from 'react-native'` in non-.native. files |
| 3 | No next/navigation or expo-router in shared code | use `@ui-library/router` |
| 4 | No CSS imports in shared code | `.css` files outside `app/` |
| 5 | Strings in JSX inside `<Text>` | bare strings in `<View>` crash React Native |
| 6 | No StyleSheet.create | doesn't exist on web |
| 7 | No static inline style objects | use `className` with NativeWind |
| 8 | Platform splits when implementations differ | max 2 `Platform.OS` checks per file |
| 9 | No Platform.OS === 'web' in library | create a .native. split instead |
| 10 | All storage through services | no direct `localStorage` / `AsyncStorage` |
| 11 | No mock data in components | only in hook initialization |
| 12 | No empty catch blocks | handle, log, or rethrow |

## Exempt Paths

- `app/` — Next.js shell
- `**/*.native.ts(x)` — platform split files
- `dependencies/ui-library/**` — library
- `node_modules/**`
