# Horizon 26.3 в Horizon Launcher: диагностика и решение на Windows

> **Статус:** успешно проверено на одном ПК с Windows.  
> **Сценарий:** Horizon Nightly `nightly-2026-11-08-fdc87ae`, Minecraft Java 26.3, локально импортированная установка, офлайн-запуск.  
> **Лаунчер:** в процессе диагностики обновлён до 0.2.58.  
> **Это не официальный патч Horizon и не универсальная гарантия.** Используйте инструкцию как подробный разбор одного воспроизводимого случая.

## Кратко

В рассматриваемом случае Horizon не мог завершить подготовку Minecraft 26.3. В логах встречались ошибки Java `AccessDeniedException`, сбои `javac.exe`, а затем ошибка загрузки реестров (`Registry Loading`). В ходе диагностики выяснилось, что Java не могла выполнить `Path.toRealPath()` для путей внутри профиля Windows, хотя обычное чтение JAR через `jar tf` проходило.

После запуска подготовки **для точной Nightly-версии** команда успешно завершилась:

```text
Prepared Horizon native-client resources: ...\nightly-2026-11-08-fdc87ae\.local\horizon-client\26.3
Done in 32.8 s
nightly-2026-11-08-fdc87ae is prepared.
```

После этого игра запустилась.

**Важная оговорка о причине:** в ACL профиля и `AppData\Roaming` были обнаружены нестандартные явные правила `Deny`, но предложенный скрипт изменения ACL завершился проверкой `Unexpected ACL` и не применил изменение. Поэтому нельзя утверждать, что именно удаление этих правил исправило проблему. Надёжно подтверждённые факты: Java-проверка `toRealPath()` сначала падала на путях профиля, позднее все проверенные пути стали возвращать `OK`; подготовка правильной Nightly после этого завершилась успешно.

## Симптомы и ошибки, которые встречались

В одном расследовании могли появляться несколько ошибок. Они не обязательно имеют одну причину.

### 1. Файлы Horizon не подготовлены

```text
Horizon's prepared game files are missing:
no horizon-resources.json in ...\.local\horizon-client\26.3
```

Это означает, что нужный файл подготовленных ресурсов отсутствует. Само по себе сообщение ещё не говорит, почему подготовка не завершилась.

### 2. Java не может разрешить реальный путь

```text
java.nio.file.AccessDeniedException: ...\server-game.jar
at ... WindowsLinkSupport.getRealPath
at ... WindowsPath.toRealPath
```

В данном случае обычное чтение архива проходило:

```powershell
jar tf $file > $null
```

Но отдельная проверка Java `Path.toRealPath()` не проходила. Это разные операции: успешное чтение файла не доказывает, что Java сможет разрешить его реальный путь.

### 3. Ошибка компилятора на этапе экспорта

```text
javac.exe failed (exit code: 1) while exporting block behavior
```

В `export.log` были сообщения `cannot find symbol` для классов Horizon, а в конце — `AccessDeniedException` при закрытии ZIP/JAR файловой системы для `jtracy-1.14.38.jar`.

После дальнейшей диагностики и подготовки другой версии экспорта поведение блоков прошло, но возник следующий сбой.

### 4. Ошибка экспорта реестров команд

```text
javac.exe failed (exit code: 1) while exporting command registries
```

Лог `command-registries-export.log` показывал сообщения Minecraft о сбое загрузки реестров. Также выводились предупреждения OSHI/WMI о системной информации и `NullPointerException` при формировании отчёта о сбое.

Не следует автоматически считать предупреждения OSHI/WMI первопричиной. В этом случае полезнее было сначала прочитать сообщение об исходном сбое реестров и проверить подготовочные пути.

## Что было проверено

- Папка и JAR существовали.
- У файла JAR не было атрибута ReadOnly.
- `jar tf` мог прочитать содержимое JAR.
- Файл не был Reparse Point / symlink.
- У пользователя были разрешения Full Control на многих вложенных папках.
- Java могла разрешить `C:\`, `C:\Users` и `C:\Users\Elena`, но первоначально возвращала `AccessDeniedException` для `AppData\Roaming` и вложенных путей.
- `Path.toRealPath()` и `File.getCanonicalPath()` в отдельном тесте вели себя по-разному: первый первоначально падал, второй возвращал путь.
- В ACL `C:\Users\Elena` обнаружилось явное правило Deny для пользователя, включавшее Delete и Read; в `AppData\Roaming` обнаружились явные Deny для Delete у пользователя и SYSTEM.
- WMI repository сообщил, что хранилище согласовано. Проверка счётчиков производительности не указала на отключённые PerfProc/PerfOS.
- Лаунчер обновили до 0.2.58.
- Подготовка была запущена именно для Nightly `nightly-2026-11-08-fdc87ae`, а не только для каталога `v0.2.54`.

## Безопасная диагностика

Открой PowerShell. Для чтения логов и запуска обычных диагностических проверок не нужно автоматически выдавать всем полный доступ. Администраторский PowerShell может понадобиться для отдельных проверок ACL, но не запускайте команды изменения разрешений, пока не поняли результат.

### 1. Найти executable лаунчера

```powershell
$launcher = "$env:LOCALAPPDATA\Programs\Horizon Launcher\horizon-launcher.exe"
Test-Path $launcher
```

Если команда возвращает `True`, executable найден.

### 2. Посмотреть поддерживаемые команды

```powershell
& $launcher --help
```

Синтаксис команд может различаться между версиями. Сверяйтесь с `--help` установленной версии.

### 3. Проверить Java и JAR

```powershell
java -version

