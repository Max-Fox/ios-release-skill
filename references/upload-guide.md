# Upload IPA to App Store Connect

## Поиск IPA

Find the most recently modified `.ipa` in `archives_dir` by modification time
(not filename sort). Show the found IPA name and version, ask user to confirm
before uploading. If no `.ipa` found — stop and suggest running `archive` first.

## Методы загрузки

Try upload in this order, stopping at first success:

### Шаг 1 — altool с API-ключом

Работает на всех версиях Xcode, может показать deprecation warning на 16+:
```
xcrun altool --upload-app \
  --type ios \
  --file {path/to/app.ipa} \
  --apiKey {KEY_ID} \
  --apiIssuer {ISSUER_ID}
```
If API key not found at `$HOME/.appstoreconnect/private_keys/AuthKey_{KEY_ID}.p8`, ask for:
- Key ID (10 chars), Issuer ID (UUID), path to `.p8` file
- Где найти: App Store Connect → Users & Access → Integrations → API Keys

If the provided `.p8` path is not already in `$HOME/.appstoreconnect/private_keys/`,
offer to move it there:
```
mkdir -p $HOME/.appstoreconnect/private_keys
mv {provided_path} $HOME/.appstoreconnect/private_keys/AuthKey_{KEY_ID}.p8
```
Сообщи: "Перенесено в стандартное место — в следующий раз altool найдёт ключ автоматически."
Move only after user confirms.

### Шаг 2 — Transporter CLI

Если altool не сработал или нет API-ключа:
```
xcrun Transporter \
  -u {apple_id_email} \
  -p {app_specific_password} \
  -f {path/to/app.ipa}
```
Примечание: требуется Transporter.app из Mac App Store (бесплатно).
`{app_specific_password}` — это НЕ пароль Apple ID. Его нужно сгенерировать на
appleid.apple.com → Sign-In and Security → App-Specific Passwords.

### Шаг 3 — ручной fallback

Если оба способа не сработали:
Сообщи: "Открой Transporter.app и перетащи IPA из `{path/to/app.ipa}`."
