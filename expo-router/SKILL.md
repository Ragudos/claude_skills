---
name: expo-router
description: >
  Expert guidance for writing, reviewing, and fixing Expo Router v3 code in React Native projects.
  Use this skill whenever the user mentions Expo Router, file-based routing, expo-router, navigation
  in Expo, app directory structure, deep linking, tabs/stack/drawer navigation, protected routes,
  typed routes, or asks how to navigate between screens in a React Native/Expo app. Also trigger
  when the user shares navigation code that uses `expo-router` imports, or asks why their routing
  isn't working. Covers writing new route files from scratch, fixing broken routing logic, and
  guiding users toward correct patterns and away from common pitfalls.
---

# Expo Router v3 — Routing Expert

You are an expert on Expo Router v3 (shipped with Expo SDK 50+). When helping users, always write
correct, idiomatic v3 code and proactively flag anti-patterns. If a user's code uses an older
pattern (e.g., redirect-based auth instead of `Stack.Protected`, `useSearchParams` instead of
`useLocalSearchParams`), point it out and show the v3 way.

---

## 1. File-Based Routing Conventions

Every file inside `app/` is a route. The file path maps directly to the URL.

```
app/
├── _layout.tsx          ← Root layout (required)
├── index.tsx            → /
├── about.tsx            → /about
├── settings/
│   ├── _layout.tsx      ← Layout for /settings/*
│   ├── index.tsx        → /settings
│   └── account.tsx      → /settings/account
├── (tabs)/              ← Route group (no URL segment)
│   ├── _layout.tsx
│   └── feed.tsx         → /feed
├── [id].tsx             → /:id  (dynamic)
├── [...rest].tsx        → /* (catch-all)
└── +not-found.tsx       → catches unmatched routes
```

### Notation quick-reference

| Notation | Meaning |
|---|---|
| `index.tsx` | Default route for its directory |
| `_layout.tsx` | Layout wrapper — not a page |
| `(group)/` | Route group — organizes routes, no URL segment added |
| `[param].tsx` | Dynamic segment — e.g. `/user/123` |
| `[...rest].tsx` | Catch-all (wildcard) |
| `+not-found.tsx` | 404 handler |
| `+html.tsx` | Web HTML boilerplate override |
| `+native-intent.tsx` | Handle unmatched native deep links |

### ✅ Do
- Keep the `app/` directory as the only routing root.
- Use `(group)/` folders freely to organize routes without affecting URLs.
- Every navigable directory should have an `index.tsx` so the router always has a valid default.
- Co-locate non-route files (components, hooks) **outside** `app/` (e.g., `src/components/`).

### ❌ Don't
- Don't put reusable components inside `app/` — they'll be treated as routes.
- Don't nest `app/` inside another `app/`.
- Don't use reserved Metro path names (e.g., `/assets`).

---

## 2. Layouts & Nested Navigators

`_layout.tsx` defines the navigator type for its sibling routes.

```tsx
// app/_layout.tsx — root stack
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="modal" options={{ presentation: 'modal' }} />
    </Stack>
  );
}
```

```tsx
// app/(tabs)/_layout.tsx — tab navigator
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ headerShown: false }}>
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          tabBarIcon: ({ color }) => <Ionicons name="home" size={24} color={color} />,
        }}
      />
      <Tabs.Screen
        name="settings"
        options={{
          title: 'Settings',
          tabBarIcon: ({ color }) => <Ionicons name="settings" size={24} color={color} />,
        }}
      />
    </Tabs>
  );
}
```

### Nesting stacks inside tabs (common pattern)

```
app/
├── _layout.tsx           ← Root Stack
└── (tabs)/
    ├── _layout.tsx       ← Tabs
    ├── index.tsx         ← Home tab
    └── feed/
        ├── _layout.tsx   ← Stack inside feed tab
        ├── index.tsx     ← Feed list
        └── [postId].tsx  ← Feed detail
```

### ✅ Do
- Use `<Stack.Screen name="..." options={{...}} />` inside layouts to configure screens declaratively.
- Set `headerShown: false` on the tab layout screen in the parent stack to avoid double headers.
- Use `<Slot />` in layouts when you don't want a navigator wrapper (renders current child directly).

### ❌ Don't
- Don't deeply nest navigators more than 3 levels — it degrades performance and complicates deep linking.
- Don't duplicate screen declarations — each screen should be declared only once.
- Don't render navigators conditionally based on state inside a layout — use `Stack.Protected` instead (see §5).

---

## 3. Dynamic Routes & Params

### File naming
- `[id].tsx` — single dynamic segment
- `[...slug].tsx` — catch-all (array of segments)
- `[[optionalId]].tsx` — optional catch-all

### Reading params — always use `useLocalSearchParams`

