# Hammerfight 1.004 (Steam) на Windows 10: управление, разрешение 1920×1080 и русский текст

Практическое руководство, проверенное на конкретной конфигурации. Все изменения обратимы.

> Статус: это пользовательское руководство, не официальная документация. Для Википедии такой текст обычно не подходит как оригинальное исследование. Лучшее место — GitHub Gist, личная вики, Викиучебник или профильный форум.

## Проблема

Steam-версия Hammerfight (AppID 41100, build 1.004) на современных системах:

- не реагирует на щелчки мыши, пока не отключён режим поддержки нескольких мышей;
- при включённом Raw Input теряет или искажает движение мыши;
- по умолчанию работает в 800×600 и растягивает картинку на монитор 1920×1080;
- не содержит файл русской локализации, хотя русская версия существовала в других изданиях;
- после увеличения разрешения текст становится резче, но исходные спрайты 512×512 остаются мягкими.

## Конфигурация, на которой проверено

- Windows 10 Pro, сборка 19045.
- Steam, Hammerfight AppID 41100, версия 1.004.
- Монитор 1920×1080, NVIDIA GeForce RTX 3060 Ti.
- Мышь ELECOM USB Mouse (`HID\VID_093A&PID_2521`).

Каталог игры в примерах: `E:\SteamLibrary\steamapps\common\Hammerfight`.

## Подготовка

1. Закройте игру: она перезаписывает `Config.ini` при выходе.
2. Создайте папку бэкапа, например `E:\MovedData\hammerfight-fix-backup`.
3. Скопируйте туда `Config.ini`, `Data\Dialogs\english.seria` и `media.script`.
4. Разрешите повышение прав через UAC: игра должна быть помечена `RUNASADMIN`. Если флага нет, добавьте его (путь подставьте свой):

```reg
Windows Registry Editor Version 5.00

[HKEY_CURRENT_USER\Software\Microsoft\Windows NT\CurrentVersion\App Compat Flags\Layers]
"E:\\SteamLibrary\\steamapps\\common\\Hammerfight\\Hammerfight.exe"="RUNASADMIN"
```

Закрытие процесса с повышением прав:

```powershell
Start-Process cmd.exe -ArgumentList '/c','taskkill /F /IM Hammerfight.exe' -Verb RunAs -Wait
```

## Шаг 1. Щелчки мыши

При первом запуске нажмите **Enter**. В `client.log` должна появиться строка:

```text
HIDS_ACTIVE Simple activated.
```

Это отключает поддержку двух и более мышей. После изменения перезапустите игру.

Если игра запущена с повышением прав, обычная отправка клавиши из процесса без повышения прав не сработает. Проверенный помощник:

```powershell
param([int]$ProcessId)
$ErrorActionPreference = 'SilentlyContinue'
$wsh = New-Object -ComObject WScript.Shell
$ok = $wsh.AppActivate($ProcessId)
Start-Sleep -Milliseconds 500
if ($ok) {
  $wsh.SendKeys('{ENTER}')
  Start-Sleep -Milliseconds 300
  $wsh.SendKeys('{ENTER}')
}
Write-Output "activate=$ok"
```

Запуск:

```powershell
$p = Get-Process Hammerfight | Select-Object -First 1
$helper = 'C:\Path\To\hammer-enter.ps1'   # путь к сохранённому выше скрипту
Start-Process powershell.exe -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File',$helper,'-ProcessId',"$($p.Id)" -Verb RunAs -Wait
```

Путь к `hammer-enter.ps1` замените на свой — в примере он был во временной папке пользователя `radict`.

## Шаг 2. Движение мыши

В `Config.ini` установите:

```ini
CORE_INIT_RI=false
CORE_MOUSE_OVERALL=1
```

- `CORE_INIT_RI=false` отключает Raw Input. В логе тогда будет `InputTask: not using RI.`
- При ошибочно заданном `CORE_MOUSE0` появляется `ERROR: getRI - GetRawInputBuffer failed.`
- Не задавайте вручную `CORE_MOUSE_SPEED0=1.0`: игра сама перезаписывает чувствительность (в примере `CORE_MOUSE_SPEED0=3.53333`).
- Сторонние HID-утилиты и драйверы могут мешать определению мыши; об этом есть предупреждение в `readme.txt` игры.

## Шаг 3. Разрешение и резкость

В `Config.ini`:

