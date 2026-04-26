# Navigation Patterns — Full Examples

## 1. Tabs + Stack (most common app structure)

```
app/
├── _layout.tsx           ← Root stack
├── +not-found.tsx
└── (tabs)/
    ├── _layout.tsx       ← Tab bar
    ├── index.tsx         ← Home
    ├── search.tsx        ← Search
    └── profile/
        ├── _layout.tsx   ← Stack inside profile tab
        ├── index.tsx     ← Profile list / overview
        └── [userId].tsx  ← User detail
```

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen name="+not-found" />
    </Stack>
  );
}
```

```tsx
// app/(tabs)/_layout.tsx
import { Tabs } from 'expo-router';
import { Ionicons } from '@expo/vector-icons';

export default function TabLayout() {
  return (
    <Tabs screenOptions={{ tabBarActiveTintColor: '#007AFF' }}>
      <Tabs.Screen
        name="index"
        options={{
          title: 'Home',
          headerShown: false,
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="home-outline" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="search"
        options={{
          title: 'Search',
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="search-outline" size={size} color={color} />
          ),
        }}
      />
      <Tabs.Screen
        name="profile"
        options={{
          title: 'Profile',
          headerShown: false,
          tabBarIcon: ({ color, size }) => (
            <Ionicons name="person-outline" size={size} color={color} />
          ),
        }}
      />
    </Tabs>
  );
}
```

```tsx
// app/(tabs)/profile/_layout.tsx
import { Stack } from 'expo-router';

export default function ProfileStackLayout() {
  return (
    <Stack>
      <Stack.Screen name="index" options={{ title: 'Profile' }} />
      <Stack.Screen name="[userId]" options={{ title: 'User' }} />
    </Stack>
  );
}
```

---

## 2. Modal over tabs

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';

export default function RootLayout() {
  return (
    <Stack>
      <Stack.Screen name="(tabs)" options={{ headerShown: false }} />
      <Stack.Screen
        name="compose"
        options={{
          presentation: 'modal',
          title: 'New Post',
        }}
      />
    </Stack>
  );
}
```

```tsx
// Trigger from anywhere:
import { router } from 'expo-router';
router.push('/compose');
```

---

## 3. Drawer Navigation

```tsx
// app/_layout.tsx
import { GestureHandlerRootView } from 'react-native-gesture-handler';
import { Drawer } from 'expo-router/drawer';

export default function Layout() {
  return (
    <GestureHandlerRootView style={{ flex: 1 }}>
      <Drawer screenOptions={{ drawerType: 'slide' }}>
        <Drawer.Screen name="index" options={{ title: 'Home', drawerLabel: 'Home' }} />
        <Drawer.Screen name="notifications" options={{ title: 'Notifications' }} />
        <Drawer.Screen name="settings" options={{ title: 'Settings' }} />
      </Drawer>
    </GestureHandlerRootView>
  );
}
```

---

## 4. Shared routes between tabs

When Feed and Search tabs both need a user profile screen:

```
app/
└── (tabs)/
    ├── _layout.tsx
    ├── (feed,search)/         ← shared route group (cloned into both tabs)
    │   └── user/
    │       └── [id].tsx       → accessible as /user/:id from either tab
    ├── (feed)/
    │   └── index.tsx          → /  (feed tab home)
    └── (search)/
        └── index.tsx          → /  (search tab home)
```

Note: When deep-linking to a shared route from outside the app, Expo Router picks the first group alphabetically (in this case `feed`).

---

## 5. Full Auth Flow with Stack.Protected

```
app/
├── _layout.tsx       ← Root: Stack + SessionProvider
├── sign-in.tsx       ← Public
└── (app)/
    ├── _layout.tsx   ← Protected group layout
    └── (tabs)/
        ├── _layout.tsx
        ├── index.tsx
        └── profile.tsx
```

```tsx
// ctx.tsx (auth context)
import { createContext, use, type PropsWithChildren } from 'react';
import { useStorageState } from './useStorageState';

const AuthContext = createContext<{
  signIn: () => void;
  signOut: () => void;
  session?: string | null;
  isLoading: boolean;
}>({ signIn: () => null, signOut: () => null, session: null, isLoading: false });

export function SessionProvider({ children }: PropsWithChildren) {
  const [[isLoading, session], setSession] = useStorageState('session');
  return (
    <AuthContext value={{
      signIn: () => setSession('xxx-token'),
      signOut: () => setSession(null),
      session,
      isLoading,
    }}>
      {children}
    </AuthContext>
  );
}

export const useSession = () => use(AuthContext);
```

```tsx
// app/_layout.tsx
import { Stack } from 'expo-router';
import { SessionProvider, useSession } from '@/ctx';

function RootNavigator() {
  const { session, isLoading } = useSession();

  // Prevent flash of wrong screen while loading auth state
  if (isLoading) return null;

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

---

## 6. Platform-specific tabs

```tsx
// app/_layout.tsx
import AppTabs from './app-tabs'; // resolved via platform extension

export default function RootLayout() {
  return <AppTabs />;
}
```

```tsx
// app/app-tabs.native.tsx — iOS & Android
import { Tabs } from 'expo-router';
export default function AppTabs() {
  return <Tabs>{/* native tab config */}</Tabs>;
}
```

```tsx
// app/app-tabs.tsx — Web
export default function AppTabs() {
  return <>{/* custom web tab UI */}</>;
}
```

---

## 7. initialRouteName for deep link back navigation

```tsx
// app/(tabs)/_layout.tsx
export const unstable_settings = {
  initialRouteName: 'index',
};

export default function TabLayout() { ... }
```

When a user cold-launches via deep link to `myapp://profile/123`, the back button will take them to the tab index instead of exiting the app.
