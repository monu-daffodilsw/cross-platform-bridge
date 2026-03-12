Follow these cross-platform rules for every file you create or modify. These rules ensure the code runs on both web (Next.js) and native (React Native / Expo) from a single shared codebase.

## Exempt Paths — Rules Are Off Here

Never apply these rules to:
- `app/` — Next.js shell, web-only by design
- `**/*.native.ts` / `**/*.native.tsx` — platform split files
- `dependencies/ui-library/**` — library defines the primitives
- `node_modules/**`

---

## 1. No HTML Elements in Shared Code

React Native has no DOM. HTML tags crash on native. Replace every HTML element with the equivalent from `@ui-library`:

| HTML | @ui-library |
|---|---|
| `div`, `section`, `article`, `header`, `footer`, `main`, `nav` | `View` |
| `p`, `span`, `h1`–`h6`, `strong`, `em`, `b`, `i`, `label` | `Text` |
| `button` | `Pressable` |
| `input`, `textarea` | `TextInput` |
| `img` | `Image` |
| `ul` / `ol` + `li` | `View` + mapped `View`s or `FlatList` |
| `a` | `Link` from `@ui-library/router` |
| `svg`, `path`, `circle` | `Svg`, `Path`, `Circle` from `@ui-library` |
| `select` | platform-split or `@ui-library` picker |

When migrating `div` → `View`, apply this flex direction audit without exception:

| div className | Action on View |
|---|---|
| No `flex` class at all | No change — block/column behavior matches RN default |
| `flex` but no `flex-row` or `flex-col` | **Add `flex-row`** — web flex defaults to row, must be explicit on View |
| `flex flex-col` | Use `flex` only, **remove `flex-col`** — redundant, RN default is already column |
| `flex flex-row` | Keep as-is |
| `flex flex-col` + other classes | Keep all other classes, **remove only `flex-col`** |

---

## 2. No Direct React Native Imports in Shared Code

Only `.native.tsx` / `.native.ts` files may import from `react-native`, `react-native-svg`, `react-native-safe-area-context`, `@react-native-*`, or any react-native package. Every shared file imports from `@ui-library` instead.
```tsx
// ❌
import { View, Text, TouchableOpacity } from 'react-native';

// ✅
import { View, Text, Pressable } from '@ui-library';
```

---

## 3. No next/navigation, next/link, or expo-router in Shared Code

All navigation goes through `@ui-library/router`. The only exception is `app/` which wires up the `RouterProvider` internally.
```tsx
// ❌
import { useRouter } from 'next/navigation';
import Link from 'next/link';
import { useRouter } from 'expo-router';

// ✅
import { useRouter, usePathname, useParams, Link } from '@ui-library/router';
import { RouterProvider, useRouterContext, buildPath, matchPath } from '@ui-library/router';
```

`RouterProvider` requires a `routes` prop — always pass the project route definitions:
```tsx
import { routes } from '@/router/routes';
<RouterProvider routes={routes}>{children}</RouterProvider>
```

Navigate by route name — never by path string:
```tsx
const router = useRouter();

router.navigate('projectBoard', { id });   // ✅
router.push('/projects/123');              // ❌
```

---

## 4. No CSS Imports in Shared Code

CSS does not exist in React Native. NativeWind translates `className` to native styles — no `.css` imports needed. CSS imports are only allowed inside `app/`.

---

## 5. Strings in JSX Must Be Inside `<Text>`

React Native crashes when a bare string appears directly inside a `View` or any non-text container. Web silently allows it — enforce the stricter native rule everywhere.
```tsx
// ❌
<View>Hello world</View>
<View>{someString}</View>

// ✅
<View><Text>Hello world</Text></View>
<View><Text>{someString}</Text></View>
```

---

## 6. Use className for Styling — Not StyleSheet or Inline Objects

NativeWind translates Tailwind utility classes to native styles on native and to CSS on web.

**No `StyleSheet.create`** — it does not exist on web.

