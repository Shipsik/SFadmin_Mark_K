# HelpdeskTool — Контекст для следующего агента

> Сформировано: 2026-08-31 (обновлено)
> Место: `C:\Users\mkarimov\Documents\HelpdeskTool`
> Платформа: Windows, PowerShell 5.1 (win32)
> Git-репозиторий: да

---

## 1. Краткая суть текущего состояния

Это интерактивная консольная утилита (helpdesk), написанная на PowerShell. Добавлены два новых раздела в главное меню: **Сканеры и COM-порты** и **USB и беспроводные устройства** — с полной диагностикой, исправлениями и rollback-интеграцией. Также добавлен раздел **Почта и 1С** («Диагностика и исправление почты (1C)») с 14 функциями: DNS/SPF/DMARC/NS/SOA-анализ, автоопределение провайдера (Google / Yandex / Mail.ru / Microsoft 365 / др.), IMAP/SMTP/TLS-интеллект, трассировка пути, прокси, конфиг 1C.

Текущая версия: **v3.2.2** (exe пересобран). Весь регрессионный набор зелёный (11/11 тестов + 2 exe-теста + 2 новых mail-теста).

---

## 2. Файлы

| Путь | Роль |
|---|---|
| `HelpdeskTool.psm1` | **Главный рабочий файл** (~10000 строк). Вся логика. |
| `HelpdeskTool.ps1` | Лаунчер (обёртка + switch `-Mode` для элевации). |
| `HelpdeskTool.exe` | Собранный exe, **v3.2.1**. |
| `HelpdeskTool_clean.psm1` | **СТАРЫЙ базовый файл** (untracked, НЕТ пейджера, 195 функций). НЕ использовать. |
| `%TEMP%\opencode\build_hk_exe.ps1` | Скрипт сборки exe (PS2EXE-GUI v0.5.0.34). |
| `%TEMP%\opencode\parse_check.ps1` | Проверка синтаксиса. |

### Тестовые скрипты (`%TEMP%\opencode\`)

| Файл | Статус |
|---|---|
| `test_filter_bugs.ps1` | **T1-T4 PASS** |
| `test_scenarios.ps1` | **9/9 PASS** |
| `test_interactive2.ps1` | **14/14 PASS** |
| `test_menu_items.ps1` | **ALL_ITEMS_OK** |
| `test_roundtrip.ps1` | **ROUNDTRIP_PASS=True** |
| `test_rb.ps1` | **CASE1/CASE2 OK, CASE3_HANDLED (OK3=False ожидаемо)** |
| `test_analytics2.ps1` | **EXPORT_OK HAS_STEPS=True** |
| `test_startup.ps1` | **STARTUP_NO_FILTER_PASS=True** |
| `test_mail_fix.ps1` | **gmail/yandex IMAP+SMTP resolve, 1C без краха** |
| `test_mail_tls.ps1` | **ALL_PASS**: TLS-движок реально исполнен, closed-port/empty-host, resolve, 1C, Trace |
| `test_mail_e2e.ps1` | **E2E_OK=True**: Test-MailFull целиком без краха и хенга. На живом Gmail: TLS 1.0-1.3=True, IMAP banner+Auth, SMTP+STARTTLS+AUTH работают |
| `probe_mail_sections.ps1` | **PROBE_FAILS=0**: IMAP/SMTP/Trace/KB+Diag bounded (Jobs+Timeout 40с) |
| `test_filter_exe.ps1` | **EXE_FILTER_PASS=True** |
| `test_software_exe.ps1` | **EXE_SOFTWARE_PASS=True** |

---

## 3. Критически важные технические ограничения

### Кодировка
- PS 5.1, `Set-StrictMode`. Нет `??`, нет тернарника `?:`, нет `&&`/`||` в конвейере.
- Возврат массива из одного элемента: `return ,(...)`.
- **КРИТИЧНО:** тестовые `.ps1` через `write`-tool = UTF-8, но `powershell -File` читает CP866. **Кириллицу через `[char]`-кодовые точки** (`function Cyr([int[]]$codes){...}`). Ставить `[Console]::OutputEncoding=[Text.Encoding]::UTF8` и `$OutputEncoding=New-Object System.Text.UTF8Encoding($false)`.

