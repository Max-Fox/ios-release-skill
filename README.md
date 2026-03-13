# iOS Release Skill for Claude Code

A skill for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that provides complete iOS release management: build, test, fix errors, archive, upload to App Store Connect, version bumping, pre-release checks, and changelog generation.

## Features

| Command | Description |
|---------|-------------|
| `/ios status` | Current version, build, branch, tag |
| `/ios build` | Debug build + auto-fix errors |
| `/ios clean` | Clean DerivedData and build cache |
| `/ios test` | Run tests on simulator |
| `/ios bump-build` | Increment build number +1 |
| `/ios bump-version` | New version + reset build to 1 |
| `/ios tag` | Git tag current version |
| `/ios pre-release-check` | 11 parallel checks before release |
| `/ios archive` | Archive + export IPA for App Store Connect |
| `/ios upload` | Upload IPA to App Store Connect |
| `/ios release` | Full cycle: checks → bump → archive |
| `/ios changelog` | Generate "What's New" for App Store Connect |
| `/ios init` | Project setup and config |

## Key highlights

- **Fast incremental builds** — reuses Xcode IDE's DerivedData cache, builds only the active architecture. Incremental builds take seconds, not minutes.
- **Auto-fix build errors** — automatically fixes up to 10 errors per attempt (3 attempts max), grouped by file.
- **Parallel pre-release checks** — all 11 checks run simultaneously (~10s instead of ~120s).
- **Multi-locale changelog** — generates "What's New" text in all your app's languages from git history.
- **Smart localization check** — supports `.xcstrings` (Xcode 15+) with plural/variation forms, offers to auto-translate missing keys.

## Installation

### Quick install

```bash
claude skill add --from Max-Fox/ios-release-skill
```

### Manual install

1. Clone or download this repo
2. Copy the `SKILL.md` and `references/` folder to your Claude Code skills directory:

```bash
mkdir -p ~/.claude/skills/ios
cp SKILL.md ~/.claude/skills/ios/
cp -r references ~/.claude/skills/ios/
```

3. Register the skill in your Claude Code settings (`~/.claude/settings.json`):

```json
{
  "permissions": {
    "additionalDirectories": [
      "~/.claude/skills/ios"
    ]
  }
}
```

## Configuration

On first use, run `/ios init` to set up your project. This creates `.claude/ios-release.yml`:

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
