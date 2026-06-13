# Лог изменений

## 2026-06-12 — Проверка и удаление ПО, установленного вне Chocolatey

Добавлена возможность находить программы из списка `windows-software.json`,
установленные на Windows в обход Chocolatey (вручную, через официальные
установщики и т.п.), выгружать их в текстовый отчет и удалять через
штатные деинсталляторы ("Удаление программ").

### Новые файлы

#### `uninstall-software.ps1`

Отдельный PowerShell скрипт с меню для проверки и удаления ПО вне Chocolatey.

- **Пункт меню 1 — Scan and create report**: сканирует ветки реестра
  "Удаления программ":
  - `HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall`
  - `HKLM\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall`
  - `HKCU\Software\Microsoft\Windows\CurrentVersion\Uninstall`

  Программа считается установленной вне Chocolatey, если ее `DisplayName`
  найден в реестре, а пакет (`choco_id`) отсутствует в `choco list`.
  Результат выводится на экран и сохраняется в отчет
  `non-choco-software-report.txt` рядом со скриптом. Для каждой программы
  в отчете: имя, версия, путь установки (`InstallLocation`), строка
  деинсталляции (`UninstallString`), ключ реестра.

- **Пункт меню 2 — Uninstall found programs**: по каждой найденной
  программе запрашивает подтверждение (y/n) и запускает штатный
  деинсталлятор:
  - MSI-пакеты (`UninstallString` содержит `msiexec`) — удаление по
    коду продукта: `msiexec /x {ProductCode}`;
  - остальные — запуск `uninstall.exe` с аргументами из реестра
    (интерактивно, как из "Удаления программ").

  После удаления ключ реестра перепроверяется, и отчет обновляется.

Основные функции:

- `Get-RegistryInstalledPrograms` — чтение всех программ из трех веток реестра;
- `Get-ChocoInstalledIds` — список пакетов, установленных через Chocolatey
  (поддерживаются choco 1.x с `--local-only` и choco 2.x);
- `Get-NonChocoInstalled` — сопоставление списка из JSON с реестром и choco;
- `Write-Report` — запись отчета в `.txt` (UTF-8);
- `Start-ProgramUninstall` — разбор `UninstallString` и запуск деинсталлятора.

Сопоставление имен выполняется по началу `DisplayName` с границей слова,
поэтому "Git" не совпадает с "GitHub Desktop".

Вывод скрипта — только ASCII (английский), согласно правилу проекта
о символах в терминале.

#### `uninstall-software.bat`

Обертка для запуска двойным кликом без настройки ExecutionPolicy:

```bat
@echo off
powershell -NoProfile -ExecutionPolicy Bypass -File "%~dp0uninstall-software.ps1"
pause
```

Сохранен в кодировке ASCII (требование cmd.exe).

#### `non-choco-software-report.txt`

Генерируемый отчет (создается при запуске проверки). Пример записи:

```
[1] Git
    Display name : Git
    Version      : 2.54.0
    Install path : C:\Program Files\Git\
    Uninstall    : "C:\Program Files\Git\unins000.exe"
    Registry key : HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Git_is1
```

### Изменения в существующих файлах

#### `install-software.ps1`

- Добавлены функции `Get-ChocoInstalledIds`, `Get-RegistryInstalledPrograms`,
  `Find-RegistryProgram`, `Get-NonChocoInstalled`, `Write-NonChocoReport`,
  `Show-NonChocoSoftware` (та же логика обнаружения, что и в
  `uninstall-software.ps1`).
- Новый пункт меню **«4. Проверить ПО, установленное вне Chocolatey
  (отчет .txt)»** — выполняет проверку, создает
  `non-choco-software-report.txt` и подсказывает запустить
  `uninstall-software.bat` для удаления.
- В `Install-Software` перед установкой добавлена проверка реестра:
  если программа уже установлена вне Chocolatey, установка пропускается
  с предупреждением (чтобы не получить две копии программы).
- В списке программ (пункт меню 3) символы «✓»/«✗» заменены на
  ASCII-варианты `[+]`/`[-]` согласно правилу проекта.
- Сообщение о неверном выборе меню обновлено: диапазон пунктов 0–4.

#### `README.md`

- В структуру проекта добавлены `uninstall-software.bat`,
  `uninstall-software.ps1` и описание генерируемого отчета.
- В раздел «Быстрый старт» добавлены пункт меню 4 и инструкция
  по запуску `uninstall-software.bat`.
- Добавлены разделы «4. Проверка ПО, установленного вне Chocolatey»
  и «5. Удаление ПО вне Chocolatey» с описанием логики проверки,
  формата отчета и способа удаления.

### Технические замечания

- Кодировки файлов соответствуют конвенции проекта: `.ps1` — UTF-16 LE
  с BOM (для корректной кириллицы в PowerShell 5.1), `.bat` — ASCII,
  отчет `.txt` — UTF-8.
- Синтаксис обоих `.ps1` файлов проверен парсером PowerShell (0 ошибок).
- Проверка протестирована на реальной машине: найдено 5 программ,
  установленных вне Chocolatey (Git, Cursor, Node.js, Google Chrome,
  Obsidian), отчет сформирован корректно, включая MSI-пакет (Node.js)
  и exe-деинсталляторы.