### Скриптованный ввод клавиш
- Токены: `@down` `@up` `@enter` `@esc` `@bksp` `/`/chars — **каждый на отдельной строке** `$global:HkStdinLines`.
- Обычный текст берёт **только первый символ**.
- `HK_TEST_INTERACTIVE=1` включает интерактивную ветку при перенаправленном вводе/выводе.
- В тест-режиме при пустой очереди `Get-HkScriptKey` возвращает **Enter** (auto-Enter). Не заканчивать тест-последовательность в состоянии no-match фильтра.

### Рендеринг
- `Initialize-HkBuffer` (`:896`) = `return $false` (double-buffer откачен). Мёртвый код буфера инертен.
- `Render-HkRow` (`:1028`): trailing spaces рисуются с цветом последнего сегмента (исправлено в v3.1).
- `Render-HkFrame` (`:960`): diff-кэш сравнивает сегменты целиком (текст + fg + bg), не только текст.
- `Clear-Host` ОДИН раз; фреймы через `SetCursorPosition` + diff-кэш `$_prevFrame` → минимальное мерцание.
- `Clear-Screen` инвалидирует кэш фреймов.

### Точки интереса в коде
- `Select-Item` (~`:1548`): рендер, key-handling, no-match ветка.
- `Show-HkPager` (~`:418`).
- `Render-HkRow` (`:1028`): рендер строки с цветами.
- `Render-HkFrame` (`:960`): diff-кэш фреймов.
- `Show-HelpdeskMenu` (~`:8600`): 16 пунктов меню (вкл. раздел «Оборудование ввода»).
- `Export-ModuleMember` (~`:8790`): список экспортируемых функций.

---

## 4. Новые функции (COM/USB)

### Секция «Сканеры и COM-порты» (`Show-ComPorts`, ~:8526)

**Диагностика:**
| Функция | Что делает |
|---|---|
| `Get-ComDevices` | Все COM-порты, VID/PID, статус |
| `Get-ComPortMap` | Таблица порт → устройство |
| `Get-ComPortConflicts` | Конфликты портов |
| `Get-GhostComDevices` | «Призрачные» (отключённые/неизвестные) устройства |
| `Get-ComPortHistory` | История изменений портов (журнал) |
| `Get-ScannerDiagnostics` | Диагностика сканеров по VID/PID + рекомендации |
| `Test-ComPort` | Тест доступности COM-порта |
| `Save-ComPortSnapshot` | Снимок всех портов в JSON |

**Исправления:**
| Функция | Что делает | Rollback |
|---|---|---|
| `Set-ComPortNumber` | Переназначение COM-порта | `comport` |
| `Set-ComPortParams` | Настройка baud/parity/data/stop | `comparams` |
| `Restart-ComDevice` | PnP-перезапуск сканера | нет (транзитная операция) |
| `Remove-GhostComDevice` | Удаление «призраков» (с `pnputil` fallback) | нет |
| `Reset-ScannerDefaults` | Сброс сканера на заводские через SerialPort | нет (команда сброса) |

### Секция «USB и беспроводные устройства» (`Show-UsbDevices`, ~:8563)

**Диагностика:**
| Функция | Что делает |
|---|---|
| `Get-UsbDevicesExtended` | Все USB-устройства с расширенной инфо |
| `Get-UsbPowerStatus` | Статус USB power management |
| `Get-WirelessDevices` | Беспроводные клавиатуры/мыши |
| `Get-WirelessBatteryLevel` | Заряд батареи беспроводных (WMI BatteryStatus) |
| `Get-UsbHubLoad` | Загрузка USB-хабов |
| `Get-UsbEventLog` | USB-ошибки в журнале (с подсказкой про права) |