```ini
GFX_WINDOWED=false
GFX_SCREEN_WIDTH=1920
GFX_SCREEN_HEIGHT=1080
GFX_DSCREEN_WIDTH=96
GFX_DSCREEN_HEIGHT=54
```

Пояснение:

- `GFX_SCREEN_WIDTH/HEIGHT` задаёт реальный полноэкранный буфер. В `hge.log` должно быть `Mode: 1920 x 1080 x X8R8G8B8`.
- Исходные `GFX_DSCREEN_WIDTH=40` и `GFX_DSCREEN_HEIGHT=30` задают игровую область 4:3, из-за чего на экране 16:9 картинка растягивается.
- Значение `96×54` сохраняет прежний масштаб 20 px/unit и даёт пропорции 16:9. Текст становится заметно резче.
- Сами спрайты сделаны под 512×512, поэтому остаются мягкими. Это ограничение исходных ресурсов, а не настройки движка.

Записывайте `Config.ini` без UTF-8 BOM:

```powershell
$f = 'E:\SteamLibrary\steamapps\common\Hammerfight\Config.ini'
$t = [IO.File]::ReadAllText($f)
$t = $t -replace 'GFX_DSCREEN_WIDTH=40','GFX_DSCREEN_WIDTH=96' -replace 'GFX_DSCREEN_HEIGHT=30','GFX_DSCREEN_HEIGHT=54'
[IO.File]::WriteAllText($f, $t, (New-Object Text.UTF8Encoding($false)))
```

BOM вызывает ошибку `ERROR: Deser of APP failed, corrupted script.`

## Шаг 4. Русский язык

В Steam-версии исполняемый файл жёстко ссылается на `Data/Dialogs/english.seria`. Файла `russian.seria` и переключателя языка нет; в метаданных Steam указан только английский.

Решение — заменить этот текстовый файл на файл фанатской русификации, сохранив оригинал.

Конкретный проверенный источник:

- страница: <https://www.playground.ru/hammerfight/file/rusifikator_hammerfight_tekst_dlya_steam_versii-918998>
- файл: `Hammerfight_ru_s_text.exe`, 330341 байт;
- SHA256: `4d6e4720bea372be4fb8d7ba6290900f7cceb6884a17581906b532e0ecb9512d`;
- внутри RAR/SFX только `Data\Dialogs\english.seria` (202970 байт), без исполняемых файлов.

Сайт выдаёт ссылку через API `file.download` с задержкой 60 секунд; ссылка из ответа начинается с `//`, нужно добавить `https:`.

```powershell
$dir = 'E:\SteamLibrary\steamapps\common\Hammerfight'
Copy-Item "$dir\Data\Dialogs\english.seria" 'E:\MovedData\hammerfight-fix-backup\english.seria.before-ru' -Force
Copy-Item 'путь\к\скачанному\english.seria' "$dir\Data\Dialogs\english.seria" -Force
```

После этого в `client.log` должно быть:

```text
DM::load successful with Data/Dialogs/english.seria.
```

Меры безопасности:

- используйте только законно полученную копию игры;
- скачивайте русификатор только с доверенной страницы и сверяйте SHA256 с опубликованным значением;
- проверяйте архив: в описанном случае внутри был только один текстовый файл без исполняемых файлов.

## Проверка результата

Обязательные строки в логах:

```text
hge.log:      Mode: 1920 x 1080 x X8R8G8B8
client.log:   DM::load successful with Data/Dialogs/english.seria.
client.log:   InputTask: not using RI.
client.log:   HIDS_ACTIVE Simple activated.
```

Запрещённые строки:

```text
getRI - GetRawInputBuffer failed
Deser of APP failed, corrupted script
```

## Откат

1. Закройте игру.
2. Восстановите `Config.ini` из бэкапа.
3. Для русского языка верните `english.seria.before-ru`.
4. Убедитесь, что в `Config.ini` нет BOM.
5. Перезапустите игру.

## Неудачная попытка увеличения спрайтов

Проверялась идея сделать спрайты резче через 2×-увеличение атласа `menue.png` методом Nearest Neighbor и пропорциональное умножение координат `rect`/`hotspot` в `media.script`. В проекте 64 атласа, 1557 спрайтов, 143 анимации. На одном атласе это сломало отображение элементов меню, поэтому изменение было полностью откачено.

Вывод: простое увеличение атласов не является безопасным способом повысить резкость. Нужен аккуратный AI-апскейл с ручной проверкой каждого спрайта или новая перерисовка ресурсов.

## Инструкция для нейросетей и автоматических агентов

