---
name: nativewind-theme
description: >
  Expert guidance for setting up and maintaining a single-source-of-truth theme system in
  React Native Expo projects using NativeWind v4 (Tailwind CSS v4). Use this skill whenever
  the user mentions NativeWind, Tailwind in React Native, CSS variables in Expo, theming with
  nativewind, dark mode in Expo, or asks how to share design tokens between Tailwind config
  and TypeScript/StyleSheet code. Also trigger when the user has inconsistent theme values
  across their app, wants to inject CSS vars on app init, or asks how to use their Tailwind
  theme tokens in Animated or StyleSheet. Covers setup from scratch, migrating an existing
  theme, dark mode switching, and extending Tailwind config from a TS token file.
---

# NativeWind v4 — Single-Source-of-Truth Theme System

You are an expert on NativeWind v4 with Expo. The core principle: **one `theme/index.ts` file
owns all design tokens. CSS custom properties and `tailwind.config.ts` are both generated from
it. Nothing is hardcoded anywhere else.**

---

## 1. Architecture Overview

```
theme/
├── index.ts          ← Single source of truth (TS token object)
├── css-vars.ts       ← Generates CSS custom properties string from tokens
└── use-theme.ts      ← Hook for consuming tokens in StyleSheet / Animated

app/
└── _layout.tsx       ← Injects CSS vars via <vars()> on app init

global.css            ← @theme block + :root vars (generated/referenced)
tailwind.config.ts    ← Extends Tailwind from theme/index.ts tokens
```

**Data flow:**

```
theme/index.ts
    ↓               ↓                    ↓
global.css       tailwind.config.ts    useTheme() hook
(CSS vars)       (tw utility classes)  (StyleSheet/Animated)
    ↓
app/_layout.tsx
(vars() injection)
```

---

## 2. The Token File — `theme/index.ts`

This is the only file you ever edit when changing a color, spacing value, or font.

```ts
// theme/index.ts

export const tokens = {
  colors: {
    // Semantic names — map to raw values or palette aliases
    background: "#ffffff",
    backgroundMuted: "#f4f4f5",
    foreground: "#09090b",
    foregroundMuted: "#71717a",
    primary: "#6d28d9",
    primaryForeground: "#ffffff",
    secondary: "#e4e4e7",
    secondaryForeground: "#18181b",
    accent: "#f59e0b",
    accentForeground: "#ffffff",
    border: "#e4e4e7",
    ring: "#6d28d9",
    destructive: "#ef4444",
    destructiveForeground: "#ffffff",

    // Dark mode overrides — same keys, different values
    dark: {
      background: "#09090b",
      backgroundMuted: "#18181b",
      foreground: "#fafafa",
      foregroundMuted: "#a1a1aa",
      primary: "#7c3aed",
      primaryForeground: "#ffffff",
      secondary: "#27272a",
      secondaryForeground: "#fafafa",
      accent: "#f59e0b",
      accentForeground: "#000000",
      border: "#27272a",
      ring: "#7c3aed",
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
    "2xl": "48px",
    "3xl": "64px",
  },

  radii: {
    sm: "4px",
    md: "8px",
    lg: "12px",
    xl: "16px",
    full: "9999px",
  },

  typography: {
    fontSans: "Inter_400Regular",
    fontBold: "Inter_700Bold",
    // Scale in px — used in Tailwind config and StyleSheet
    textXs: 12,
    textSm: 14,
    textBase: 16,
    textLg: 18,
    textXl: 20,
    text2xl: 24,
    text3xl: 30,
    text4xl: 36,
  },
} as const;

export type Tokens = typeof tokens;
```

**Key rules:**

- All values are strings (for CSS vars) except `typography.*` sizes which are numbers (for StyleSheet).
- Use semantic names (e.g. `background`, not `white`). Palette aliases live in comments only.
- The `dark` block lives inside `colors` — not a separate top-level key.

---

## 3. CSS Variables Generator — `theme/css-vars.ts`

Converts the token object into a CSS custom-property string for injection.

