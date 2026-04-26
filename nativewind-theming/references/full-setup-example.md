# Full Setup Example — NativeWind v4 Theme System

A minimal but complete Expo project wiring all pieces of the theme system together.

## File Tree

```
my-app/
├── app/
│   ├── _layout.tsx
│   └── index.tsx
├── components/
│   └── Card.tsx
├── theme/
│   ├── index.ts
│   ├── css-vars.ts
│   ├── use-theme.ts
│   └── color-scheme-store.ts
├── global.css
├── tailwind.config.ts
├── babel.config.js
└── metro.config.js
```

## babel.config.js

```js
module.exports = function (api) {
  api.cache(true);
  return {
    presets: [
      ["babel-preset-expo", { jsxImportSource: "nativewind" }],
      "nativewind/babel",
    ],
  };
};
```

## metro.config.js

```js
const { getDefaultConfig } = require("expo/metro-config");
const { withNativeWind } = require("nativewind/metro");

const config = getDefaultConfig(__dirname);

module.exports = withNativeWind(config, { input: "./global.css" });
```

## theme/index.ts

```ts
export const tokens = {
  colors: {
    background: "#ffffff",
    backgroundMuted: "#f4f4f5",
    foreground: "#09090b",
    foregroundMuted: "#71717a",
    primary: "#6d28d9",
    primaryForeground: "#ffffff",
    border: "#e4e4e7",
    destructive: "#ef4444",
    destructiveForeground: "#ffffff",

    dark: {
      background: "#09090b",
      backgroundMuted: "#18181b",
      foreground: "#fafafa",
      foregroundMuted: "#a1a1aa",
      primary: "#7c3aed",
      primaryForeground: "#ffffff",
      border: "#27272a",
      destructive: "#f87171",
      destructiveForeground: "#000000",
    },
  },
  spacing: {
    xs: "4px",
    sm: "8px",
    md: "16px",
    lg: "24px",
    xl: "32px",
  },
  radii: {
    sm: "4px",
    md: "8px",
    lg: "12px",
    full: "9999px",
  },
  typography: {
    fontSans: "Inter_400Regular",
    fontBold: "Inter_700Bold",
    textBase: 16,
    textLg: 18,
    text2xl: 24,
  },
} as const;
```

## theme/css-vars.ts

```ts
import { tokens } from "./index";

function toKebab(str: string) {
  return str.replace(/([A-Z])/g, "-$1").toLowerCase();
}

export function buildVarsObject(
  mode: "light" | "dark" = "light",
): Record<string, string> {
  const { dark, ...light } = tokens.colors;
  const colors = mode === "dark" ? dark : light;
  const result: Record<string, string> = {};

  for (const [key, val] of Object.entries(colors)) {
    result[`--color-${toKebab(key)}`] = val as string;
  }
  for (const [key, val] of Object.entries(tokens.spacing)) {
    result[`--spacing-${key}`] = val;
  }
  for (const [key, val] of Object.entries(tokens.radii)) {
    result[`--radius-${key}`] = val;
  }
  return result;
}
```

## theme/use-theme.ts

```ts
import { useColorScheme } from "react-native";
import { tokens } from "./index";

export function useTheme() {
  const { dark, ...light } = tokens.colors;
  const isDark = useColorScheme() === "dark";
  return {
    colors: isDark
      ? (dark as Record<string, string>)
      : (light as Record<string, string>),
    spacing: tokens.spacing,
    radii: tokens.radii,
    typography: tokens.typography,
    isDark,
  };
}
```

## app/\_layout.tsx

