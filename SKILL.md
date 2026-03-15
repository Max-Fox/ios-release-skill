---
name: ios-release
argument-hint: "[status|build|clean|test|pre-release-check|bump-build|bump-version|archive|upload|release|tag|changelog|init]"
description: >
  Complete iOS release management: build, test, fix errors, archive, upload to
  App Store Connect, version/build bumping, pre-release checks, changelog generation.
  Use this skill whenever the user mentions anything related to iOS development workflow:
  building the project, fixing build errors, running tests, preparing for release,
  archiving, uploading IPA, bumping version or build number, creating git tags,
  generating "What's New" text, or checking project status. Also triggers on Russian:
  "собери", "исправь ошибки", "архив", "отправь в App Store", "проверь перед релизом",
  "релиз", "что нового", "статус". Even if the user just says something like
  "it doesn't build", "почему не собирается", "подготовь к релизу", or
  "ошибки при сборке" — this skill should handle it.
---

# iOS Release Skill

## Before Every Command

Read `.claude/ios-release.yml` before executing anything.
If no config exists — run Discovery, then offer `init` to create one.

---

## Discovery

When project paths are unknown, find them by reading the filesystem:

- **Workspace/project:** prefer `.xcworkspace` (exclude ones nested inside `.xcodeproj/`), fall back to `.xcodeproj`. Store the actual filename as `{workspace}` (e.g. `MyApp.xcworkspace`) — it may differ from the scheme name.
- **Info.plist:** main app target only — exclude paths containing `Test`, `DerivedData`, `Pods`, `Frameworks`, `.build`
- **Scheme:** run `xcodebuild -list`, pick the scheme whose name matches the project/workspace name. If no exact match — pick the first scheme that is not a test scheme (no `Test`, `Tests`, `UITests` suffix). If still ambiguous — ask the user.
- **Simulator:** use config value first; if missing or unavailable, pick the newest available iPhone from `xcrun simctl list devices available`
- **Build system:** read `CFBundleVersion` from Info.plist — if value is `$(CURRENT_PROJECT_VERSION)`, use `agvtool`; if literal number, use `PlistBuddy`
- **Xcode version:** `xcodebuild -version` — affects upload tool choice

---

## Config (`.claude/ios-release.yml`)

```yaml
scheme: MyApp
simulator: iPhone 17
# Locale codes in Xcode format: en, ru, zh-Hans, fr, de, ja, etc.
# Если указана одна локаль — проверки локализации пропускаются.
# Добавь все поддерживаемые локали, чтобы pre-release-check проверял переводы.
localizations: [en]
bundle_id_prefix: com.mycompany.myapp
archives_dir: ~/Desktop/ios-archives
test_timeout: 300              # таймаут тестов в секундах (по умолчанию 300)

extra_checks:
  has_subscriptions: false   # enable Guideline 3.1.2 checks (privacy_policy + terms_of_use links)

rejection_history: []        # shown as reminders at end of pre-release-check
```

All fields optional — discovery fills in missing values.

**After creating config:** if `archives_dir` or other paths are personal, add
`ios-release.yml` to `.gitignore`. If settings are shared, commit to repo.
Never commit without asking the user.

---

## No Argument — Help

If invoked without a command argument (`/ios` with no args), show this table:

```
📱 iOS Release Skill — команды
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

  📋 Информация
  status             Текущая версия, билд, ветка, тег

  🔨 Сборка
  build              Debug-сборка + автоисправление ошибок
  clean              Очистка DerivedData и кеша сборки
  test               Запуск тестов на симуляторе

  🔢 Версионирование
  bump-build         Инкремент билда +1
  bump-version       Новая версия + сброс билда на 1
  tag                Git-тег текущей версии

  🚀 Релиз
  pre-release-check  Основные проверки перед релизом
  archive            Архив + экспорт в App Store Connect
  upload             Загрузка готового IPA в App Store Connect
  release            Полный цикл: проверки → bump → архив
  changelog          Генерация What's New для App Store Connect

  ⚙️ Настройка
  init               Настройка проекта и конфига

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Использование: /ios-release <команда>
```

