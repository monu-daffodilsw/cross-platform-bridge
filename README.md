# uni-bridge

Claude Code plugin that enforces cross-platform rules so your code runs on both **web** (Next.js) and **native** (React Native / Expo) from a single shared codebase.

Rules are always active — applied automatically on every file you create or modify. No slash commands. No manual steps.

---

## Install

Copy the `SKILL.md` file into your project's `.claude/skills/` directory. Claude Code automatically picks up skills from this location — no commands or config needed.

### Option 1 — Project-only (recommended)

Rules apply only to this project:

```bash
# Run from your project root
mkdir -p .claude/skills/uni-bridge
curl -o .claude/skills/uni-bridge/SKILL.md \
  https://raw.githubusercontent.com/monu-daffodilsw/uni-bridge/main/skills/uni-bridge/SKILL.md
```

### Option 2 — Global

Rules apply to all projects on your machine:

```bash
mkdir -p ~/.claude/skills/uni-bridge
curl -o ~/.claude/skills/uni-bridge/SKILL.md \
  https://raw.githubusercontent.com/monu-daffodilsw/uni-bridge/main/skills/uni-bridge/SKILL.md
```

### Option 3 — Manual copy

1. Download `skills/uni-bridge/SKILL.md` from this repo
2. Place it at one of these paths:
   - **Project-only:** `<your-project>/.claude/skills/uni-bridge/SKILL.md`
   - **Global:** `~/.claude/skills/uni-bridge/SKILL.md`

### After install

Restart Claude Code. The cross-platform rules are now active automatically on every file you create or modify.

---

## Update

Re-run the `curl` command from the install step to pull the latest `SKILL.md` from GitHub, then restart Claude Code.

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