```tsx
// app/user/[id].tsx
import { useLocalSearchParams } from 'expo-router';
import { Text, View } from 'react-native';

export default function UserScreen() {
  const { id } = useLocalSearchParams<{ id: string }>();
  return <View><Text>User: {id}</Text></View>;
}
```

### Passing params when navigating

```tsx
import { Link, router } from 'expo-router';

// Declarative
<Link href={{ pathname: '/user/[id]', params: { id: '42' } }}>Go to user</Link>

// Imperative
router.push({ pathname: '/user/[id]', params: { id: '42' } });
```

### `useLocalSearchParams` vs `useGlobalSearchParams`

| Hook | When to use |
|---|---|
| `useLocalSearchParams` | **Default.** Returns params for the currently focused route. Doesn't re-render on sibling route changes. |
| `useGlobalSearchParams` | Only when you need to react to param changes from a different (non-focused) screen. |

### ✅ Do
- Always use `useLocalSearchParams` unless you specifically need global reactivity.
- Type your params with a generic: `useLocalSearchParams<{ id: string }>()`.
- All params arrive as strings (or arrays of strings) — parse/coerce as needed.

### ❌ Don't
- Don't use `useSearchParams` — it was removed in v3. Use `useLocalSearchParams`.
- Don't pass objects or non-serializable values as route params — stringify first.
- Don't use `useLocalSearchParams` inside `_layout.tsx` to drive auth logic — params aren't reliably available there.

---

## 4. Tab & Drawer Navigation

### Drawer

```tsx
// app/_layout.tsx
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { Drawer } from 'expo-router/drawer';

export default function Layout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Drawer>
        <Drawer.Screen name="index" options={{ title: 'Home' }} />
        <Drawer.Screen name="notifications" options={{ title: 'Notifications' }} />
      </Drawer>
    </GestureHandlerRootView>
  );
}
```

### Platform-specific tabs (native vs. web)

Use platform-specific file extensions to diverge implementations:
- `app-tabs.native.tsx` — loaded on iOS/Android
- `app-tabs.tsx` — loaded on web

See `references/navigation-patterns.md` for a full example.

### Shared routes between tabs

Route groups support shared routes using `(feed,search)/` syntax — routes inside are cloned into both tab groups. Deep-linking will resolve to the first group alphabetically.

---

## 5. Authentication & Protected Routes

**v3 way: `Stack.Protected`** (SDK 53+). Don't use the redirect-in-layout pattern for new code.

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';
import { useSession } from '@/ctx';

function RootNavigator() {
  const { session } = useSession();
  return (
    <Stack>
      <Stack.Protected guard={!!session}>
        <Stack.Screen name="(app)" />
      </Stack.Protected>
      <Stack.Protected guard={!session}>
        <Stack.Screen name="sign-in" />
      </Stack.Protected>
    </Stack>
  );
}

export default function Root() {
  return (
    <SessionProvider>
      <RootNavigator />
    </SessionProvider>
  );
}
```

When `guard={false}`, the screen is inaccessible and the router automatically redirects to the next available screen.

### Role-based access (nested Protected)

```tsx
<Stack.Protected guard={isLoggedIn}>
  <Stack.Protected guard={isAdmin}>
    <Stack.Screen name="admin" />
  </Stack.Protected>
  <Stack.Screen name="profile" />
</Stack.Protected>
```

### Loading state — keep native splash screen up until data is ready

Use `expo-splash-screen` to hold the native splash screen open until all first-load data
(auth state, fonts, feature flags, etc.) has resolved. Returning `null` or a JS-rendered
fallback causes a visible white flash; the native splash screen does not.

```tsx
// app/_layout.tsx
import { useEffect } from 'react';
import { Stack } from 'expo-router';
import * as SplashScreen from 'expo-splash-screen';
import { useFonts } from 'expo-font';
import { SessionProvider, useSession } from '@/ctx';

// Prevent the splash screen from auto-hiding before everything is ready
SplashScreen.preventAutoHideAsync();

function RootNavigator() {
  const { session, isLoading } = useSession();
  const [fontsLoaded] = useFonts({ /* ... */ });

  const ready = !isLoading && fontsLoaded;

  useEffect(() => {
    if (ready) SplashScreen.hideAsync();
  }, [ready]);

  // Return null (not a spinner) — the native splash is still visible
  if (!ready) return null;

  return (
    <Stack screenOptions={{ headerShown: false }}>
      <Stack.Protected guard={!!session}>
        <Stack.Screen name="(app)" />
      </Stack.Protected>
      <Stack.Protected guard={!session}>
        <Stack.Screen name="sign-in" />
      </Stack.Protected>
    </Stack>
  );
}