**Исправления:**
| Функция | Что делает | Rollback |
|---|---|---|
| `Disable-UsbSelectiveSuspend` | Отключение USB Selective Suspend | `usbpower` (SelectiveSuspendEnabled) |
| `Disable-UsbPowerMgmt` | Отключение WaitWakeEnabled | `usbpower` (WaitWakeEnabled) |
| `Reset-UsbDevice` | Disable+Enable цикл | нет (транзитная) |
| `Reset-UsbHubPower` | Перезапуск USB-хаба | нет (транзитная) |
| `Enable-DisabledUsbDevice` | Включение отключённого устройства | нет |

### VID/PID базы данных
- `$script:ScannerVIDs` (~:130) — VID сканеров (Honeywell, Zebra, Datalogic, CipherLab, Newland, Mindeo, FTDI/FTDI)
- `$script:WirelessHIDs` (~:146) — VID беспроводных клавиатур/мышей (Logitech, Microsoft, Razer, Corsair)
- `$script:KnownBadVIDs` (~:160) — «плохие» VID (глючные контроллеры)

### Диспетчер эLEVации
- `HelpdeskTool.ps1:68-78` — switch `$Mode` для 9 COM/USB-режимов:
  `SetComPortNumber`, `SetComPortParams`, `RestartComDevice`, `RemoveGhostCom`, `DisableUsbSS`, `DisableUsbPM`, `ResetUsbDev`, `ResetUsbHub`, `EnableUsbDev`
- **ВАЖНО:** новые функции **обязаны** быть в `Export-ModuleMember` иначе elevated process их не увидит.

---

## 5. Исправленные баги (в этой сессии)

### 5.1 `Remove-PnpDevice` не существует на Win10 < 1809
**Проблема:** `Remove-PnpDevice` появился только в Win10 1809. На старых системах — `CommandNotFoundException`.
**Решение:** `Remove-GhostComDevice` проверяет `Get-Command Remove-PnpDevice` и fallback на `pnputil /remove-device`.

### 5.2 `Disable-UsbPowerMgmt` — нет rollback
**Проблема:** Пишет в `HKLM:\...\WDF\WaitWakeEnabled` без отката.
**Решение:** `Add-RollbackRecord -Type 'usbpower'`. Ветка `'usbpower'` в `Undo-RollbackRecord` расширена на `WaitWakeEnabled`.

### 5.3 `Get-ScannerDiagnostics` — небезопасный доступ к хешу
**Проблема:** `$script:ScannerVIDs[$vid].Maker` без `.ContainsKey()`.
**Решение:** Обёрнуто в `ContainsKey()`.

### 5.4 Pager: подсветка при скролинге слетает
**Проблема:** `Render-HkRow` trailing spaces рисовались без цвета (дефолтные цвета консоли). Diff-кэш в `Render-HkFrame` сравнивал только текст, пропуская строки с изменившимся цветом.
**Решение:** Trailing spaces рисуются с цветом последнего сегмента. Diff-кэш сравнивает сегменты целиком (текст + fg + bg).

### 5.5 `Get-ScannerDiagnostics` — $scanners не заполнялся
**Проблема:** Массив `$scanners` объявлен но никогда не пополнялся → всегда показывал «сканеры не найдены».
**Решение:** Добавлен `$scanners += $p` после отображения информации о сканере.

### 5.6 Меню-позиции в тестах
Добавлены 2 новых пункта в меню (после «Откат изменений»). Тесты обновлены: 14 → 16 для «Выход» и «О программе».

### 5.7 Почта: «AuthenticateAsClient на Boolean» (SMTP/TLS/IMAP)
**Проблема:** `$ssl` потоковая переменная конфликтовала с параметром `[bool]$Ssl` (PowerShell case-insensitive — это одна переменная), а `param([string]$Host)` конфликтовал с автоматической переменной `$Host`. `$ssl.AuthenticateAsClient()` вызывался на Boolean.
**Решение:** Параметр переименован в `$Srv` во всех трёх движках (`Test-MailTlsIntelligence`, `Test-MailImapIntelligence`, `Test-MailSmtpIntelligence`), потоковая переменная — в `$sslStream`. SslStream создаётся через явный `[System.Net.Security.RemoteCertificateValidationCallback]{$true}`. Все 6 вызовов обновлены на `-Srv`.

