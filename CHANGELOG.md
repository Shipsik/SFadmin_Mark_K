# Changelog

Все заметные изменения проекта документируются в этом файле.
Формат основан на [Keep a Changelog](https://keepachangelog.com/ru/1.1.0/) и [Semantic Versioning](https://semver.org/lang/ru/).

## [Unreleased]

### Добавлено
- Runtime-перезагрузка настроек агента: `UiAgentApprovalService` и `AiToolExecutor` подписаны на `SettingsChanged` — правки «Всегда разрешать/запрещать» и лимитов применяются без перезапуска.
- `AiToolExecutor`: таймаут инструмента (`AgentMaxToolRuntimeSeconds`), семафор параллельности (`AgentMaxConcurrentTools`), контроль лога вызовов (`AgentLogToolCalls`), колбэк `onToolComplete`.

### Удалено (мёртвый код)
- `FixIssuesCommand`, `ExplainErrorsCommand`, `GeneratePsCommand`, `CheckSystemCommand`, `FullDiagnosticsCommand` (дублировали QuickActions)
- `RenameSessionCommand` + метод `RenameSession`
- `ApprovalPending`, `ApprovalToolName`, `AgentMode`, `AskAsync`

## [0.3.0] — 2026-08-01

### Добавлено
- Полный редизайн AI-функционала:
  - Единый источник безопасных тулов: `Core/Policies/AgentToolPolicies.cs` (31 инструмент)
  - 5 тулов базы знаний: `search_kb`, `read_kb_article`, `update_kb_article`, `delete_kb_article`, `list_kb_categories`
  - Авто-запись результатов диагностики в KB (`RunFullDiagnosticsTool`)
  - Окно подтверждения: риск-уровни (🟢/🟡/🔴), горячие клавиши (Y/N/S/A), 3 режима разрешения
  - Групповое подтверждение батчей тулов с легендой рисков
  - 7 настраиваемых ограничений агента в Settings
  - Retry с экспоненциальным бэкоффом для HTTP 429

### Исправлено
- `CloudLlmService.StartAsync` читал ключ без суффикса провайдера → «LLM: unavailable» при валидном ключе
- Кнопки «Спросить ИИ» на вкладках Сеть/БЗ/Плагины/Хранилище (отсутствовал `NetworkViewModel.AskAiCommand`)
- `NetworkViewModel.ServiceStatusCommand` — синтаксис лямбды

## [0.2.0] — 2026-07-20

### Добавлено
- Пакет оси таймлайна: шкала 24px, жёлтые HH:mm:ss у меток, ромб графиков 3.5px
- Нативные PerformanceCounter для CPU/диска (WMI LoadPercentage врёт на FX-8320)

### Исправлено
- «CPU 100% всегда» на Сводке — `EnsureCounters()` съедал честный 10-секундный сэмпл
- Гонка `_ping` (loss 100%→0%) — разделены `_pingGateway`/`_pingInternet`

## [0.1.0] — 2026-07-12

### Добавлено
- Первый рабочий прототип: диагностика сети/домена, база знаний, AI-чат
