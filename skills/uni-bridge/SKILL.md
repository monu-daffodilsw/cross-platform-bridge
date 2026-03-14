## Foundational Rule — Cross-Platform by Default

Every file you create or modify must work on both web (Next.js) and native (React Native / Expo). Before writing any code, verify that every library, API, and browser/OS feature you plan to use works on both platforms. If it doesn't — create a `.native` split. Never assume cross-platform compatibility; prove it.

This rule applies to everything: UI, auth, payments, notifications, file access, sensors, deep links, analytics — no exceptions. The sections below are specific patterns for applying this principle, not an exhaustive list.

---

Follow the cross-platform rules below for every file you create or modify.

## Exempt Paths — Rules Are Off Here

Never apply these rules to:
- `app/` — Next.js shell, web-only by design
- `**/*.native.ts` / `**/*.native.tsx` — platform split files
- `node_modules/**`

## `dependencies/ui-library` — Protected by Default

**Do not modify any file inside `dependencies/ui-library` unless the user explicitly asks to update the library.** During migration or feature work, treat it as read-only. If a primitive is missing or broken, stop and report it.

**When the user explicitly asks to update the library:**
- Read the existing code first — understand the current API, exports, and behavior before changing anything
- Do not remove or rename existing exports — other files depend on them
- Do not change existing prop interfaces — add new optional props instead of modifying existing ones
- Do not add project-specific logic — the library stays generic
- Test that existing imports still work after the change

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

### Flex Direction — `View` Defaults to Column, Web Does Not

`View` is always a flex container with `flex-direction: column` by default. On web, `div` is a block element (not flex) and `display: flex` defaults to row. This mismatch silently breaks layouts. **Do not blindly convert — analyze the original layout intent first.**

**Step 1: Determine the original visual direction.** Look at the existing web code and ask: are the children laid out horizontally (row) or vertically (column)?

**Step 2: Apply the correct class on View based on the original intent:**

| Original web behavior | Action on View |
|---|---|
| Children are horizontal (inline, flexbox row, side-by-side) | **Add `flex-row`** — View defaults to column, must override |
| Children are vertical (block flow, stacked) | No direction class needed — matches View default |
| `flex` but no `flex-row` or `flex-col` | **Add `flex-row`** — web flex defaults to row |
| `flex flex-col` | **Remove `flex-col`** — redundant, View default is column |
| `flex flex-row` | Keep as-is |
| `inline-flex` | **Add `flex-row`** — inline-flex defaults to row on web |
| No `flex` and children use `inline`, `inline-block`, or `float` | **Add `flex-row`** — these are horizontal patterns, View would stack them vertically |

**Key insight:** The question is never "what CSS class did the div have?" — it's "what direction were the children visually flowing?" If they were side by side on web, you need `flex-row` on View regardless of how the web code achieved that layout.

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

- Only modify when the user explicitly asks to update the library
- Zero project-specific logic — no hardcoded routes, data, or business logic
- Zero project-specific imports — never `@/` or `~/`
- Everything the library needs from the project comes through props or configuration
- Provides: UI primitives, `@ui-library/router`, `Platform`, and `Storage`
- When updating: preserve all existing exports and prop interfaces — add, don't replace

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
  ui-library/      ← protected, only modify when user explicitly asks
```

Shared code never lives in `app/`. Platform split files always live next to their shared counterpart.

---

## 13. No Reliance on CSS Inheritance

React Native does not support CSS inheritance. On web, text styling classes on a parent `div` cascade to child text nodes. On native, they do not — `View`, `Pressable`, and `Link` ignore text/font classes entirely.

**Rule:** Text and font classes must go directly on the `Text` or `Icon` component that renders the content — never on a parent `View`, `Pressable`, or `Link`.

| Classes that must be on `Text` / `Icon` directly |
|---|
| `text-*` (size, color, opacity) |
| `font-*` (weight, family) |
| `leading-*` (line height) |
| `tracking-*` (letter spacing) |
| `text-left`, `text-center`, `text-right` (alignment) |

```tsx
// ❌ — text-sm and font-medium are on the parent View, won't reach Text on native
<View className="text-sm font-medium">
  <Text>Hello</Text>