### 5.8 Почта: IMAP-хост «:0» для Gmail
**Проблема:** SRV-запись `_imap._tcp` для Gmail возвращает `Port=0` → `$result.ImapPort = 0`; проверки `-eq $null` не ловили пустую строку; не было провайдер-basefallback.
**Решение:** SRV-порт применяется только если `>0`; проверки заменены на `[string]::IsNullOrWhiteSpace()`; добавлен провайдер-дефолт в конце (Google → `imap.gmail.com:993`/`smtp.gmail.com:587`+STARTTLS; Yandex → `imap.yandex.ru:993`/`smtp.yandex.ru:465`; Mail.ru → `imap.mail.ru`/`smtp.mail.ru`; Microsoft → `outlook.office365.com`/`smtp.office365.com:587`+STARTTLS).

### 5.9 Почта: `Get-MailConfig1C` — DisplayName не найден
**Проблема:** `Get-ItemProperty '...Uninstall\*'` при нестандартной структуре ключей возвращал объект без свойства `DisplayName`; `Where-Object { $_.DisplayName -match ... }` падал.
**Решение:** Фильтр обёрнут в `$_.PSObject.Properties.Name -contains 'DisplayName'`; чтение `DisplayVersion`/`InstallLocation` также с guard'ами.

### 5.10 Почта: SMTP 587 (Gmail) — STARTTLS вместо неявного SSL
**Проблема:** Для порта 587 всегда делался неявный SSL (`if ($Ssl)`), а Gmail на 587 требует STARTTLS.
**Решение:** Добавлен флаг `$result.SmtpStartTls`; в `Test-MailSmtpIntelligence` при `$StartTls -and $result.SupportsStartTls` выполняется команда `STARTTLS` и апгрейд потока через SslStream; Google/Microsoft дефолты устанавливают `SmtpStartTls=$true, SmtpSsl=$false`.

### 5.11 Почта: крах TLS-движка «Не удается индексировать в объект SslProtocols»
**Проблема:** `$protoList` собирался через `@()` с переносами строк — PowerShell ФЛАТИТ вложенные массивы без запятой (`,@(...)`). На 1-й итерации `$proto[0]` выполнялся на enum `SslProtocols` → исключение вне `try` → смерть всего меню.
**Решение:** каждая строка обрамлена `,@(...)`, добавлен guard `if (-not ($proto -is [System.Array]) -or $proto.Count -lt 2) { continue }`, извлечение `$proto[0]/[1]` внутри `try`, метка `'TLS 13'`→`'TLS 1.3'`.

### 5.12 Почта: `Test-MailFull` зависает навсегда
**Проблема:** Все чтения в движках (`$reader.ReadLine()`: banner/CAPABILITY/NAMESPACE/EHLO/STARTTLS) — блокирующие, без таймаутов; при замолчавшем/throttled сервере (Gmail режет IP после серии подключений) `ReadLine` висел бесконечно. `Test-NetConnection -TraceRoute` в `Trace-MailPath` на рабочей сети — до минут.
**Решение:** добавлены `Connect-HkTcp` (BeginConnect+WaitOne), `Invoke-HkSslAuth` (BeginAuthenticateAsClient+WaitOne(8с)), `Read-HkMailLine` (логит ReadTimeout и возвращает `$null`); `$stream.ReadTimeout=10000` (+SslStream) во всех 3 движках; EHLO цикл = кап 60 итериций; `Trace-MailPath` переписан на `tracert.exe -d -h 15 -w 500` (без reverse-DNS) с `WaitForExit(20000)+Kill`. Итог: worst-case ~20-25с на секцию вместо вечного хенга.

