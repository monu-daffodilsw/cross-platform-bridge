# uni-bridge

Claude Code plugin that enforces cross-platform rules so your code runs on both **web** (Next.js) and **native** (React Native / Expo) from a single shared codebase.

Rules are always active — applied automatically on every file you create or modify. No slash commands. No manual steps.

---

## Install

**1. Add the marketplace** — in Claude Code settings (`~/.claude/settings.json`):

```json
{
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

**2. Install the plugin:**

```
/install-plugin uni-bridge@uni-bridge
```

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