```ts
// theme/css-vars.ts
import { tokens } from "./index";

type ColorTokens = Omit<typeof tokens.colors, "dark">;

function toKebab(str: string): string {
  return str.replace(/([A-Z])/g, "-$1").toLowerCase();
}

function buildColorVars(
  colors: Record<string, string>,
  prefix = "--color",
): string {
  return Object.entries(colors)
    .map(([key, val]) => `  ${prefix}-${toKebab(key)}: ${val};`)
    .join("\n");
}

function buildSpacingVars(): string {
  return Object.entries(tokens.spacing)
    .map(([key, val]) => `  --spacing-${key}: ${val};`)
    .join("\n");
}

function buildRadiiVars(): string {
  return Object.entries(tokens.radii)
    .map(([key, val]) => `  --radius-${key}: ${val};`)
    .join("\n");
}

const { dark, ...lightColors } = tokens.colors;

/** The full :root block (light mode) */
export const lightVars = `
:root {
${buildColorVars(lightColors as Record<string, string>)}
${buildSpacingVars()}
${buildRadiiVars()}
}`.trim();

/** The .dark class block (dark mode overrides) */
export const darkVars = `
.dark {
${buildColorVars(dark as Record<string, string>)}
}`.trim();

/**
 * Flat object for use with NativeWind's vars() function.
 * Pass this to a root <View style={vars(lightCSSVars)}> on init.
 */
export function buildVarsObject(
  colorMap: Record<string, string>,
  mode: "light" | "dark" = "light",
): Record<string, string> {
  const source = mode === "dark" ? dark : colorMap;
  const result: Record<string, string> = {};

  for (const [key, val] of Object.entries(source)) {
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

---

## 4. Tailwind Config — `tailwind.config.ts`

Extends Tailwind's theme **from** the token file. Never duplicate values here.

```ts
// tailwind.config.ts
import type { Config } from "tailwindcss";
import { tokens } from "./theme/index";

const { dark, ...lightColors } = tokens.colors;

function toKebab(str: string): string {
  return str.replace(/([A-Z])/g, "-$1").toLowerCase();
}

// Map token keys → CSS variable references
const colorAliases = Object.fromEntries(
  Object.keys(lightColors).map((key) => [
    toKebab(key),
    `var(--color-${toKebab(key)})`,
  ]),
) as Record<string, string>;

export default {
  content: ["./app/**/*.{ts,tsx}", "./components/**/*.{ts,tsx}"],
  presets: [require("nativewind/preset")],
  theme: {
    extend: {
      colors: colorAliases,
      spacing: Object.fromEntries(
        Object.entries(tokens.spacing).map(([k, v]) => [
          k,
          `var(--spacing-${k})`,
        ]),
      ),
      borderRadius: Object.fromEntries(
        Object.entries(tokens.radii).map(([k, v]) => [k, `var(--radius-${k})`]),
      ),
      fontFamily: {
        sans: [tokens.typography.fontSans],
        bold: [tokens.typography.fontBold],
      },
      fontSize: {
        xs: tokens.typography.textXs,
        sm: tokens.typography.textSm,
        base: tokens.typography.textBase,
        lg: tokens.typography.textLg,
        xl: tokens.typography.textXl,
        "2xl": tokens.typography.text2xl,
        "3xl": tokens.typography.text3xl,
        "4xl": tokens.typography.text4xl,
      },
    },
  },
} satisfies Config;
```

Now `className="bg-background text-foreground"` resolves through CSS vars → token values.

---

## 5. CSS Injection — `app/_layout.tsx`

NativeWind v4 uses `vars()` (from `nativewind`) to inject CSS custom properties onto a View.
Wrap the root layout in a View with `style={vars(cssVarsObject)}` so **all descendants inherit
the CSS variables**.

```tsx
// app/_layout.tsx
import { useEffect, useState } from "react";
import { View, useColorScheme } from "react-native";
import { vars } from "nativewind";
import { Stack } from "expo-router";
import * as SplashScreen from "expo-splash-screen";
import { buildVarsObject } from "@/theme/css-vars";
import { tokens } from "@/theme/index";

SplashScreen.preventAutoHideAsync();

const { dark, ...lightColors } = tokens.colors;

// Pre-build both var objects — no runtime computation on re-renders
const lightThemeVars = vars(
  buildVarsObject(lightColors as Record<string, string>, "light"),
);
const darkThemeVars = vars(
  buildVarsObject(lightColors as Record<string, string>, "dark"),
);

