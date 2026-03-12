Follow these cross-platform rules for every file you create or modify. These rules ensure the code runs on both web (Next.js) and native (React Native / Expo) from a single shared codebase.

## Exempt Paths — Rules Are Off Here

Never apply these rules to:
- `app/` — Next.js shell, web-only by design
- `**/*.native.ts` / `**/*.native.tsx` — platform split files
- `node_modules/**`

## `dependencies/ui-library` Is Read-Only

**Never create, edit, or delete any file inside `dependencies/ui-library` during migration or feature work.** If a primitive is missing or broken, stop and report it — do not fix it by modifying the library.

---

## 0. Required Setup — Do This Before Anything Else

### babel.config.js
`react-native-reanimated/plugin` must be the **last plugin** in `babel.config.js`. Missing this causes `makeMutable` to be undefined at runtime and crashes native.
```js
module.exports = {
  presets: ['babel-preset-expo'],
  plugins: [
    'nativewind/babel',
    'react-native-reanimated/plugin', // ← must be last
  ],
};
```

### metro.config.js
```js
const { getDefaultConfig } = require('expo/metro-config');
const { withNativeWind } = require('nativewind/metro');

const config = getDefaultConfig(__dirname);
module.exports = withNativeWind(config, { input: './app/globals.css' });
```

### global.css
Must be imported in the root `app/layout.tsx`:
```tsx
import './globals.css';
```

### app/[[...slug]]/page.tsx — Catch-All Route
Delete all Next.js file-based routes except `app/layout.tsx` and `app/globals.css`. All routing is handled by `RouterProvider` through a single catch-all:
```tsx
import { RouterProvider } from '@ui-library/router';
import { routes } from '@/router/routes';

export default function Page({ params }: { params: { slug?: string[] } }) {
  const initialPath = '/' + (params.slug?.join('/') ?? '');
  return (
    <RouterProvider routes={routes} initialPath={initialPath}>
      {/* Nav + screen switcher here */}
    </RouterProvider>
  );
}
```

`initialPath` prevents hydration mismatches on direct URL access. Never keep `app/page.tsx` or any other nested route files alongside this catch-all.

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
| `div` with `overflow-scroll` / `overflow-auto` | `ScrollView` |
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
import { View, Text, TouchableOpacity, Platform } from 'react-native';

// ✅
import { View, Text, Pressable, Platform } from '@ui-library';
```

---

## 3. No next/navigation, next/link, or expo-router in Shared Code

All navigation goes through `@ui-library/router`. The only exception is `app/` which wires up the `RouterProvider` internally.
```tsx
// ❌
import { useRouter } from 'next/navigation';
import Link from 'next/link';
import { useRouter } from 'expo-router';

// ✅ in shared code
import { useRouter, usePathname, useParams, Link } from '@ui-library/router';

// app/-only — never in shared components
import { RouterProvider, useRouterContext } from '@ui-library/router';
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

// ✅ dynamic value — inline style is correct
<View style={{ width: calculatedWidth, backgroundColor: userColor }}>
```

### Web-Only Classes — Always Prefix with `web:`

These classes have no native equivalent. NativeWind v4 routes `transition-*` through Reanimated's `makeMutable` — without the `web:` prefix they crash native. Always guard them:
```tsx
// ❌
<View className="transition-colors hover:text-primary focus-visible:ring-2 group-hover:opacity-80">

// ✅
<View className="web:transition-colors web:hover:text-primary web:focus-visible:ring-2 web:group-hover:opacity-80">
```

| Must prefix with `web:` |
|---|
| `transition-*` |
| `hover:*` |
| `group-hover:*` |
| `focus-visible:*` |
| `animate-*` |

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
export { useClipboard } from './useClipboard';
```

---

## 8. Hooks

**Pure React hooks** (only `useState`, `useEffect`, `useRef`, `useCallback`, `useMemo`) → single `.ts` file, no split needed.

**Hooks that touch platform APIs** → always create a `.native.ts` split:

| Hook touches | Needs split |
|---|---|
| `Storage` from `@ui-library` | No — library handles platform internally |
| `navigator.geolocation` | Yes |
| `navigator.clipboard` | Yes |
| `SpeechRecognition` | Yes |
| `window.addEventListener` | Yes |
| `useState` / `useEffect` only | No |

---

## 9. Storage

All storage reads and writes use `Storage` from `@ui-library`. Never call `localStorage`, `sessionStorage`, or `AsyncStorage` directly — the library provides the platform abstraction.
```ts
// ❌
localStorage.setItem('key', value);
import AsyncStorage from '@react-native-async-storage/async-storage';

// ✅
import { Storage } from '@ui-library';
await Storage.setItem('key', value);
await Storage.getItem('key');
await Storage.removeItem('key');
```

---

## 10. Library (`dependencies/ui-library`)

- Zero project-specific logic — no hardcoded routes, data, or business logic
- Zero project-specific imports — never `@/` or `~/`
- Everything the library needs from the project comes through props or configuration
- Provides: UI primitives, `@ui-library/router`, `Platform`, and `Storage`

---

## 11. Error Handling

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

## 12. File Structure

File structure varies per project. Always read the existing project structure before creating or moving files — do not assume any specific folder layout.

The only guaranteed constant across all projects is:
```
dependencies/
  ui-library/      ← read-only, never modified per project
```

Shared code never lives in `app/`. Platform split files always live next to their shared counterpart.

---

## Self-check Before Writing Any File

- [ ] `dependencies/ui-library` not touched — zero files created, edited, or deleted inside it
- [ ] `babel.config.js` has `react-native-reanimated/plugin` as the last plugin
- [ ] `metro.config.js` has `withNativeWind` configured
- [ ] `globals.css` imported in `app/layout.tsx`
- [ ] All Next.js file-based routes removed — single `app/[[...slug]]/page.tsx` catch-all with `RouterProvider` and `initialPath`
- [ ] No HTML tags — using `View`, `Text`, `Pressable`, `Image` from `@ui-library`
- [ ] Every `div` with `flex` class audited for flex direction mismatch
- [ ] `flex` only (no direction) → `flex-row` added to View
- [ ] `flex flex-col` → `flex-col` removed from View
- [ ] `overflow-scroll` / `overflow-auto` divs → `ScrollView`
- [ ] No `import ... from 'react-native'` in a non-`.native.` file
- [ ] `Platform` imported from `@ui-library` not `react-native`
- [ ] No `next/navigation`, `next/link`, or `expo-router` outside `app/`
- [ ] Navigation uses `@ui-library/router` — `useRouter`, `usePathname`, `useParams`, `Link`
- [ ] Navigation uses route names, never path strings
- [ ] `RouterProvider` and `useRouterContext` only in `app/`
- [ ] No `.css` imports in shared code
- [ ] Every JSX string inside `<Text>`
- [ ] No `StyleSheet.create` — using `className`
- [ ] No static `style={{ }}` — using `className`
- [ ] `transition-*`, `hover:*`, `group-hover:*`, `focus-visible:*`, `animate-*` prefixed with `web:`
- [ ] Storage uses `Storage` from `@ui-library` — no direct `localStorage` or `AsyncStorage`
- [ ] No empty `catch {}` blocks
- [ ] If 3+ `Platform.OS` checks needed → create a `.native.` split