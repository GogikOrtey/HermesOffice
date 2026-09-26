# HermesOffice

Форк [GogikOrtey/HermesOffice](https://github.com/GogikOrtey/HermesOffice) (`origin`, публичный, ветка `main` отслеживает `origin/main`). Исходный репозиторий — [criptogus/HermesOffice](https://github.com/criptogus/HermesOffice) (`upstream`). Локальная копия на коммите `daa0e17`. Монорепозиторий npm-workspaces: шесть Electron-приложений в `apps/` (docs, sheets, slides, pdf, markdown, shell) и движки в `packages/`. Excel-импорт — Rust sidecar `apps/sheets/native/xlsx-engine`.

Подробностей самого проекта здесь нет: код не разбирали и архитектуру приложений не описывали. Зафиксировано только то, что понадобилось, чтобы сделать форк и собрать программу под Windows.

## Сборка Windows

- Node 22.15, npm 10.9, Rust `stable-x86_64-pc-windows-msvc` (VS 2019 C++ уже стоит). `core.longpaths=true` в локальном git.
- Перед `npm run notices` на свежем клоне нужен `node tools/write-build-info.mjs`: `apps/shell/electron-builder.cjs` требует `apps/shell/build/build-info.json` уже при загрузке конфига.
- `npm run build:all` кладёт `xlsx-sidecar.exe` в `apps/sheets/native/xlsx-engine/target/release/`. Упаковщик Windows читает путь кросс-компиляции `target/x86_64-pc-windows-gnu/release/xlsx-sidecar.exe` — exe туда копируется после `build:all`, исходники не патчатся.
- NSIS (неподписанный, x64): `electron-builder --win` из `apps/shell`. Каталог вывода вынесен из OneDrive в `%LOCALAPPDATA%\hermesoffice-dist`, потому что Защитник и OneDrive блокируют запись свежего `HermesOffice.exe` (`EBUSY` / `EPERM`).
- Установщик: `%LOCALAPPDATA%\hermesoffice-dist\HermesOffice Setup 0.8.3.exe` (~129 МБ, `NotSigned`). Рядом `win-unpacked\HermesOffice.exe` — запуск без установки. SmartScreen на неподписанный файл предупреждает.
- Автообновление и аналитика в этой сборке выключены: нет `HERMESOFFICE_UPDATE_URL` и GA4.

## Агент: состав запроса и кеш префикса

Кеша ответов и prompt caching нет. `AgentLoop` (`packages/agent-core`) каждый ход заново шлёт системный промпт, схемы инструментов и историю. Единственный короткий кеш — успешный `GET /health` Hermes-шлюза на 30 с (`packages/ai-provider/src/hermes-health.ts`).

Провайдеры собирают тело в `packages/ai-provider/src/protocols/`: Claude — `tools`, затем `system`, затем `messages`; OpenAI-совместимые (в том числе Hermes `127.0.0.1:8642/v1`) — первое сообщение `system`, инструменты отдельным полем `tools`; Gemini — `systemInstruction` и `tools` отдельно от `contents`. Снимок документа клеится к сообщению пользователя (`buildContext`), полный текст приходит ответами инструментов и дальше едет в истории.

На первом ходе запроса основную долю занимают промпт и схемы (документы ~15+15 тыс. символов, таблицы ~10+17, слайды ~17+50, PDF ~5+20). Снимок документа короткий: у документов список блоков до 8 тыс. символов (превью 60), у таблиц только лист и выделение, у слайдов оглавление с превью 50 символов. После нескольких `read_blocks` / `read_pages` (до 24 тыс. символов) или `read_range` (до 2000 ячеек) история с текстом документа обгоняет префикс.

Нативный кеш префикса (идея — прослойка LiteLLM, не склад готовых ответов) покрывает системный промпт и схемы: они стабильны на ходах одного задания и стоят перед снимком документа. Встроенный кеш LiteLLM на весь запрос почти не попадает. Claude кеширует только с `cache_control` (сейчас `system` — обычная строка; LiteLLM может вставить точку через `cache_control_injection_points`). OpenAI и Gemini кешируют длинный одинаковый префикс сами, если прокси не переставляет поля. DeepSeek (тот же OpenAI-совместимый протокол, без `cache_control`) кеширует префикс на своей стороне автоматически: проверено по статистике использования, отдельные метки в запросе не нужны. Порог около 1024 токенов; связка промпта и инструментов его перекрывает. Префикс разъезжается, если меняется директива языка (`systemSuffix`), отфильтрован облачный инструмент (`generate_image`) или ход после лимита шагов уходит с `tools: []`. Сжатие истории в резюме инвалидирует кеш переписки, не голову из промпта и схем.
