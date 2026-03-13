# iOS Release Skill for Claude Code

A skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that provides complete iOS release management: build, test, fix errors, archive, upload to App Store Connect, version bumping, pre-release checks, and changelog generation.

## Features

| Command | Description |
|---------|-------------|
| `/ios-release status` | Current version, build, branch, tag |
| `/ios-release build` | Debug build + auto-fix errors |
| `/ios-release clean` | Clean DerivedData and build cache |
| `/ios-release test` | Run tests on simulator |
| `/ios-release bump-build` | Increment build number +1 |
| `/ios-release bump-version` | New version + reset build to 1 |
| `/ios-release tag` | Git tag current version |
| `/ios-release pre-release-check` | 11 parallel checks before release |
| `/ios-release archive` | Archive + export IPA for App Store Connect |
| `/ios-release upload` | Upload IPA to App Store Connect |
| `/ios-release release` | Full cycle: checks → bump → archive |
| `/ios-release changelog` | Generate "What's New" for App Store Connect |
| `/ios-release init` | Project setup and config |

## Key highlights

- **Fast incremental builds** — reuses Xcode IDE's DerivedData cache, builds only the active architecture. Incremental builds take seconds, not minutes.
- **Auto-fix build errors** — automatically fixes up to 10 errors per attempt (3 attempts max), grouped by file.
- **Parallel pre-release checks** — all 11 checks run simultaneously (~10s instead of ~120s).
- **Multi-locale changelog** — generates "What's New" text in all your app's languages from git history.
- **Smart localization check** — supports `.xcstrings` (Xcode 15+) with plural/variation forms, offers to auto-translate missing keys.

## Benchmark

Tested with [skill-creator](https://github.com/anthropics/claude-plugins-official/tree/main/skill-creator) eval on 8 real-world iOS release scenarios:

| Metric | With Skill | Without Skill |
|--------|-----------|---------------|
| Pass rate | **100%** (8/8) | 12.5% (1/8) |

Test cases: build, clean, bump-version, bump-build, pre-release-check, archive, changelog, status.

## Installation

Clone the repo into your Claude Code skills directory:

```bash
git clone https://github.com/Max-Fox/ios-release-skill.git ~/.claude/skills/ios-release
```

Skills in `~/.claude/skills/` are discovered automatically. Restart Claude Code and the `/ios-release` command will be available.

## Configuration

On first use, run `/ios-release init` to set up your project. This creates `.claude/ios-release.yml`:

```yaml
scheme: MyApp
simulator: iPhone 17
localizations: [en, ru]
archives_dir: ~/Archives
test_timeout: 300

extra_checks:
  has_subscriptions: false

rejection_history: []
```

All fields are optional — the skill auto-discovers what it can.

## Requirements

- macOS with Xcode installed
- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) CLI

## License

MIT
