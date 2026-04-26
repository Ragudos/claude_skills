# Versioning Lifecycle

## When to Start

Do NOT version before alpha testing begins.
Start at `0.1.0-alpha.1` on first public/external prototype release.

## Pre-release Stages

| Stage  | Format          | Trigger                         |
| ------ | --------------- | ------------------------------- |
| Alpha  | `0.x.x-alpha.n` | First external tester           |
| Beta   | `0.x.x-beta.n`  | Feature complete, wider testing |
| RC     | `0.x.x-rc.n`    | Final pre-release validation    |
| Stable | `1.0.0`         | First public stable release     |

## Notes

- `MAJOR=0` signals unstable to all consumers
- Reset pre-release counter (`alpha.n`) when MINOR bumps
- No versioning during internal dev — no external consumer, no version