Ниже — детерминированный протокол. Не выдумывайте пути и не меняйте несколько параметров одновременно без проверки.

```yaml
goal: "Hammerfight 1.004 Steam: working mouse, 1920x1080, Russian text on Windows 10"
safety:
  - "Do not delete saves or registry keys without backup."
  - "Do not download copyrighted games or untrusted executables."
  - "Ask the user before any public publishing or real-world side effects."
preconditions:
  game_install: "E:\\SteamLibrary\\steamapps\\common\\Hammerfight"
  config: "{game_install}\\Config.ini"
  logs:
    - "{game_install}\\client.log"
    - "{game_install}\\hge.log"
  powershell: "powershell.exe -NoProfile -ExecutionPolicy Bypass"
steps:
  1:
    action: terminate_process
    command: "Start-Process cmd.exe -ArgumentList '/c','taskkill /F /IM Hammerfight.exe' -Verb RunAs -Wait"
    verify: "Get-Process Hammerfight returns nothing"
  2:
    action: backup
    files:
      - "Config.ini"
      - "Data/Dialogs/english.seria"
      - "media.script"
    to: "E:\\MovedData\\hammerfight-fix-backup"
  3:
    action: edit_config_no_bom
    set:
      CORE_INIT_RI: "false"
      CORE_MOUSE_OVERALL: "1"
      GFX_WINDOWED: "false"
      GFX_SCREEN_WIDTH: "1920"
      GFX_SCREEN_HEIGHT: "1080"
      GFX_DSCREEN_WIDTH: "96"
      GFX_DSCREEN_HEIGHT: "54"
    write_command: "[IO.File]::WriteAllText($f,$t,(New-Object Text.UTF8Encoding($false)))"
    forbidden:
      - "UTF-8 BOM"
      - "explicit CORE_MOUSE_SPEED0 edits"
      - "CORE_INIT_RI=true"
  4:
    action: install_russian_text
    source_url: "https://www.playground.ru/hammerfight/file/rusifikator_hammerfight_tekst_dlya_steam_versii-918998"
    file_name: "Hammerfight_ru_s_text.exe"
    expected_sha256: "4d6e4720bea372be4fb8d7ba6290900f7cceb6884a17581906b532e0ecb9512d"
    archive_content: "Data/Dialogs/english.seria only, no executables"
    target: "{game_install}\\Data\\Dialogs\\english.seria"
    api_delay_seconds: 60
  5:
    action: launch_and_send_enter
    launch: "steam://run/41100"
    helper_script: "hammer-enter.ps1 using WScript.Shell AppActivate(pid) and SendKeys('{ENTER}') twice"
    helper_run: "Start-Process powershell.exe -ArgumentList '-NoProfile','-ExecutionPolicy','Bypass','-File','<helper.ps1>','-ProcessId',\"$pid\" -Verb RunAs -Wait"
verification:
  required_log_lines:
    - "Mode: 1920 x 1080 x X8R8G8B8"
    - "InputTask: not using RI."
    - "HIDS_ACTIVE Simple activated."
    - "DM::load successful with Data/Dialogs/english.seria."
  forbidden_log_lines:
    - "getRI - GetRawInputBuffer failed"
    - "Deser of APP failed, corrupted script"
rollback:
  - "close game first"
  - "restore Config.ini and english.seria from backup"
  - "do not edit media.script or sprite atlases"
known_bad_approaches:
  - "GFX_WINDOWED=true with 800x600: blurry stretched image"
  - "CORE_INIT_RI=true: raw input error"
  - "2x sprite upscale in media.script: breaks UI, revert immediately"
```

## Где ещё доступна инструкция

- Постоянная ссылка на репозиторий: <https://github.com/2017radict/Hammerfight-Windows10-fix>
- Копия в GitHub Gist: <https://gist.github.com/2017radict/528122677f774f816d4578fb8e850934>

## Источники

- Обсуждение Steam «Русский язык в игре»: <https://steamcommunity.com/app/41100/discussions/0/864978109530387836/?l=russian>
- Руководство Steam «How to install v1.0005 and unlock Tribal Wars game mode»: <https://steamcommunity.com/sharedfiles/filedetails/?id=2597103960>
- Страница фанатского текстового русификатора: <https://www.playground.ru/hammerfight/file/rusifikator_hammerfight_tekst_dlya_steam_versii-918998>
- Steam Store API для AppID 41100: <https://store.steampowered.com/api/appdetails?appids=41100>
- Встроенный `readme.txt` игры.