### 5.13 Почта: StrictMode-крах `Get-MailKnowledgeBase`
**Проблема:** `if ($null -ne $script:MailKB)` — первое обращение к неназначенной script-переменной → «Переменная $script:MailKB не может быть получена». Падал раздел «Автодиагностика».
**Решение:** `Get-Variable -Name MailKB -Scope Script -ErrorAction SilentlyContinue`; кэш живёт в `$script:MailKB`.

### 5.14 Почта: `$input` (автовар) вместо email
**Проблема:** `$input = Read-Host` перекрывал автоматическую переменную; в `Add-Audit` и логировании читался `$input` (перечислитель) — мусор.
**Решение:** переименовано в `$mailInput` (Test-MailFull + DNS-меню), `Add-Audit` пишет `$mailInput`.

### 5.15 Почта: `[int]$portIn` бросает при некорректном вводе
**Проблема:** `[int]$portIn` без try в `Show-MailDiagnostics` (TLS/IMAP/SMTP) → исключение при не-цифре.
**Решение:** `$port = if ($portIn) { try { [int]$portIn } catch { 993|465 } } else { 993|465 }`.

### 5.16 Почта: отображение `AUTH: System.Object[]`
**Проблема:** `-f` имеет больший приоритет, чем `-join`: `('...' -f a, b -join ', ')` форматировал массив как `System.Object[]`, а join выполнялся над результатом `-f`.
**Решение:** скобка `($smtp.AuthMethods -join ', ')` в `Test-MailFull`.

### 5.17 Почта: «Нет доступного пространства выполнения» — крах TLS/IMAP/SMTP живого сервера
**Проблема:** `Invoke-HkSslAuth` использовал async `BeginAuthenticateAsClient` (Begin+WaitOne+End). Callback/завершение async выполнялись на threadpool-потоке без PowerShell-runspace → `EndAuthenticateAsClient` бросал «Нет доступного пространства выполнения для запуска сценариев в этом потоке». Из-за этого на живом Gmail TLS всегда False, IMAP/SMTP падали с обрезанной ошибкой — это выглядело как «Gmail режет IP», но на деле был баг кода.
**Решение:** `Invoke-HkSslAuth` переписан на sync `AuthenticateAsClient` внутри отдельного PS-runspace (`[System.Management.Automation.PowerShell]::Create()` + `BeginInvoke`/`WaitOne(8с)`/`Stop()`), ошибки берутся из `$ps.Streams.Error`. Теперь на живом сервере: `TLS 1.0-1.3=True`, `Sys=True`, IMAP banner + Auth, SMTP+STARTTLS работают реально.

### 5.18 Почта: `SslProtocols::Tls13` падал / не было системного протокола
**Проблема:** на .NET Framework < 4.8 `[SslProtocols]::Tls13` (12288) не существует → на этих системах либо крах при конструировании `$protoList`, либо TLS 1.3 навсегда False.
**Решение:** добавлен Try для `Tls13`, добавлен 5-й протокол `[SslProtocols]0` (`TLS (системный)` → `$result.TlsSystem` = Windows сам negotiated лучший из доступных). Вывод показывается как `TLS 1.0..1.3 системный=`.

### 5.19 Почта: `Cert.PublicKey.Key.KeySize` — крах на ECDSA (Gmail)
**Проблема:** у ECDSA-сертификатов (Gmail/Google Trust Services) `PublicKey.Key` не имеет `KeySize` → StrictMode-исключение прерывало разбор сертификата.
**Решение:** `try { $result.KeySize = $cert.PublicKey.Key.KeySize } catch { }`.

### 5.20 Почта: ложный «Имя хоста не совпадает с сертификатом»
**Проблема:** `VerifyRemoteCertificateName` зависел от построенной цепочки (которую мы байпасим колбэком `{$true}`) и ошибочно давал MISMATCH на валидных сертификатах (Gmail).
**Решение:** при исключении Verify — fallback-проверка имени по SAN (`DNS:`-тью) и `CN=хост` в Subject; MISMATCH только если имени реально нет.