**No static inline style objects** — a style prop whose every value is a static literal should be `className` instead. Inline `style` props are only for values computed at runtime.
```tsx
// ❌
const s = StyleSheet.create({ box: { flex: 1 } });
<View style={{ flex: 1, backgroundColor: '#fff', padding: 16 }}>

// ✅
<View className="flex-1 bg-white p-4">

// ✅  dynamic value — inline style is correct
<View style={{ width: calculatedWidth, backgroundColor: userColor }}>
```

**Design tokens:**
- Background: `bg-[#0a0f1e]`
- Accent: `text-[#6366f1]` / `bg-[#6366f1]`
- Success: `text-[#10b981]`
- Warning: `text-[#f59e0b]`
- Glass: `bg-white/5 border border-white/10 backdrop-blur`
- Headings: `font-space-mono`
- Body: `font-dm-sans`

---

## 7. Platform Splits

Create a `.native.` split only when the entire implementation differs — different imports, different APIs, different logic. A single differing value → use `Platform.OS` inline. A completely different implementation → split.

**Split rules:**
- `File.tsx` — web implementation
- `File.native.tsx` — native implementation
- Both export the same name, accept the same props, return the same shape
- Both files live side by side in the same folder
- Metro auto-picks `.native.*`. Turbopack ignores `.native.*`.
- Max 2 `Platform.OS` checks per shared file — more than 2 means a split is needed
- Never use `Platform.OS === 'web'` inside `dependencies/ui-library` — create a split instead

**Anchor file** (TypeScript stub, never bundled at runtime):
```ts
// hooks/useClipboard.ts
export { useClipboard } from './useClipboard.web';
```

---

## 8. Hooks

**Pure React hooks** (only `useState`, `useEffect`, `useRef`, `useCallback`, `useMemo`) → single `.ts` file, no split needed.

**Hooks that touch platform APIs** → always create a `.native.ts` split:

| Hook touches | Needs split |
|---|---|
| `localStorage` / `sessionStorage` | Yes |
| `navigator.geolocation` | Yes |
| `navigator.clipboard` | Yes |
| `SpeechRecognition` | Yes |
| `window.addEventListener` | Yes |
| `useState` / `useEffect` only | No |

---

## 9. Library (`dependencies/ui-library`)

- Zero project-specific logic — no hardcoded routes, data, or business logic
- Zero project-specific imports — never `@/` or `~/`
- Everything the library needs from the project comes through props or configuration
- Provides: UI primitives, `@ui-library/router`, and `Platform` (same API as React Native's `Platform`)
- Does **not** provide storage abstraction — storage lives in the project's `services/` layer only

---

## 10. Error Handling

No empty catch blocks. Every `catch` must contain at least one statement — a log, a fallback, a state update, or a rethrow.
```ts
// ❌
try { doSomething(); } catch (e) {}

// ✅
try {
  doSomething();
} catch (e) {
  console.error('Failed:', e);
}
```

---

## 11. File Structure

File structure varies per project. Always read the existing project structure before creating or moving files — do not assume any specific folder layout.

The only guaranteed constant across all projects is:
```
dependencies/
  ui-library/      ← standalone shared library, never modified per project
```

Shared code never lives in `app/`. Platform split files always live next to their shared counterpart.

---

## Self-check Before Writing Any File

- [ ] No HTML tags — using `View`, `Text`, `Pressable`, `Image` from `@ui-library`
- [ ] Every `div` with `flex` class audited for flex direction mismatch
- [ ] `flex` only (no direction) → `flex-row` added to View
- [ ] `flex flex-col` → `flex-col` removed from View
- [ ] No `import ... from 'react-native'` in a non-`.native.` file
- [ ] No `next/navigation`, `next/link`, or `expo-router` outside `app/`
- [ ] Navigation uses `@ui-library/router` — `useRouter`, `usePathname`, `useParams`, `Link`
- [ ] No `.css` imports in shared code
- [ ] Every JSX string inside `<Text>`
- [ ] No `StyleSheet.create` — using `className`
- [ ] No static `style={{ }}` — using `className`
- [ ] Navigation uses route names, never path strings
- [ ] No empty `catch {}` blocks
- [ ] If 3+ `Platform.OS` checks needed → create a `.native.` split