---

## Commands

### `status`
Быстрый обзор текущего состояния проекта. Выводит:
1. Текущая версия и build number (из Info.plist или agvtool)
2. Последний git-тег (`git tag --sort=-creatordate | head -1`)
3. Текущая ветка и чистота git (`git status --porcelain`)
4. Scheme и simulator из конфига

**Формат вывода:**
```
📱 {scheme} — статус
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Версия:       2.1.0 (build 48)
Последний тег: v2.0.0
Ветка:        main (чисто / 3 изменения)
Scheme:       MyApp
Simulator:    iPhone 17
━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

### `build`
Run a Debug build for simulator with **incremental compilation** — reusing
Xcode's own DerivedData so repeated builds are fast (seconds, not minutes).
Use `-workspace` if a `.xcworkspace` exists, otherwise use `-project`:
```
# With workspace (preferred when .xcworkspace exists):
xcodebuild build -workspace {workspace} -scheme {scheme} \
  -destination 'platform=iOS Simulator,name={simulator}' \
  -configuration Debug \
  ONLY_ACTIVE_ARCH=YES

# With project only:
xcodebuild build -project {project}.xcodeproj -scheme {scheme} \
  -destination 'platform=iOS Simulator,name={simulator}' \
  -configuration Debug \
  ONLY_ACTIVE_ARCH=YES
```
Key flags that make CLI builds fast:
- `ONLY_ACTIVE_ARCH=YES` → builds only the needed architecture, not all
- `-destination` instead of `-sdk` → better build system caching
- Do NOT pass `-derivedDataPath` — without it xcodebuild automatically
  reuses Xcode IDE's DerivedData at `~/Library/Developer/Xcode/DerivedData/`,
  giving full incremental build speed. Passing it explicitly can cause
  permission errors or break cache sharing.

Report `** BUILD SUCCEEDED **` with warning count (only if > 0),
or list all `error:` lines with file and line number.

**If build fails — automatically run `fix` logic** (see `fix` section below).
Do not ask — just start fixing. This makes `build` a single command that
builds and fixes in one step.

**Примечание:** если сборка падает с ошибками про отсутствие архитектуры или
unsupported framework — возможно, проект использует фреймворки, несовместимые
с симулятором (например, hardware-зависимые SDK). Сообщи пользователю и
предложи собрать для устройства: `-destination 'generic/platform=iOS'` с `CODE_SIGNING_ALLOWED=NO`.

### `clean`
Run in sequence (use `-workspace` or `-project` same as `build`):
```
xcodebuild clean -workspace {workspace} -scheme {scheme}
find $HOME/Library/Developer/Xcode/DerivedData -maxdepth 1 -name "{scheme}*" -exec rm -rf {} +
```
Report how many DerivedData folders were deleted (0 is fine — say so explicitly).

After cleaning, ask: "Очистить также кеш SPM? (может помочь при проблемах с пакетами)"
If yes:
```
rm -rf $HOME/Library/Caches/org.swift.swiftpm
```

Suggest running `build` afterwards to verify. Use when seeing strange errors,
stale caches, or after dependency changes.

### `fix`
Iteratively fix build errors. Max 3 attempts, max 10 errors per attempt.
If more than 10 errors — show full list, fix first 10, ask whether to continue.

**Group errors by file.** For each file with errors: open it once, apply all
fixes in a single edit, save and close. Never open the same file twice in one attempt.

If fixing one file causes new errors in another file — those count as a new
attempt. The 3-attempt limit counts from the start of the entire `fix` session,
not per chain of cascading errors. Report: "Исправлено X, но появилось N новых
ошибок в [file]. Начинаю попытку 2 из 3."

**Fix automatically:** syntax errors, type mismatches, missing returns,
undeclared identifiers.

**Deprecated API replacements:** fix only if the replacement API is available
within the project's minimum deployment target. Check `IPHONEOS_DEPLOYMENT_TARGET`
in the `.pbxproj` file before replacing. If the replacement requires a higher OS
version — report to user instead of fixing.

**Before editing any file:** read ±10 lines of context around each error.

**Never fix — report to user instead:**
- Signing / provisioning errors — "Настрой в Xcode → Target → Signing"
- Missing module / package — "Добавь зависимость в Package.swift или Podfile"
- Swift Macro errors — часто false positives; предложи `clean` первым
- Any error in `DerivedData/`, `Pods/`, `.build/`, or `*.generated.swift`

After each attempt: report errors found, what was fixed, what was skipped,
then re-run build. After 3 attempts with remaining errors, show what's left and stop.

### `test`
Run tests for simulator. Use `-workspace` or `-project` same as `build`.
Use timeout from config (`test_timeout`, default 300 seconds).

**Note:** macOS does not have `timeout` (it's GNU coreutils). Use `perl` as a
portable alternative:
```
# With workspace:
perl -e 'alarm {test_timeout}; exec @ARGV' -- \
  xcodebuild test -workspace {workspace} -scheme {scheme} \
  -destination 'platform=iOS Simulator,name={simulator}' \
  ONLY_ACTIVE_ARCH=YES