export default function RootLayout() {
  const colorScheme = useColorScheme();
  const [ready, setReady] = useState(false);

  useEffect(() => {
    // Put any async init here (fonts, auth, etc.)
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

**Why a root `<View>` wrapper?**  
`vars()` sets CSS custom properties on a native View node. All NativeWind-styled children
inherit them via the CSS cascade. Without this wrapper, `var(--color-background)` resolves
to nothing.

---

## 6. `global.css`

Required by NativeWind v4. The `@theme` block tells Tailwind CSS v4 about your custom
properties. Keep it in sync with token keys (or generate it — see reference file).

```css
/* global.css */
@import "tailwindcss";
@import "nativewind/globals";

@theme {
  --color-background: initial;
  --color-background-muted: initial;
  --color-foreground: initial;
  --color-foreground-muted: initial;
  --color-primary: initial;
  --color-primary-foreground: initial;
  --color-secondary: initial;
  --color-secondary-foreground: initial;
  --color-accent: initial;
  --color-accent-foreground: initial;
  --color-border: initial;
  --color-ring: initial;
  --color-destructive: initial;
  --color-destructive-foreground: initial;

  --spacing-xs: initial;
  --spacing-sm: initial;
  --spacing-md: initial;
  --spacing-lg: initial;
  --spacing-xl: initial;
  --spacing-2xl: initial;
  --spacing-3xl: initial;

  --radius-sm: initial;
  --radius-md: initial;
  --radius-lg: initial;
  --radius-xl: initial;
  --radius-full: initial;
}
```

`initial` signals "I'll define this elsewhere" — the actual values come from `vars()` at runtime.

---

## 7. `useTheme` Hook — Tokens in StyleSheet / Animated

For code that can't use Tailwind class names (StyleSheet, Animated, third-party components):

```ts
// theme/use-theme.ts
import { useColorScheme } from "react-native";
import { tokens } from "./index";

const { dark, ...lightColors } = tokens.colors;

export function useTheme() {
  const scheme = useColorScheme();
  const isDark = scheme === "dark";

  const colors = isDark
    ? (dark as Record<string, string>)
    : (lightColors as Record<string, string>);

  return {
    colors,
    spacing: tokens.spacing,
    radii: tokens.radii,
    typography: tokens.typography,
    isDark,
  };
}
```

**Usage:**

```tsx
import { StyleSheet, View } from "react-native";
import { useTheme } from "@/theme/use-theme";

export function Card({ children }: { children: React.ReactNode }) {
  const { colors, radii, spacing } = useTheme();

  const styles = StyleSheet.create({
    card: {
      backgroundColor: colors.backgroundMuted,
      borderRadius: parseInt(radii.lg),
      padding: parseInt(spacing.md),
      borderWidth: 1,
      borderColor: colors.border,
    },
  });

  return <View style={styles.card}>{children}</View>;
}
```

**For Animated values** — use tokens directly (no hook needed, values are static):

```ts
import { tokens } from "@/theme/index";
const { dark, ...light } = tokens.colors;
// Reference light.primary or dark.primary directly in Animated.timing toValue
```

---

## 8. Dark Mode Switching

The root layout's `useColorScheme()` drives the `vars()` swap automatically. For **manual
override** (user toggle):

```tsx
// theme/color-scheme-store.ts
import { create } from "zustand";
import { Appearance } from "react-native";

type Scheme = "light" | "dark" | "system";

interface ColorSchemeStore {
  preference: Scheme;
  setPreference: (s: Scheme) => void;
}

export const useColorSchemeStore = create<ColorSchemeStore>((set) => ({
  preference: "system",
  setPreference: (preference) => {
    set({ preference });
    if (preference !== "system") {
      Appearance.setColorScheme(preference); // propagates to useColorScheme()
    } else {
      Appearance.setColorScheme(null); // reset to system
    }
  },
}));
```

```tsx
// In _layout.tsx — read preference, fall back to system
import { useColorSchemeStore } from '@/theme/color-scheme-store';

const { preference } = useColorSchemeStore();
const systemScheme = useColorScheme();
const activeScheme = preference === 'system' ? systemScheme : preference;

// Then:
style={[{ flex: 1 }, activeScheme === 'dark' ? darkThemeVars : lightThemeVars]}
```

---

## 9. Common Pitfalls

| Problem                                 | Cause                                                 | Fix                                                             |
| --------------------------------------- | ----------------------------------------------------- | --------------------------------------------------------------- |
| `var(--color-*)` renders as nothing     | `vars()` not applied to root View                     | Wrap root in `<View style={vars(...)}>` in `_layout.tsx`        |
| Dark mode not updating                  | `useColorScheme()` used inside a non-reactive context | Move scheme check into a component using the hook               |
| StyleSheet values look wrong            | Using CSS string `'16px'` directly in StyleSheet      | Use `parseInt(tokens.spacing.md)` or store px values as numbers |
| Tailwind class `bg-primary` not working | Key not in `@theme` block in `global.css`             | Add `--color-primary: initial;` to `global.css` `@theme`        |
| Type errors on `tokens.colors.dark.*`   | `dark` is nested under `colors`                       | Destructure: `const { dark, ...light } = tokens.colors`         |
| vars() object recomputed every render   | Built inline in JSX                                   | Pre-build outside component: `const lightVars = vars(...)`      |

---

## 10. Quick-Start Checklist

- [ ] Create `theme/index.ts` with all tokens
- [ ] Create `theme/css-vars.ts` with `buildVarsObject`
- [ ] Create `theme/use-theme.ts` hook
- [ ] Update `tailwind.config.ts` to import from token file
- [ ] Add all token keys to `global.css` `@theme` block with `initial`
- [ ] Wrap root in `<View style={vars(...)}>` in `app/_layout.tsx`
- [ ] Verify dark mode: swap `darkThemeVars` in and check colors update

---

## Reference Files

- `references/full-setup-example.md` — Complete minimal Expo project wiring all pieces together
- `references/generating-global-css.md` — Script to auto-generate `global.css` from token file (prevents drift)

Read these when the user wants a full working example or asks about automating token-to-CSS sync.