### 5.21 Почта: SMTP AUTH пустой на STARTTLS (587)
**Проблема:** Gmail в первом EHLO (до STARTTLS) не отдаёт строку AUTH — методы появляются только после апгрейда на TLS во втором EHLO. Код читал AUTH только из первого EHLO → `AUTH: (не определены)`.
**Решение:** после STARTTLS-upgrade отправляется повторный `EHLO`, парсятся `AUTH`/`SIZE`/`8BITMIME`/`CHUNKING`/`PIPELINING`. Теперь: `AUTH: LOGIN, PLAIN, XOAUTH2, ...`.

### 5.22 Почта: полные тексты ошибок + пустой AUTH в выводе
**Проблема:** Warnings показывали обрезанную обёртку PowerShell (`Нет доступного прос…`).
**Решение:** в catch IMAP/SMTP и TLS берётся `InnerException.Message`, при пустом — `Message`. Пустой `AuthMethods` выводится как `(не определены)`.

### 5.23 Почта: неправильный резолв IMAP/SMTP (MX ≠ IMAP ≠ SMTP)
**Проблема:** Для корпоративных серверов (lovekuhnya.ru) MX-хост `mx0.lovekuhnya.ru` ≠ реальный IMAP/SMTP сервер. Наш инструмент использовал MX-хост как IMAP/SMTP → connection refused. `Test-HkPort` проверял только TCP — TCP проходил, но TLS/сервис не отвечал.
**Решение:** Добавлен `Test-HkMailPort` (проверка баннера: IMAP `* OK` / SMTP `220`). Порт-пробинг: 993→143, 465→587. Если все порты одного хоста не отвечают — fallback на альтернативные хосты (`mail.domain`, `domain`) с теми же портами. Если ни один хост/порт не работает — предупреждение «Введите вручную». Для `lovekuhnya.ru`: `mail.lovekuhnya.ru` → CNAME → Yandex, порты Yandex работают → автоматическое переключение.

### 5.23a Почта: CNAME-детекция провайдера (NetAngels → Yandex)
**Проблема:** Провайдер определялся только по MX-хостам (`mx0.lovekuhnya.ru` → нет "yandex" → Corporate). RFC-резолв ставил `imap.lovekuhnya.ru:993` до детекции провайдера → switch-блок с Yandex-хостами не срабатывал.
**Решение:** Добавлена CNAME-детекция `mail.$domain` после MX/SPF-проверок. При обнаружении CNAME на `yandex.net`/`google.com`/etc → сброс `ImapHost`/`SmtpHost` в `$null` → switch подставляет стандартные хосты провайдера (imap.yandex.ru:993, smtp.yandex.ru:465).

### 5.23b Почта: NS-детекция хостинг-провайдера + комбинированный вывод + собственный почтовый сервер
**Проблема:** Провайдер показывался как "Yandex" для NetAngels-доменов — пользователь хочет видеть хостинг-провайдера (NetAngels) и почтового (Yandex). При этом NetAngels имеют СВОЙ почтовый сервер `mail.netangels.ru` (Thunderbird подключается к нему), а не использует Yandex для IMAP/SMTP.
**Решение:** Разделение `$result.Provider` (хостинг, по NS) и `$result.MailProvider` (почта, по CNAME). Вывод комбинируется: "NetAngels (Yandex)". NS-детекция идёт через каталог `$script:MailProviderCatalog` (данные, не хардкод): NS-домен → почтовый сервер хостинга. Автообнаружение как у Thunderbird: упорядоченный список кандидатов (почтовый сервер хостинга → host домена клиента → облачный провайдер), каждый пробуется 993→143 / 465→587, первый рабочий побеждает. Для lovekuhnya.ru выбирается `mail.netangels.ru:143` и `mail.netangels.ru:465`.