# With project only:
perl -e 'alarm {test_timeout}; exec @ARGV' -- \
  xcodebuild test -project {project}.xcodeproj -scheme {scheme} \
  -destination 'platform=iOS Simulator,name={simulator}' \
  ONLY_ACTIVE_ARCH=YES
```
`xcodebuild test -destination` boots the simulator automatically —
do not run `simctl boot` manually.
If tests time out, suggest the user check for network calls or infinite loops,
or increase `test_timeout` in config.

Report pass/fail count and details of each failure (test name + reason).
If no tests exist in the scheme, say so clearly.

### `pre-release-check`
Load config first.

**⚡ Parallel execution is critical for speed.** All 12 checks are independent.
Launch them ALL in parallel using multiple tool calls in a single message:
- Start the Debug build (check 12) FIRST — it's the slowest (~90s)
- In the SAME message, launch all static checks (1–11) as parallel tool calls
- This way total time ≈ Release build time, not the sum of all checks
- After all checks complete, collect results and print the summary table

Do NOT run checks sequentially — that wastes minutes of user's time.

**1. Deployment target**
Read `IPHONEOS_DEPLOYMENT_TARGET` from `.pbxproj`. If < 16.0 — ⚠️
"Deployment target {version} — Apple может потребовать минимум 16.0 для новых обновлений."
Информационный пункт, не блокирующий. Если ≥ 16.0 — ✅.

**2. Permission strings**
Grep `Info.plist` for all keys ending in `UsageDescription` — check every
one of them, not just a predefined list. Flag any key with a missing or
empty string value. Common examples: NSMicrophoneUsageDescription,
NSSpeechRecognitionUsageDescription, NSCameraUsageDescription,
NSPhotoLibraryUsageDescription, NSLocationWhenInUseUsageDescription,
NSBluetoothAlwaysUsageDescription, NSUserTrackingUsageDescription.

If `localizations` has more than one entry, also check `InfoPlist.strings`
in each `{locale}.lproj` — verify the same keys exist and are non-empty
in every locale listed in config.

**3. Localization completeness**
If `localizations` has only one entry — пропусти этот шаг, выведи
"⏭ Локализация — одна локаль ({locale}), проверка пропущена".

If no `.xcstrings` or `.strings` files found in the project — предложи:
"Файлы локализации не найдены. Создать Localizable.xcstrings? (.xcstrings —
актуальный формат с Xcode 15, поддерживает plural, device variations и
визуальный редактор в Xcode)."
If user agrees — create a minimal `Localizable.xcstrings` in the main app
target directory with `sourceLanguage` matching the first entry in
`localizations` config. The file must be added to the Xcode project manually
by the user (drag into Xcode), mention this after creation.

If `localizations` has more than one entry:

Detect the string catalog format for each locale:
- `.xcstrings` format (Xcode 15+): parse as JSON. A key is considered
  translated if it has a `localizations` entry for that locale with EITHER:
  - `stringUnit` with `state` != `"needs_review"` and a non-empty `value`, OR
  - `variations` (e.g. plural forms) — any key with `variations` is considered
    translated, because Xcode stores plural/device variations in this format
  Keys with no entry for a locale, or with `state: "needs_review"`, count as missing.
  Keys in the source locale (first in `localizations`) that have no explicit
  localization entry are normal — Xcode uses the key itself as the value.
  Do NOT count source locale missing keys.
- `.strings` format (classic): parse `"key" = "value";` lines.
  A key is missing if it has no entry in that locale file.

Using the base locale (first entry in `localizations`) as reference:
- For each other locale: find keys missing or needing review
- Report per locale. Zero missing = ✅. Any missing = ⚠️ with list (max 10 shown).

**Important:** when fixing translations, use the Edit tool for targeted changes
in `.xcstrings` files. Do NOT rewrite the entire file with `json.dump` — Xcode
uses its own JSON formatting and a full rewrite creates massive diffs (thousands
of lines) that break git clients.
- This check does not verify translation quality — only that keys exist and are not flagged.

**If any locale has missing keys:** спроси пользователя
"Есть непереведённые ключи. Доперевести автоматически или оставить как есть?"
Если пользователь согласен — перевести недостающие ключи на нужные локали
и записать в соответствующие `.xcstrings` / `.strings` файлы. Затем повторить
проверку локализации и обновить результат.

**4. Placeholder URLs**
Grep Swift and plist files for `example.com`, `YOUR_URL`, `INSERT_URL`,
`TODO`, `FIXME` near privacy/terms keywords.

**5. Subscription compliance (Guideline 3.1.2)**
If `extra_checks.has_subscriptions` is `true`:
- Grep Swift files for references to `privacy_policy`, `privacyPolicy`,
  `privacy policy`, `PrivacyPolicy` — verify at least one real URL exists
  (not a placeholder). ⛔ if not found.
- Grep Swift files for references to `terms_of_use`, `termsOfUse`,
  `terms of use`, `TermsOfUse`, `eula`, `EULA` — verify at least one real
  URL exists (not a placeholder). ⛔ if not found.
- Also check if the app uses `SubscriptionStoreView` (SwiftUI built-in) —
  if yes, note that Apple handles terms/privacy display automatically, but
  the URLs must still be configured in App Store Connect.

If `has_subscriptions` is `false` or not set — skip with
"⏭ Подписки — не настроены, проверка пропущена".

**6. IAP product IDs**
If `bundle_id_prefix` is set, grep Swift files for strings starting with
`{bundle_id_prefix}.` (exclude commented lines and test/mock/preview files).
Цель: убедиться что продуктовые ID присутствуют в коде и их количество
совпадает с ожидаемым. Выведи найденные ID списком. Если ни одного не
найдено — ⚠️ предупреди (возможно, IAP ещё не настроены или ID захардкожены
в другом формате).

**7. Debug-код**
Grep Swift files for `print(`, `debugPrint(`, `NSLog(` that are NOT inside
`#if DEBUG` / `#endif` blocks. Exclude test/mock/preview files.
Для проверки: прочитай файл и убедись что вызов находится вне `#if DEBUG`
блока. Если найдены — ⚠️ с количеством и списком файлов (max 10).
Это не блокер, но мусор в консоли = непрофессионально.

**8. Хардкод API-ключей**
Grep Swift files for patterns that look like hardcoded secrets:
- Strings starting with `sk-`, `sk_live_`, `sk_test_`, `pk_live_`, `pk_test_`
- Strings starting with `key_`, `api_key`, `apiKey` assigned to a string literal
- Long base64-like strings (40+ chars of `[A-Za-z0-9+/=]`) assigned to constants
Exclude: comments, test/mock/preview files, Info.plist keys, `UserDefaults` keys.
If found — ⛔ "Найден хардкод ключа в {file}:{line}. Используй Keychain или .xcconfig."

**9. App Icon**
Check that AppIcon asset exists and is not empty:
- Find `Assets.xcassets/AppIcon.appiconset/Contents.json`
- Parse JSON — verify at least one `filename` entry exists (not all empty)
If missing or empty — ⛔ "AppIcon не найден или пустой."

**10. Git state**
Warn if uncommitted changes or not on main/master/release branch.

**11. Liquid Glass**
Grep Swift files for `.glassEffect`. Exclude test/mock/preview files.
- If found — ✅ "Liquid Glass — используется ({count} файлов). Новый стиль Apple iOS 26."
- If not found — ℹ️ "Liquid Glass — не используется. Рассмотри `.glassEffect()` для соответствия новому стилю iOS 26."
Информационный пункт — не блокер и не warning.

**12. Debug build** — verify code compiles, launch in parallel with checks 1–11
Use the same command as `build` (Debug, simulator). A full Release build is
redundant here because `archive` already does one — no need to build Release twice.
```
xcodebuild build -project {project}.xcodeproj -scheme {scheme} \
  -destination 'platform=iOS Simulator,name={simulator}' \
  -configuration Debug \
  ONLY_ACTIVE_ARCH=YES
```

Print `rejection_history` from config as reminders after the table.

**Output format:**
```
Pre-Release Check — MyApp v2.1.0 (build 48)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
✅ Deployment target        — iOS 17.0
✅ Permission strings       — все ключи на месте (en, ru)
✅ Локализация              — все локали переведены (en, ru, de)
✅ Placeholder URLs         — не найдены
✅ Подписки (3.1.2)         — privacy_policy ✓, terms_of_use ✓
⏭ IAP product IDs          — bundle_id_prefix не задан
⚠️  Debug-код                — 12 вызовов print() в 4 файлах
✅ API-ключи                — хардкод не найден
✅ App Icon                 — AppIcon.appiconset OK
✅ Git                      — чисто, ветка: main
✅ Liquid Glass              — используется (3 файлов)
✅ Debug build
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
⛔ 0 ошибок  ⚠️ 1 предупреждение

📋 Напоминания из истории отклонений:
  • 3.1.1 — Отправляй IAP-продукты вместе с билдом
```

### `bump-build`
Detect build system first, then increment build number by 1:
- `agvtool` — `agvtool next-version -all`
- `PlistBuddy` — read `CFBundleVersion`; parse only the integer part before
  the first dot (e.g. `"47.0"` → `47`, `"1.0.0"` → `1`); increment by 1;
  write back as a plain integer string (e.g. `"48"`)

### `bump-version`
Ask for new version if not provided. Accept format X.X or X.X.X (two or three
dot-separated integers, each ≥ 0) — reject anything else and ask again.

Detect build system, then update marketing version:

**Important:** `agvtool` updates only Info.plist files, but NOT `MARKETING_VERSION`
in `project.pbxproj`. If the project uses `MARKETING_VERSION` in build settings
(check with `grep MARKETING_VERSION *.xcodeproj/project.pbxproj`), you MUST also
update it directly in `project.pbxproj` using replace-all on the old value.
Otherwise Xcode will use the old version from pbxproj, ignoring Info.plist.

Steps:
1. Run `agvtool new-marketing-version X.X`
2. **Then** replace `MARKETING_VERSION = {old_version};` with
   `MARKETING_VERSION = {new_version};` in `project.pbxproj` (all occurrences)
3. If project has multiple targets, предупреди пользователя и спроси
   подтверждение. Если пользователь откажется — используй `PlistBuddy`
   для обновления только Info.plist основного таргета.
- `PlistBuddy` fallback — update `CFBundleShortVersionString` in Info.plist

**After updating version — reset build number to 1** (not increment):
- `agvtool` — `agvtool new-version -all 1`
- `PlistBuddy` — set `CFBundleVersion` to `"1"`

Это стандартная практика: новая версия начинается с билда 1.

### `archive`
1. Запусти `status` — покажи текущую версию
2. Спроси: "Что поднимаем — версию или билд?"
   - **Версию** → запусти `bump-version` (запросит номер, сбросит билд на 1)
   - **Билд** → запусти `bump-build` (инкремент +1)
   - Пользователь может отказаться от обоих — тогда оставить как есть
3. Ensure `ExportOptions.plist` exists — if not, create with
   `method: app-store-connect`, `destination: upload`, `signingStyle: automatic`.
   Tell user to consider committing this file (do not commit automatically)
4. Archive (use `-workspace` or `-project` same as `build`):
   ```
   xcodebuild archive -workspace {workspace} -scheme {scheme} \
     -configuration Release -allowProvisioningUpdates -sdk iphoneos \
     -archivePath {expanded_archives_dir}/{scheme}-v{version}-b{build}.xcarchive
   ```
5. Export IPA:
   ```
   xcodebuild -exportArchive \
     -archivePath {expanded_archives_dir}/{scheme}-v{version}-b{build}.xcarchive \
     -exportOptionsPlist ExportOptions.plist \
     -exportPath {expanded_archives_dir}/{scheme}-v{version}-b{build}/ \
     -allowProvisioningUpdates
   ```
   Where `{expanded_archives_dir}` = `archives_dir` from config with `~`
   replaced by `$HOME` (e.g. `~/Desktop/ios-archives` → `$HOME/Desktop/ios-archives`).
6. Confirm IPA exists, print its path and size in MB.
   If IPA > 200 MB — ⚠️ "IPA больше 200 МБ — пользователи не смогут скачать по cellular без Wi-Fi."
   If IPA > 4 GB — ⛔ "IPA больше 4 ГБ — App Store Connect не примет."

Check for `** ARCHIVE SUCCEEDED **` and `** EXPORT SUCCEEDED **` in output —
xcodebuild wraps all status messages in double asterisks.

### `upload`
Read `references/upload-guide.md` for detailed upload instructions.

In short: find the latest `.ipa` in `archives_dir`, confirm with user,
then try altool → Transporter CLI → manual fallback, stopping at first success.

### `release`
Полный цикл выпуска. Каждый шаг выполняется один раз — без дублирования.
Порядок: сначала проверки, потом bump, потом сборка — чтобы при ошибке
версия осталась нетронутой.

1. Запусти `status` — покажи текущую версию
2. Run `pre-release-check` — stop on ⛔, ask on ⚠️
3. Run `test` — if tests exist and there are failures, ask whether to continue.
   If no test target configured — show "⏭ Тесты — схема не настроена, пропущено"
   and continue without stopping.
4. Спроси: "Что поднимаем — версию или билд?"
   - **Версию** → запусти `bump-version` (запросит номер, сбросит билд на 1)
   - **Билд** → запусти `bump-build` (инкремент +1)
   - Пользователь может отказаться — тогда оставить как есть
5. Ensure `ExportOptions.plist` exists (create if missing, same as `archive`)
6. Archive + export (same xcodebuild commands as `archive`)
7. Print summary: version, build, archive path, upload status.
   Примечание: билды появляются в App Store Connect через 15–60 минут.
8. Спроси: "Хочешь сгенерировать changelog для App Store Connect (What's New)?"
   If yes — run `changelog`.
9. Спроси: "Закоммитить изменения версии? (bump файлы)"
   If yes — `git add -A && git commit -m "v{version} (build {build})"`
10. Спроси: "Поставить тег v{version}?"
    If yes — `git tag v{version}`
11. Спроси: "Запушить коммит и тег в remote?"
    If yes — `git push && git push origin v{version}`

Шаги 9–11 каждый требует подтверждения. Если пользователь отказался на любом —
пропустить и перейти к следующему.

### `tag`
Read current version and build number from the project. Then ask:
"Поставить тег v{version} на текущий коммит? (build {build})"

If confirmed:
```
git tag v{version}
```
Сообщи: "Тег v{version} поставлен. Не забудь запушить: `git push origin v{version}`"
Do not push automatically — only tag locally.

If a tag with this version already exists, warn user and ask whether to overwrite (`git tag -f v{version}`).

### `changelog`
Генерация текста "What's New" для App Store Connect на основе git-истории.

**Шаг 1 — выбор диапазона:**
Покажи два варианта:
1. С последнего тега — run `git tag --sort=-creatordate | head -5` to show recent tags, then use `git log {last_tag}..HEAD --oneline`
2. С конкретного коммита — run `git log --oneline -20` and show the list, ask user to pick one

If no tags exist and user picks option 1 — сообщи и предложи `/ios tag` после
релиза, затем предложи выбрать конкретный коммит.

**Шаг 2 — генерация текста:**
Collect commit messages from the chosen range. Filter out noise commits:
- bump-build, bump-version, Merge branch, Merge pull request,
  fix typo, whitespace, formatting, wip, tmp, chore, lint,
  однословные коммиты (".", "update", "changes", "fix", "test")

Group remaining commits into user-facing categories:
- 🆕 Новое / New
- 🔧 Улучшения / Improvements
- 🐛 Исправления / Bug Fixes

**Шаг 3 — генерация по локалям:**
Generate one "What's New" block per locale listed in `localizations` config.
- Keep each locale under 4000 characters (App Store Connect limit)
- Пиши в тоне, подходящем для конечных пользователей — без технического жаргона,
  без внутренних ID тикетов
- Translate into the target locale language (en → English, ru → Russian, etc.)

**Формат вывода:**
```
── What's New — v{version} ──────────────────────

🇬🇧 English (en):
[ready text]

🇷🇺 Russian (ru):
[ready text]

─────────────────────────────────────────────────
Скопируй в App Store Connect → My Apps → {App} → {Version} → What's New
```

### `init`
Проверь и установи необходимые инструменты:

**1. Homebrew**
```
which brew
```
Если отсутствует — спроси пользователя, хочет ли он установить Homebrew.
Если да:
```
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```
Если нет — продолжи без Homebrew, но предупреди что некоторые инструменты
придётся устанавливать вручную.

**2. Xcode Command Line Tools**
```
xcode-select -p
```
If missing:
```
xcode-select --install
```

**3. Transporter.app (опционально)**
Check if Transporter is installed:
```
find /Applications -name "Transporter.app" -maxdepth 2
```
Если отсутствует — сообщи: "Установи Transporter из Mac App Store (бесплатно) —
он нужен для загрузки IPA если altool не сработает. Поиск: 'Transporter' от Apple."
Не блокируй — это опционально, продолжи с init.

After tools are set up, run discovery to auto-detect what's possible, then
ask the user only for values that can't be discovered automatically.

**Автоматически (discovery):**
- `scheme` — из `xcodebuild -list`
- `simulator` — из `xcrun simctl list devices available` (newest iPhone)
- `test_timeout` — default `300`

**Спросить у пользователя:**
1. `localizations` — "Какие локали поддерживает приложение? (например: en, ru, de)"
   Подсказка: посмотри `.lproj` папки и `.xcstrings` файлы в проекте, предложи
   найденные локали как значение по умолчанию. Пользователь подтвердит или поправит.
2. `has_subscriptions` — "В приложении есть встроенные подписки (IAP)? (да/нет)"
3. `bundle_id_prefix` — "Префикс product ID для IAP (например com.company.app),
   или пропусти если не используешь:"
4. `archives_dir` — "Куда сохранять архивы? (например ~/Archives, ~/Desktop/Archives):"
   Если пользователь не ответит — использовать default `~/Archives`.

**После сбора:** сгенерируй `.claude/ios-release.yml`, покажи пользователю
для проверки и запиши после подтверждения. Посоветуй добавить в `.gitignore`
если есть персональные пути, или закоммитить если настройки общие.
Never commit without asking the user.

---

## Hard Rules

- Never modify files in `DerivedData/`, `Pods/`, `.build/`, or any `*.generated.swift`
- Never run `git commit`, `git push`, or modify git history without explicit user approval
- In `fix`: group edits by file — open each file once, fix all its errors, save once
- In `fix`: only replace deprecated APIs if the replacement fits the project's deployment target
- `release` never runs `pre-release-check` or `bump-build` twice
- Always use `$HOME` instead of `~` in all paths passed to xcodebuild and shell commands
- All user-facing messages in Russian (команды, ошибки, подсказки)
- Before installing anything (Homebrew, tools) — always ask user for confirmation