$file = "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae\.local\minecraft\26.3\jars\server-game.jar"
Test-Path $file
Get-Item -LiteralPath $file -Force | Format-List FullName,Length,Attributes,IsReadOnly
jar tf $file > $null
```

Успешное выполнение `jar tf` подтверждает, что Java смогла прочитать ZIP/JAR, но не гарантирует, что `toRealPath()` тоже пройдёт.

### 4. Проверить реальный путь через Java

Создай тестовый Java source file:

```powershell
@'
import java.nio.file.Path;

public class RealPathProbe {
    public static void main(String[] args) {
        for (String value : args) {
            try {
                System.out.println("OK   " + Path.of(value).toRealPath());
            } catch (Exception e) {
                System.out.println("FAIL " + value);
                System.out.println("     " + e);
            }
        }
    }
}
'@ | Set-Content "$env:TEMP\RealPathProbe.java" -Encoding ascii
```

Запусти проверку цепочки путей (замени имя Nightly при необходимости):

```powershell
$base = "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae"
$paths = @(
    "$env:APPDATA",
    "$env:APPDATA\horizon-launcher",
    "$base",
    "$base\.local",
    "$base\.local\minecraft",
    "$base\.local\minecraft\26.3",
    "$base\.local\minecraft\26.3\jars",
    "$base\.local\minecraft\26.3\jars\server-game.jar"
)

java "$env:TEMP\RealPathProbe.java" $paths
```

- `OK` означает, что Java смогла разрешить конкретный путь.
- `FAIL` показывает точный путь, на котором проверка остановилась.

Это диагностический тест. Он сам ничего не меняет.

### 5. Проверить ACL и сохранить резервную копию

```powershell
$profile = $env:USERPROFILE
$acl = Get-Acl -LiteralPath $profile
$acl.Sddl | Set-Content "$env:TEMP\profile-acl-backup.sddl" -Encoding ASCII

icacls "$env:USERPROFILE"
icacls "$env:USERPROFILE\AppData"
icacls "$env:APPDATA"

(Get-Acl -LiteralPath "$env:APPDATA") |
    Format-List Owner,AreAccessRulesProtected,Access
```

Для подробного просмотра правил:

```powershell
$targets = @(
    "$env:USERPROFILE",
    "$env:USERPROFILE\AppData",
    "$env:APPDATA",
    "$env:APPDATA\horizon-launcher"
)

foreach ($p in $targets) {
    Write-Host "`n========== $p ==========" -ForegroundColor Cyan
    try {
        $a = Get-Acl -LiteralPath $p -ErrorAction Stop
        "OWNER: $($a.Owner)"
        "PROTECTED: $($a.AreAccessRulesProtected)"
        $a.Access | Select-Object IdentityReference,AccessControlType,
            FileSystemRights,IsInherited,InheritanceFlags |
            Format-Table -AutoSize -Wrap
    } catch {
        Write-Host "ACL ERROR: $($_.Exception.Message)" -ForegroundColor Red
    }
}
```

**Не копируйте найденные здесь Deny-правила и не удаляйте их автоматически.** Запрет Delete сам по себе не означает запрет чтения; у каждого правила нужно понять точные права, пользователя/группу, наследование и область действия. Нельзя обещать, что удаление Deny исправит `toRealPath()`.

## Подготовить нужную версию Horizon

Ключевой практический момент этого случая: подготовку нужно выполнять для той Nightly, которую вы собираетесь запускать.

Сначала проверь список доступных версий:

```powershell
& $launcher versions
```

Уточни точную метку установленной Nightly в выводе. Затем используй её в команде. Пример из успешного случая:

```powershell
& $launcher prepare --version nightly-2026-11-08-fdc87ae --minecraft "D:\minetcraft"
```

Здесь `D:\minetcraft` — **пример пути из одного случая**. Укажи свой реальный каталог Minecraft Java 26.3, не копируй этот путь вслепую.

Важные замечания:

- `prepare` — команда лаунчера, поэтому вызывай `horizon-launcher.exe prepare`, а не несуществующий в PATH псевдоним `horizon-launcher prepare`.
- Параметр `--version` направляет подготовку на выбранную установленную версию.
- Для офлайн-сценария с локально импортированной игрой используй локальный каталог Minecraft. Не выбирай режим скачивания с Mojang, если у тебя нет подходящего Microsoft/Minecraft аккаунта или ты намеренно используешь локальный импорт.
- Не удаляй целиком `AppData`, профиль Windows или папку Horizon без резервной копии.

## Проверить результат

После завершения подготовки ожидай успешное итоговое сообщение, похожее на:

```text
Prepared Horizon native-client resources: ...\.local\horizon-client\26.3
Done in 32.8 s
nightly-2026-11-08-fdc87ae is prepared.
```

Дополнительно можно проверить наличие файла:

```powershell
$resources = "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae\.local\horizon-client\26.3\horizon-resources.json"
Test-Path $resources
```

Ожидаемое значение — `True`. После этого запусти ту же Nightly из Horizon Launcher.

## Где искать логи

Лог подготовки может выводить путь к файлу экспорта. В этом случае открывай именно указанный файл.

Примеры путей из данного случая:

```powershell
Get-Content "$env:APPDATA\horizon-launcher\versions\v0.2.54\.local\horizon-client-block-behavior\export.log" -Tail 100