export default function Root() {
  return (
    <SessionProvider>
      <RootNavigator />
    </SessionProvider>
  );
}
```

Gate `SplashScreen.hideAsync()` on **all** blocking conditions at once (auth, fonts, remote
config, etc.) so the splash only drops once the app is genuinely ready to render.

### ✅ Do
- Use `Stack.Protected` (also works with `Tabs.Protected` and `Drawer.Protected`).
- Wrap auth state in a React Context provider at the root layout.
- Call `SplashScreen.preventAutoHideAsync()` at module level (before any component renders).
- Gate `SplashScreen.hideAsync()` on **all** blocking first-load conditions in one `useEffect`.
- Return `null` (not a custom spinner) while waiting — the native splash remains visible.
- Keep sign-in screens **outside** any protected route group.

### ❌ Don't
- Don't use `<Redirect>` inside a layout to handle auth — it causes navigation-before-mount errors.
- Don't conditionally render different `<Stack>` trees based on auth state — that breaks history.
- Don't declare a screen twice (once inside `Stack.Protected` and once outside).
- Don't rely on `Stack.Protected` for server-side security — it's client-only.
- Don't call `SplashScreen.hideAsync()` before auth state, fonts, and other blocking data are all resolved — each unresolved condition should delay the hide.
- Don't render a JS-level loading spinner as a substitute for the native splash screen — it causes a visible white flash.

---

## 6. Deep Linking & URL Schemes

Deep links are automatic in Expo Router — every route is a deep link.

### Configure scheme in `app.json`

```json
{
  "expo": {
    "scheme": "myapp",
    "intentFilters": [...]
  }
}
```

Links: `myapp://settings/account`, `myapp://user/42`

### Universal links (https)

Configure `assetLinks.json` (Android) and `apple-app-site-association` (iOS). Set `intentFilters` in `app.json`. No code changes needed for routing.

### Fix missing back button on deep link

```tsx
// app/_layout.tsx
export const unstable_settings = {
  initialRouteName: '(tabs)',
};
```

This ensures the navigation stack is pre-populated when entering via deep link.

### +native-intent for unmatched native URLs

```tsx
// app/+native-intent.tsx
import { router } from 'expo-router';

export function handleURL({ url }: { url: string }): boolean {
  if (url.includes('magic-link')) {
    router.replace('/sign-in');
    return true;
  }
  return false;
}
```

### ✅ Do
- Test deep links in development: `npx uri-scheme open myapp://route --ios`.
- Set `initialRouteName` in `unstable_settings` so back navigation works after a cold deep link.

### ❌ Don't
- Don't manually intercept deep links with `Linking.addEventListener` — use `+native-intent.tsx`.

---

## 7. Typed Routes

Enable in `app.json`:

```json
{
  "expo": {
    "experiments": {
      "typedRoutes": true
    }
  }
}
```

Expo CLI generates `expo-env.d.ts` on first `npx expo start`. Do not commit or manually edit this file.

### Usage

```tsx
// Links are autocompleted and type-checked
<Link href="/settings/account">Settings</Link>
<Link href={{ pathname: '/user/[id]', params: { id: '42' } }}>User</Link>

// Typed params
const { id } = useLocalSearchParams<'/user/[id]'>(); // id: string

// Manual query params
const { query } = useLocalSearchParams<{ query?: string }>();

// Both route + query params
const { id, query } = useLocalSearchParams<'/user/[id]', { query?: string }>();
```

### ✅ Do
- Enable typed routes from project start — much harder to bolt on later.
- Add `expo-env.d.ts` to `.gitignore`.
- Use typed generics on `useLocalSearchParams` for full safety.

### ❌ Don't
- Don't manually edit or delete `expo-env.d.ts`.
- Don't use string literals for `href` when typed routes are on — use the typed `href` object form for dynamic routes.

---

## Common Anti-Patterns (quick reference)

| Anti-pattern | Fix |
|---|---|
| `<Redirect>` in layout for auth | Use `Stack.Protected` |
| Conditional `<Stack>` / `<Tabs>` rendering | Use `Stack.Protected` |
| `useSearchParams` | Use `useLocalSearchParams` |
| `useGlobalSearchParams` everywhere | Use `useLocalSearchParams` by default |
| Components inside `app/` | Move to `src/components/` or `components/` |
| No `index.tsx` in a navigable directory | Add `index.tsx` |
| Navigating before root layout mounts | Don't navigate in layout top-level — use effects or Protected |
| Duplicate screen declarations | Each screen declared once only |
| Passing objects as route params | Stringify / encode params first |
| Deep nesting (4+ levels) | Flatten with route groups |
| Hiding splash screen before all data loads | Gate `SplashScreen.hideAsync()` on auth + fonts + all blocking data |
| JS spinner instead of native splash | Return `null` while loading; let native splash screen show |

---

## Reference Files

- `references/navigation-patterns.md` — Full code examples: tabs+stack, drawer, shared routes, modals
- `references/deep-linking.md` — Universal links, intent filters, testing

Read these when writing complete feature implementations or when the user asks for a full working example.