</View>

// ❌ — text color on Pressable won't cascade to Text on native
<Pressable className="text-primary">
  <Text>Click me</Text>
</Pressable>

// ✅ — text classes directly on the Text component
<View>
  <Text className="text-sm font-medium">Hello</Text>
</View>

<Pressable>
  <Text className="text-primary">Click me</Text>
</Pressable>
```

---

## 14. Web-Only Libraries Need `.native` Splits With Interop

Any web library that has a `-native` or RN-specific counterpart needs a platform split. The `.native` file must use NativeWind `styled()` and `cssInterop` / `remapProps` so that `className` works on the native component.

**Pattern:**
1. **Web file** (`Component.tsx`) — uses the web library directly; `className` works natively via CSS.
2. **Native file** (`Component.native.tsx`) — imports from the `-native` package, wraps with `styled()`, and applies `cssInterop` or `remapProps` with `nativeStyleToProp` so `className` maps to the correct native style prop.
3. Both files export the same name and accept the same props.

```tsx
// icons/Icon.tsx — web
import { SomeIcon } from 'some-icon-lib';
export function Icon({ className }: { className?: string }) {
  return <SomeIcon className={className} />;
}

// icons/Icon.native.tsx — native
import { SomeIcon } from 'some-icon-lib-native';
import { cssInterop } from 'nativewind';

cssInterop(SomeIcon, {
  className: { target: 'style', nativeStyleToProp: { color: true, width: true, height: true } },
});

export function Icon({ className }: { className?: string }) {
  return <SomeIcon className={className} />;
}
```

If a library has no `-native` counterpart and uses only web APIs (DOM, Canvas, etc.), it cannot be used in shared code — isolate it behind a platform split that provides a native fallback or no-op.

---

## 15. Centralized HTTP Client

All HTTP calls go through a shared API client — never call `fetch` or `axios` directly from components, hooks, or services. The client provides:

- **Base URL from config** — no hardcoded URLs in call sites
- **Interceptors** — auth headers, token refresh, error normalization
- **Cross-platform compatibility** — works on both web and native without `Platform.OS` checks at each call site

```ts
// ❌ — hardcoded URL, no shared auth, breaks if base changes
const res = await fetch('https://api.example.com/projects');

// ❌ — base URL in every call
const res = await axios.get('https://api.example.com/projects', {
  headers: { Authorization: `Bearer ${token}` },
});

// ✅ — shared client handles base URL, auth, and error handling
import { apiClient } from '@/services/api';
const res = await apiClient.get('/projects');
```

---

## Self-check Before Writing Any File

- [ ] `dependencies/ui-library` not modified unless the user explicitly asked — no removed exports, no changed prop interfaces
- [ ] `babel.config.js` has `react-native-reanimated/plugin` as the last plugin
- [ ] `metro.config.js` has `withNativeWind` configured
- [ ] `globals.css` imported in `app/layout.tsx`
- [ ] All Next.js file-based routes removed — single `app/[[...slug]]/page.tsx` catch-all with `RouterProvider` and `initialPath`
- [ ] No HTML tags — using `View`, `Text`, `Pressable`, `Image` from `@ui-library`
- [ ] Every `div` → `View` conversion audited for visual direction — if children were horizontal on web, `flex-row` is on the View
- [ ] `flex` without direction → `flex-row` added to View
- [ ] `flex-col` removed from View (redundant — View default is column)
- [ ] Inline/float/side-by-side patterns converted to `flex-row` on View
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
- [ ] Text/font classes (`text-*`, `font-*`, `leading-*`, `tracking-*`, alignment) on `Text`/`Icon` directly — not on parent `View`/`Pressable`/`Link`
- [ ] Web-only libraries with `-native` counterparts have `.native` splits with `styled()` / `cssInterop` interop
- [ ] HTTP calls go through shared API client — no direct `fetch`/`axios` with hardcoded URLs