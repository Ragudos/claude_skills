# CI/CD Integration

## GitHub Actions

### Basic Workflow Structure

```yaml
# .github/workflows/ci.yml
name: CI

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, devleop]
  workflow_dispatch:

permissions:
  contents: write

env:
  NODE_VERSION: "20"

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ env.NODE_VERSION }}
          cache: "npm"
      - run: npm ci
      - run: npm run build
```

### Complete CI Pipeline

```yaml
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

permissions:
  contents: write
  pull-requests: write

jobs:
  # ---------------------------------------------------------
  # 1. VERSIONING & RELEASE MANAGER
  # ---------------------------------------------------------
  release-please:
    runs-on: ubuntu-latest
    needs: [lint, security, test]
    if: github.ref == 'refs/heads/main'
    outputs:
      mobile_release_created: ${{ steps.release.outputs['apps/mobile--release_created'] }}
      mobile_version: ${{ steps.release.outputs['apps/mobile--version'] }}
    steps:
      - name: Run Release Please
        id: release
        uses: googleapis/release-please-action@v4
        with:
          config-file: release-please-config.json
          manifest-file: .release-please-manifest.json

  # ---------------------------------------------------------
  # 2. CODE QUALITY & SECURITY CHECKS (Runs on all packages)
  # ---------------------------------------------------------
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm
      - name: Lint all workspaces
        run: pnpm -r run lint

  security:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - name: Run security audit
        run: pnpm audit --audit-level=high || true

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          scan-type: "fs"
          ignore-unfixed: true
          format: "table"
          severity: "HIGH,CRITICAL"

  test:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - name: Run Tests with Coverage
        run: pnpm -r run test --coverage.enabled true --coverage.reporter="text" --coverage.reporter="json-summary" --coverage.reporter="html"

      - name: Vitest Coverage Comment
        if: github.event_name == 'pull_request'
        uses: davelosert/vitest-coverage-report-action@v2
        with:
          name: "📊 Vitest Code Coverage"

      - name: Archive Coverage Report
        uses: actions/upload-artifact@v4
        with:
          name: raw-coverage-data
          path: coverage/
          retention-days: 30

  # ---------------------------------------------------------
  # 3. STAGING TRACK (DEVELOP BRANCH)
  # ---------------------------------------------------------
  build-staging:
    runs-on: ubuntu-latest
    needs: [lint, security, test]
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - uses: expo/expo-github-action@v8
        with:
          eas-version: 18.8.1
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build staging (internal)
        working-directory: apps/mobile
        run: eas build --profile preview --platform all --non-interactive --wait
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}

  e2e-staging:
    runs-on: ubuntu-latest
    needs: build-staging
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - name: Setup EAS CLI
        uses: expo/expo-github-action@v8
        with:
          eas-version: 18.8.1
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Download APK from Expo
        working-directory: apps/mobile
        run: eas build:download --profile preview --platform android --non-interactive
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}

      - name: Setup Supabase Local
        uses: supabase/setup-cli@v1
        with:
          version: latest
      - run: supabase start

      - name: Run Maestro E2E Tests
        uses: reactivecircus/android-emulator-runner@v2
        with:
          api-level: 33
          target: google_apis
          arch: x86_64
          script: |
            curl -Ls "https://get.maestro.mobile.dev" | bash
            export PATH="$PATH":"$HOME/.maestro/bin"
            adb install apps/mobile/*.apk
            maestro test apps/mobile/.maestro/

      - name: Upload Maestro Logs (On Failure)
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: maestro-debug-logs
          path: ~/.maestro/tests/

  deploy-staging:
    runs-on: ubuntu-latest
    needs: e2e-staging
    if: github.ref == 'refs/heads/develop'
    environment: staging
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - uses: expo/expo-github-action@v8
        with:
          eas-version: 18.8.1
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Submit to internal track
        working-directory: apps/mobile
        run: eas submit --profile preview --platform all --non-interactive
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}

  # ---------------------------------------------------------
  # 4A. PRODUCTION OTA PATCHES (Triggered by fix: commits)
  # ---------------------------------------------------------
  ota-production:
    runs-on: ubuntu-latest
    needs: [lint, security, test, release-please]
    if: >
      github.ref == 'refs/heads/main' && 
      needs.release-please.outputs.mobile_release_created == 'true' && 
      !endsWith(needs.release-please.outputs.mobile_version, '.0')
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - uses: expo/expo-github-action@v8
        with:
          eas-version: 18.8.1
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Publish OTA Update
        working-directory: apps/mobile
        run: eas update --branch production --message "Release v${{ needs.release-please.outputs.mobile_version }}" --non-interactive
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}

  # ---------------------------------------------------------
  # 4B. PRODUCTION FULL BUILDS (Triggered by feat: or breaking commits)
  # ---------------------------------------------------------
  build-production:
    runs-on: ubuntu-latest
    needs: [lint, security, test, release-please]
    if: >
      github.ref == 'refs/heads/main' && 
      needs.release-please.outputs.mobile_release_created == 'true' && 
      endsWith(needs.release-please.outputs.mobile_version, '.0')
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - uses: expo/expo-github-action@v8
        with:
          eas-version: 18.8.1
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Build production binary
        working-directory: apps/mobile
        run: eas build --profile production --platform all --non-interactive --wait
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}

  deploy-production:
    runs-on: ubuntu-latest
    needs: build-production
    environment: production
    steps:
      - uses: actions/checkout@v4
      - uses: ./.github/actions/setup-pnpm

      - uses: expo/expo-github-action@v8
        with:
          eas-version: 18.8.1
          token: ${{ secrets.EXPO_TOKEN }}

      - name: Submit to app stores
        working-directory: apps/mobile
        run: eas submit --profile production --platform all --non-interactive
        env:
          EXPO_TOKEN: ${{ secrets.EXPO_TOKEN }}
```