### 5.24 Почта: IMAP не поддерживал STARTTLS (порт 143)
**Проблема:** IMAP-движок работал только в implicit SSL (993) или plain (143 без шифрования). Fallback на порт 143 был бесполезен без STARTTLS.
**Решение:** Добавлен STARTTLS-апгрейд в `Test-MailImapIntelligence`: после баннера на порту 143 отправляется `STARTTLS`, при `+OK` — апгрейд через `Invoke-HkSslAuth` в PS-runspace.

### 5.25 TLS-движок: fast-fail при connection refused
**Проблема:** Когда сервер отклоняет TCP (connection refused), все 5 TLS-проб тратят ~40с на одно и то же.
**Решение:** Добавлен флаг `$fatalError` — при ошибке `отверг запрос|connection refused|таймаут подключения|timeout` остальные протоколы пропускаются.

---

## 6. Главное меню (16 пунктов)

```
1.  Сводка о системе
2.  Оборудование и диски
3.  Сеть и интернет
4.  Программы и обновления
5.  Процессы и службы
6.  Диагностика и анализ
7.  Быстрые исправления
8.  Откат изменений
9.  Сканеры и COM-порты          ← НОВОЕ
10. USB и беспроводные устройства ← НОВОЕ
11. Почта и 1С («Диагностика и исправление почты (1C)»)  ← НОВОЕ
12. {{AI}} (пункт отображается как-то иначе)
13. К3-Мебель
14. Настройки Windows
15. Полный отчёт в файл
16. О программе
17. Выход
```

> Пункты-разделители (`──...──`) не навигируются, отсчёт selectable-пунктов идёт по их реальному индексу в массиве `$menuBase`.
> Меню сейчас 17 selectable-пунктов (exit=17 в тестах).

---

## 7. Как тестировать

```
& powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Users\mkarimov\AppData\Local\Temp\opencode\parse_check.ps1"
```
Затем поочерёдно регрессию (см. таблицу). Сборка exe:
```
& powershell -NoProfile -ExecutionPolicy Bypass -File "C:\Users\mkarimov\AppData\Local\Temp\opencode\build_hk_exe.ps1"
```
После — прогнать `test_filter_exe.ps1` и `test_software_exe.ps1`.

---

## 8. Правила работы

- Тестировать каждый шаг.
- Не откатывать к Hk_clean — удалит пейджер и COM/USB-фичу.
- Не коммитить без явного запроса.
- Кириллица в `.ps1`-тестах — через `[char]`-кодовые точки.
- Тестировать через скриптовые токены, а не наугад.

---

## 9. Итоговая картина покрытия

| Проверка | Результат |
|---|---|
| parse_check | NO_PARSE_ERRORS |
| test_filter_bugs (T1-T4) | PASS |
| test_scenarios | 9/9 PASS |
| test_interactive2 | 14/14 PASS |
| test_menu_items | ALL_ITEMS_OK |
| test_roundtrip | ROUNDTRIP_PASS=True |
| test_rb | CASE1/CASE2 OK, CASE3_HANDLED (OK3=False ожидаемо) |
| test_analytics2 | EXPORT_OK HAS_STEPS=True |
| test_startup | STARTUP_NO_FILTER_PASS=True |
| test_mail_fix | gmail/yandex IMAP+SMTP resolve, 1C без краха, Boolean-ошибки нет |
| test_mail_tls | **ALL_PASS**: TLS-цикл реально исполняется (127.0.0.1:1 + imap.gmail.com:993), IMAP/SMTP closed-port, resolve, 1C, Trace no-crash |
| test_mail_e2e | **E2E_OK=True**: Test-MailFull целиком без краха и хенга. На живом Gmail: TLS 1.0-1.3=True, IMAP banner+Auth, SMTP+STARTTLS+AUTH работают |
| probe_mail_sections | **PROBE_FAILS=0**: IMAP/SMTP/Trace/KB+Diag все bounded (Trace ≤20с, IMAP ≤18с) |
| test_filter_exe | EXE_FILTER_PASS=True |
| test_software_exe | EXE_SOFTWARE_PASS=True |