Get-Content "$env:APPDATA\horizon-launcher\versions\nightly-2026-11-08-fdc87ae\.local\horizon-client\26.3\command-registries-export.log" -Tail 120
```

Путь зависит от версии лаунчера и целевой версии. Не предполагай, что у всех пользователей он совпадает.

Также можно посмотреть доступные параметры логов:

```powershell
& $launcher logs --help
```

## Типовые сообщения и что делать

| Сообщение | Значение / следующий шаг |
|---|---|
| `Horizon's prepared game files are missing` | Подготовка для этой версии не завершена или ресурсы лежат в другом version-профиле. Проверь метку версии и `horizon-resources.json`. |
| `AccessDeniedException` + `toRealPath` | Проверь Java `toRealPath()` для каждого родительского каталога. Затем исследуй ACL, не выдавая массовые права. |
| `jar tf` проходит, а `toRealPath` нет | Это разные операции. Не считай успешный просмотр JAR доказательством исправности доступа к реальному пути. |
| `javac.exe failed` | Найди указанный в сообщении export log и смотри первые содержательные ошибки, а не только последнюю строку stack trace. |
| `cannot find symbol` | Может быть следствием неправильного/неполного classpath или незавершённой подготовки. Нужен весь контекст export log. |
| `Registry Loading`, `Registry must be non-empty` | Ищи более раннюю ошибку загрузки данных/ванильных ресурсов в том же логе. Не считай OSHI-ошибку автоматически первопричиной. |
| `Invalid Class Win32_...`, `PDH Counter`, OSHI `process is null` | Системная диагностика Windows может не работать и мешать формированию crash report. Сначала отдели её от первичной ошибки Minecraft/Horizon. |
| `horizon-launcher` не распознана PowerShell | Используй полный путь к `horizon-launcher.exe` через оператор `&`. |

## Чего не делать

- Не запускай `icacls C:\Users\... /reset /T` без понимания последствий.
- Не выдавай `Everyone:F` на весь профиль или весь диск.
- Не удаляй папки Windows `AppData` и не переименовывай весь профиль.
- Не удаляй `D:\minetcraft` — локальный исходник может быть нужен для повторной подготовки.
- Не считай каждую ошибку `AccessDeniedException` одинаковой: проверь точный путь и операцию.
- Не утверждай, что Deny/Delete обязательно был причиной. В этом кейсе его наличие было обнаружено, но конкретное изменение ACL не было успешно применено.

## История успешного случая

1. Первоначальный запуск сообщал об отсутствии `horizon-resources.json`.
2. Проверка Windows ACL и JAR не выявила простой проблемы чтения файла.
3. `prepare` запускался, но падал на экспорте block behavior, затем на command registries.
4. В логах обнаружили ошибки Java `toRealPath()` для путей под `AppData\Roaming`.
5. Создали отдельный Java-тест, который последовательно проверял все родительские каталоги и JAR.
6. На последующей проверке все 8 путей вернули `OK`.
7. Запустили `prepare` с явным параметром `--version nightly-2026-11-08-fdc87ae` и локальным каталогом Minecraft.
8. Все этапы экспорта завершились, Horizon сообщил, что Nightly подготовлена.
9. Игра успешно запустилась.

Этот порядок — полезная диагностическая история, но не доказательство того, что один конкретный ACL-флаг был единственной причиной.

## Что приложить, если не помогло

При создании issue не публикуйте личные данные. Удалите имя Windows-пользователя, локальные пути с личной информацией, имена аккаунтов, токены и любые секреты.

Полезно приложить:

- Версию Horizon Launcher и точный tag Nightly.
- Версию Windows и вывод `java -version`.
- Точный текст ошибки и 50–150 строк соответствующего export log.
- Вывод `RealPathProbe` с заменённым именем пользователя.
- Результаты `icacls`/ACL только для затронутых каталогов, после удаления личных имён.
- Указание, запускаете ли вы локально импортированную копию или используете официальный sign-in flow.

---

**Disclaimer:** Horizon is an independent project. This repository is a community troubleshooting note and is not affiliated with or endorsed by the Horizon developers or Mojang/Microsoft.
