# HermesOffice

Форк [GogikOrtey/HermesOffice](https://github.com/GogikOrtey/HermesOffice) (`origin`, публичный, ветка `main` отслеживает `origin/main`). Исходный репозиторий — [criptogus/HermesOffice](https://github.com/criptogus/HermesOffice) (`upstream`). Локальная копия на коммите `daa0e17`. Монорепозиторий npm-workspaces: шесть Electron-приложений в `apps/` (docs, sheets, slides, pdf, markdown, shell) и движки в `packages/`. Excel-импорт — Rust sidecar `apps/sheets/native/xlsx-engine`.

## Сборка Windows

- Node 22.15, npm 10.9, Rust `stable-x86_64-pc-windows-msvc` (VS 2019 C++ уже стоит). `core.longpaths=true` в локальном git.
- Перед `npm run notices` на свежем клоне нужен `node tools/write-build-info.mjs`: `apps/shell/electron-builder.cjs` требует `apps/shell/build/build-info.json` уже при загрузке конфига.
- `npm run build:all` кладёт `xlsx-sidecar.exe` в `apps/sheets/native/xlsx-engine/target/release/`. Упаковщик Windows читает путь кросс-компиляции `target/x86_64-pc-windows-gnu/release/xlsx-sidecar.exe` — exe туда копируется после `build:all`, исходники не патчатся.
- NSIS (неподписанный, x64): `electron-builder --win` из `apps/shell`. Каталог вывода вынесен из OneDrive в `%LOCALAPPDATA%\hermesoffice-dist`, потому что Защитник и OneDrive блокируют запись свежего `HermesOffice.exe` (`EBUSY` / `EPERM`).
- Установщик: `%LOCALAPPDATA%\hermesoffice-dist\HermesOffice Setup 0.8.3.exe` (~129 МБ, `NotSigned`). Рядом `win-unpacked\HermesOffice.exe` — запуск без установки. SmartScreen на неподписанный файл предупреждает.
- Автообновление и аналитика в этой сборке выключены: нет `HERMESOFFICE_UPDATE_URL` и GA4.
