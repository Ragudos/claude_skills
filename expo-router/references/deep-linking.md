# Deep Linking & Universal Links

## Basic Setup

### 1. Configure scheme in app.json

```json
{
  "expo": {
    "scheme": "myapp",
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [{ "scheme": "myapp" }],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

Links into your app: `myapp://` → `/`, `myapp://settings/account` → `/settings/account`

---

## Universal Links (https:// scheme)

### iOS — apple-app-site-association

Host this file at `https://yourdomain.com/.well-known/apple-app-site-association`:

```json
{
  "applinks": {
    "apps": [],
    "details": [
      {
        "appID": "TEAMID.com.yourapp.bundle",
        "paths": ["*"]
      }
    ]
  }
}
```

In `app.json`:
```json
{
  "expo": {
    "ios": {
      "associatedDomains": ["applinks:yourdomain.com"]
    }
  }
}
```

### Android — assetlinks.json

Host at `https://yourdomain.com/.well-known/assetlinks.json`:

```json
[{
  "relation": ["delegate_permission/common.handle_all_urls"],
  "target": {
    "namespace": "android_app",
    "package_name": "com.yourapp",
    "sha256_cert_fingerprints": ["AA:BB:CC:..."]
  }
}]
```

In `app.json`:
```json
{
  "expo": {
    "android": {
      "intentFilters": [
        {
          "action": "VIEW",
          "autoVerify": true,
          "data": [{ "scheme": "https", "host": "yourdomain.com" }],
          "category": ["BROWSABLE", "DEFAULT"]
        }
      ]
    }
  }
}
```

---

## Testing Deep Links

```bash
# iOS Simulator
npx uri-scheme open myapp://settings/account --ios

# Android Emulator
npx uri-scheme open myapp://settings/account --android

# Using adb directly
adb shell am start -W -a android.intent.action.VIEW -d "myapp://settings/account"
```

---

## Handling Unmatched Native URLs (+native-intent)

Use when third-party services (e.g., email magic links, OAuth callbacks) send URLs that don't match your routes exactly.

```tsx
// app/+native-intent.tsx
import { router } from 'expo-router';

export function handleURL({ url }: { url: string }): boolean {
  // Handle magic link: myapp://auth?token=abc
  const parsed = new URL(url);
  if (parsed.pathname === '/auth' && parsed.searchParams.get('token')) {
    router.replace({
      pathname: '/sign-in',
      params: { token: parsed.searchParams.get('token')! },
    });
    return true; // handled
  }
  return false; // let Expo Router handle it normally
}
```

---

## Back Navigation After Cold Deep Link

Without `initialRouteName`, cold-launching from a deep link gives no back history.

```tsx
// app/_layout.tsx or app/(tabs)/_layout.tsx
export const unstable_settings = {
  initialRouteName: 'index', // or '(tabs)' at root level
};
```

This pre-populates the stack so the back button works after a cold deep link open.

---

## Common Pitfalls

| Problem | Cause | Fix |
|---|---|---|
| Deep link opens app but wrong screen | No route match | Add `+not-found.tsx`, check file path |
| No back button after deep link | Missing `initialRouteName` | Set `unstable_settings.initialRouteName` |
| Universal links open browser instead of app | AASA/assetlinks not served correctly | Must be served over HTTPS with correct Content-Type |
| Magic link params not received | Using `+native-intent` incorrectly | Return `true` from `handleURL` when handled |