```tsx
import { useEffect, useState } from "react";
import { View, useColorScheme } from "react-native";
import { vars } from "nativewind";
import { Stack } from "expo-router";
import * as SplashScreen from "expo-splash-screen";
import { buildVarsObject } from "@/theme/css-vars";

SplashScreen.preventAutoHideAsync();

// Build once — not on every render
const lightThemeVars = vars(buildVarsObject("light"));
const darkThemeVars = vars(buildVarsObject("dark"));

export default function RootLayout() {
  const colorScheme = useColorScheme();
  const [ready, setReady] = useState(false);

  useEffect(() => {
    setReady(true);
  }, []);
  useEffect(() => {
    if (ready) SplashScreen.hideAsync();
  }, [ready]);

  if (!ready) return null;

  return (
    <View
      style={[
        { flex: 1 },
        colorScheme === "dark" ? darkThemeVars : lightThemeVars,
      ]}
    >
      <Stack screenOptions={{ headerShown: false }} />
    </View>
  );
}
```

## app/index.tsx

```tsx
import { View, Text } from "react-native";

export default function HomeScreen() {
  return (
    <View className="flex-1 bg-background items-center justify-center">
      <Text className="text-foreground text-2xl font-bold">
        Hello NativeWind v4
      </Text>
      <Text className="text-foreground-muted text-base mt-2">
        Theme tokens wired up ✓
      </Text>
      <View className="mt-6 px-6 py-3 bg-primary rounded-lg">
        <Text className="text-primary-foreground font-bold">
          Primary Button
        </Text>
      </View>
    </View>
  );
}
```

## components/Card.tsx (StyleSheet usage)

```tsx
import { View, StyleSheet } from "react-native";
import { useTheme } from "@/theme/use-theme";

export function Card({ children }: { children: React.ReactNode }) {
  const { colors, radii, spacing } = useTheme();

  return (
    <View
      style={[
        styles.base,
        {
          backgroundColor: colors.backgroundMuted,
          borderRadius: parseInt(radii.lg),
          padding: parseInt(spacing.md),
          borderColor: colors.border,
        },
      ]}
    >
      {children}
    </View>
  );
}

const styles = StyleSheet.create({
  base: {
    borderWidth: 1,
    shadowColor: "#000",
    shadowOpacity: 0.05,
    shadowRadius: 4,
    elevation: 2,
  },
});
```

## global.css

```css
@import "tailwindcss";
@import "nativewind/globals";

@theme {
  --color-background: initial;
  --color-background-muted: initial;
  --color-foreground: initial;
  --color-foreground-muted: initial;
  --color-primary: initial;
  --color-primary-foreground: initial;
  --color-border: initial;
  --color-destructive: initial;
  --color-destructive-foreground: initial;

  --spacing-xs: initial;
  --spacing-sm: initial;
  --spacing-md: initial;
  --spacing-lg: initial;
  --spacing-xl: initial;

  --radius-sm: initial;
  --radius-md: initial;
  --radius-lg: initial;
  --radius-full: initial;
}
```

## tailwind.config.ts

```ts
import type { Config } from "tailwindcss";
import { tokens } from "./theme/index";

const { dark, ...light } = tokens.colors;
const toKebab = (s: string) => s.replace(/([A-Z])/g, "-$1").toLowerCase();

export default {
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],
  presets: [require("nativewind/preset")],
  theme: {
    extend: {
      colors: Object.fromEntries(
        Object.keys(light).map((k) => [
          toKebab(k),
          `var(--color-${toKebab(k)})`,
        ]),
      ),
      spacing: Object.fromEntries(
        Object.keys(tokens.spacing).map((k) => [k, `var(--spacing-${k})`]),
      ),
      borderRadius: Object.fromEntries(
        Object.keys(tokens.radii).map((k) => [k, `var(--radius-${k})`]),
      ),
      fontFamily: {
        sans: [tokens.typography.fontSans],
        bold: [tokens.typography.fontBold],
      },
    },
  },
} satisfies Config;
```

## Required Packages

```bash
npx expo install nativewind tailwindcss
npx expo install expo-router expo-splash-screen
# If using color-scheme-store:
npx expo install zustand
```

## nativewind/types Reference

Add to `tsconfig.json` > `compilerOptions.types`:

```json
{
  "compilerOptions": {
    "types": ["nativewind/types"]
  }
}
```

This gives you autocomplete for `className` on all React Native core components.
