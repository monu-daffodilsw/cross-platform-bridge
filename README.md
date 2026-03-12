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

Install the plugin first, then paste each phase prompt into Claude Code one at a time. Verify before moving to the next phase.

---

### Phase 1 — Foundation

> Set up Expo and React Native in this Next.js project so it runs on both web and native. Install all required dependencies, configure Metro and Babel, build the ui-library primitives and router, set up NativeWind. Do not change any existing dependencies or break the current web setup. Stop when done and wait for confirmation before Phase 2.
>
> **Verify:** `npx next build` passes with zero errors.

---

### Phase 2 — Migrate

> Migrate all existing shared components and hooks to be cross-platform. Read the project structure first. Replace HTML elements, fix flex direction, wrap bare strings in Text, replace StyleSheet and static style objects with className, replace all routing imports with @ui-library/router, replace react-native imports with @ui-library, replace direct storage calls with Storage from @ui-library, split any hooks that use browser APIs. Stop when done and wait for confirmation before Phase 3.
>
> **Verify:** `npx tsc --noEmit` passes with zero errors.

---

### Phase 3 — Verify

> Run `npx next build` and confirm zero errors. Run `npx expo start` and confirm zero Metro errors. Fix anything remaining. Report what was completed and what needs attention.

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