### Reusable Workflows

```yaml
# .github/workflows/reusable-test.yml
name: Reusable Test Workflow

on:
  workflow_call:
    inputs:
      node-version:
        required: true
        type: string
    secrets:
      npm-token:
        required: false

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ inputs.node-version }}
      - run: npm ci
      - run: npm lint
      - run: npm test
```

```yaml
# .github/workflows/ci.yml
name: CI

on: [push, pull_request]

jobs:
  call-test:
    uses: ./.github/workflows/reusable-test.yml
    with:
      node-version: "20"
    secrets:
      npm-token: ${{ secrets.NPM_TOKEN }}
```

### Matrix Builds

```yaml
jobs:
  test:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, windows-latest, macos-latest]
        node: [18, 20, 22]
        exclude:
          - os: windows-latest
            node: 18
      fail-fast: false
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: ${{ matrix.node }}
      - run: npm ci
      - run: npm test
```

## Automated Testing

### Pre-merge Checks

```yaml
# .github/workflows/pr-checks.yml
name: PR Checks

on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  analyze:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Check PR size
        run: |
          LINES=$(git diff --numstat origin/main...HEAD | awk '{sum += $1 + $2} END {print sum}')
          if [ "$LINES" -gt 1000 ]; then
            echo "::warning::Large PR ($LINES lines). Consider splitting."
          fi

      - name: Check commit messages
        run: |
          git log origin/main..HEAD --pretty=format:"%s" | while read msg; do
            if ! echo "$msg" | grep -qE "^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)"; then
              echo "::error::Invalid commit message: $msg"
              exit 1
            fi
          done

  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npm test -- --coverage --changedSince=origin/main

  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npm run lint -- --max-warnings 0
```

## Best Practices

1. **Fast Feedback**: Keep CI under 10 minutes
2. **Parallel Jobs**: Run independent jobs concurrently
3. **Caching**: Cache dependencies and build artifacts
4. **Fail Fast**: Stop on first failure in PR checks
5. **Environment Parity**: Match CI environment to production
6. **Secrets Management**: Use encrypted secrets, rotate regularly
7. **Artifact Retention**: Clean up old artifacts
8. **Status Checks**: Require all checks to pass before merge
