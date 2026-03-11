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
| `svg`, `path`, `circle`, `rect` | `Svg`, `Path`, `Circle`, `Rect` from `@ui-library` |
| `select` | platform-split or `@ui-library` picker |

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

All navigation goes through `@ui-library/router`. The only exception is `router/context.*` which wraps Next.js internally.

```tsx
// ❌
import { useRouter } from 'next/navigation';
import Link from 'next/link';

// ✅
import { useRouter, Link } from '@ui-library/router';

// Navigate by route name — never by path string
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

---

## 10. Storage

All storage reads and writes go through `@/services`. No direct `localStorage`, `sessionStorage`, or `AsyncStorage` calls outside the service layer.

```ts
// ❌ anywhere outside services/
localStorage.setItem('key', value);

// ✅
import { storage } from '@/services/localStorage';
storage.set('key', value);
```

Services have two implementations:
- `services/foo.ts` — web, uses `localStorage`
- `services/foo.native.ts` — native, uses `AsyncStorage`

Mock data (`@/utils/mockData`) is only for hook initialization — hooks seed storage on first load. Components never import mock data.

---

## 11. Error Handling

No empty catch blocks. Every `catch` must contain at least one statement — a log, a fallback, a state update, or a rethrow.

```ts
// ❌
try { storage.set(key, val); } catch (e) {}

// ✅
try {
  storage.set(key, val);
} catch (e) {
  console.error('Storage write failed:', e);
}
```

---

## 12. File Structure

```
app/               ← thin Next.js wrappers only, no shared logic
components/
  pages/           ← page components
  layouts/         ← AppLayout, Sidebar, Navbar
  cards/           ← widget and card components
  lists/           ← list and column components
  forms/           ← form components
  modals/          ← modal components
  ui/              ← Button, Input, Badge, Modal, Toast, Avatar
hooks/             ← useAuth, useTasks, useProjects, useNotifications…
services/          ← localStorage.ts + *.native.ts storage services
router/            ← routes.ts, context.tsx, hooks.ts, Link.tsx
utils/             ← utils.ts, mockData.ts
types/             ← index.ts — all TypeScript types
dependencies/
  ui-library/      ← standalone shared library
```

Shared code never lives in `app/`. Platform split files always live next to their shared counterpart.

---

## Self-check Before Writing Any File

- [ ] No HTML tags — using `View`, `Text`, `Pressable`, `Image` from `@ui-library`
- [ ] No `import ... from 'react-native'` in a non-`.native.` file
- [ ] No `next/navigation`, `next/link`, or `expo-router` outside `app/` and `router/context`
- [ ] No `.css` imports in shared code
- [ ] Every JSX string inside `<Text>`
- [ ] No `StyleSheet.create` — using `className`
- [ ] No static `style={{ }}` — using `className`
- [ ] Navigation uses route names, never path strings
- [ ] Storage through `@/services`, never direct `localStorage`
- [ ] No mock data in component files
- [ ] No empty `catch {}` blocks
- [ ] If 3+ `Platform.OS` checks needed → create a `.native.` split
