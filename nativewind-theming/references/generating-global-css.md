# Auto-Generating `global.css` from Token File

Keeping `global.css`'s `@theme` block in sync with `theme/index.ts` manually is error-prone.
This script generates the `@theme` block automatically, preventing drift.

## The Generator Script — `scripts/generate-css-theme.ts`

```ts
// scripts/generate-css-theme.ts
// Run: npx ts-node scripts/generate-css-theme.ts
import { writeFileSync } from 'fs';
import { tokens } from '../theme/index';

function toKebab(str: string): string {
  return str.replace(/([A-Z])/g, '-$1').toLowerCase();
}

function generateThemeBlock(): string {
  const { dark, ...light } = tokens.colors;
  const lines: string[] = [];

  // Color vars
  for (const key of Object.keys(light)) {
    lines.push(`  --color-${toKebab(key)}: initial;`);
  }

  lines.push('');

  // Spacing vars
  for (const key of Object.keys(tokens.spacing)) {
    lines.push(`  --spacing-${key}: initial;`);
  }

  lines.push('');

  // Radii vars
  for (const key of Object.keys(tokens.radii)) {
    lines.push(`  --radius-${key}: initial;`);
  }

  return lines.join('\n');
}

const output = `@import "tailwindcss";
@import "nativewind/globals";

/* AUTO-GENERATED — do not edit manually. Run: npx ts-node scripts/generate-css-theme.ts */
@theme {
${generateThemeBlock()}
}
`;

writeFileSync('./global.css', output, 'utf-8');
console.log('✓ global.css generated from theme/index.ts');
```

## Add as a package.json Script

```json
{
  "scripts": {
    "theme:sync": "npx ts-node scripts/generate-css-theme.ts",
    "prebuild": "npm run theme:sync"
  }
}
```

Running `npm run theme:sync` regenerates `global.css` from your tokens. The `prebuild` hook
ensures it's always fresh before an EAS build.

## Using with `ts-node`

```bash
npx expo install --dev ts-node
```

Or with `tsx` (faster, no tsconfig needed):

```bash
npx expo install --dev tsx
```
```json
"theme:sync": "npx tsx scripts/generate-css-theme.ts"
```

## What Gets Generated

Given the token file from the main skill, `global.css` will look like:

```css
@import "tailwindcss";
@import "nativewind/globals";

/* AUTO-GENERATED — do not edit manually. Run: npx ts-node scripts/generate-css-theme.ts */
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

## Validating Token / CSS Drift

Add a CI check to catch cases where someone edits `global.css` manually:

```ts
// scripts/check-css-theme.ts
import { execSync } from 'child_process';
import { readFileSync } from 'fs';

const before = readFileSync('./global.css', 'utf-8');
execSync('npx tsx scripts/generate-css-theme.ts');
const after = readFileSync('./global.css', 'utf-8');

if (before !== after) {
  console.error('❌ global.css is out of sync with theme/index.ts. Run npm run theme:sync');
  process.exit(1);
}
console.log('✓ global.css is in sync with theme/index.ts');
```

```json
"scripts": {
  "theme:check": "npx tsx scripts/check-css-theme.ts"
}
```

Add `npm run theme:check` to your CI pipeline.
