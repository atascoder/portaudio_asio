# Как просмотреть логи и отладить проблему с каналами

## TL;DR (Быстрый старт)

1. **Скомпилируйте DLL для Windows** с помощью GitHub Actions (`.github/workflows/build-windows.yml`)
2. **Замените вашу текущую portaudio.dll** на новую из артифактов
3. **Запустите ваше приложение**
4. **Откройте лог** в: `%USERPROFILE%\AppData\Local\portaudio_asio_debug.log`
   - Полный путь: `C:\Users\<ВашИмя>\AppData\Local\portaudio_asio_debug.log`

## Подробные инструкции

### 1. Получить скомпилированную DLL

**Способ А: GitHub Actions (рекомендуется)**
- Перейдите в репозиторий: https://github.com/atascoder/portaudio_asio
- Откройте вкладку **Actions**
- Найдите последний workflow **Build PortAudio for Windows**
- Нажмите на него
- Скачайте артефакт **portaudio-windows-x64-release**
- Извлеките `portaudio.dll` и `portaudio.lib`

**Способ Б: Локальная компиляция**
- Используйте Visual Studio 2022 Community (бесплатно)
- Следуйте инструкциям в [WINDOWS_BUILD_GUIDE.md](WINDOWS_BUILD_GUIDE.md)

### 2. Заменить DLL

- Найдите, где установлен ваш текущий `portaudio.dll`
- Сделайте резервную копию: `portaudio.dll.backup`
- Замените на новый файл

### 3. Запустить приложение

- Запустите приложение, которое использует PortAudio
- Создайте и начните воспроизведение аудиопотока с 2 выходными каналами

### 4. Найти и открыть лог

На **Windows**:

```
C:\Users\<ВашИмя>\AppData\Local\portaudio_asio_debug.log
```

Или используйте команду в PowerShell:
```powershell
# Открыть в блокноте
notepad $env:USERPROFILE\AppData\Local\portaudio_asio_debug.log

# Или просмотреть в PowerShell
Get-Content $env:USERPROFILE\AppData\Local\portaudio_asio_debug.log -Tail 100  # последние 100 строк
```

## Что ищем в логе

### Пример ПРАВИЛЬНОГО логирования (каналы 3-4):

```
=== OpenStream called ===
sampleRate=48000.000000, framesPerBuffer=256
Setting outputChannelSelectors[0] = 2 (physical channel 3)
Setting outputChannelSelectors[1] = 3 (physical channel 4)
Forcing output to channels starting from 3. Total output channels: 2
=== Building output channel mapping ===
outputChannelCount=2, inputChannelCount=0
Looking for physical channel 2 (user channel 0)
  asioBufferInfos[0].channelNum = 2 MATCH!
Output channel 0 (phys 2) -> ASIO buffer index 0 (SUCCESS)
Looking for physical channel 3 (user channel 1)
  asioBufferInfos[1].channelNum = 3 MATCH!
Output channel 1 (phys 3) -> ASIO buffer index 1 (SUCCESS)
Output buffer assignment: user_channel=0, asioIndex=0, channelNum=2, buffers=(0x12345678,0x87654321)
Output buffer assignment: user_channel=1, asioIndex=1, channelNum=3, buffers=(0xabcdef00,0x00fedcba)
=== End output channel mapping ===
```

**Признаки успешного маппирования:**
- ✅ `Setting outputChannelSelectors[0] = 2` (канал 3)
- ✅ `Setting outputChannelSelectors[1] = 3` (канал 4)
- ✅ `MATCH!` для каналов 2 и 3
- ✅ `(SUCCESS)` вместо `(NOT FOUND)` или `(FALLBACK)`

### Пример ПРОБЛЕМНОГО логирования (каналы 1-2):

```
Output channel 0 (phys 2) -> FALLBACK to index 0 (NOT FOUND in ASIO buffers!)
Output channel 1 (phys 3) -> FALLBACK to index 1 (NOT FOUND in ASIO buffers!)
```

**Это означает:**
- ❌ ASIO драйвер НЕ создал буферы для каналов 2 и 3
- ❌ Вместо этого создал буферы для каналов 0 и 1
- ❌ Нам пришлось использовать fallback (эти же каналы 0 и 1)

## Шаги для дополнительной отладки

### Если видите "FALLBACK" в логе:

Это означает, что ASIO драйвер вашего интерфейса НЕ поддерживает выбор канала через `ASIOCreateBuffers`. Нужны дополнительные шаги:

1. **Проверьте, какие каналы доступны:**
   ```cpp
   // В логе должны быть строки вроде:
   // asioBufferInfos[0].channelNum = ?
   // asioBufferInfos[1].channelNum = ?
   ```

2. **Измените код на нужные каналы:**
   - Найдите в [pa_asio.cpp](portaudio/src/hostapi/asio/pa_asio.cpp) строку ~2171:
   ```cpp
   outputChannelSelectors[i] = 2 + i;  /* Измените 2 на нужный номер канала */
   ```
   - Если драйвер создает буферы для каналов 0 и 1, но вы хотите использовать каналы 3 и 4, нужна другая стратегия (см. ниже)

### Если звук все равно идет в неправильные каналы:

Это может быть проблема с ASIO драйвером вашего интерфейса. Некоторые драйверы не уважают `channelNum` в `ASIOBufferInfo`.

**Решения:**
1. **Обновите драйвер** вашего звукового интерфейса
2. **Используйте драйвер ASIO "Universal"** вместо фирменного (если доступен)
3. **Переподключитесь** к интерфейсу
4. **Перезагрузитесь** (ASIO драйверы часто требуют перезагрузки)

## Как расшифровать буферы в логе

```
buffers=(0x12345678,0x87654321)
```

Это два указателя на буферы для двойной буферизации (двойного буферирования):
- `0x12345678` - буфер 0 для текущего фрейма
- `0x87654321` - буфер 1 для следующего фрейма

Если они одинаковые, это ошибка:
```
buffers=(0x12345678,0x12345678)  # ❌ Плохо! Одинаковые буферы
```

## Отключение логирования (если нужно)

Логирование автоматически включается и отключается. Если хотите отключить явно:

1. В коде замените `PaAsio_OpenDebugLog();` на пустую функцию
2. Или просто удалите файл лога:
```powershell
Remove-Item $env:USERPROFILE\AppData\Local\portaudio_asio_debug.log
```

## Если чего-то не понимаете

Свяжитесь со мной и отправьте:
1. **Содержимое лога** (`portaudio_asio_debug.log`)
2. **Модель вашего звукового интерфейса** (например, Behringer UMC404HD)
3. **Сообщение об ошибке** (если есть)

---

**Важно:** Это файловое логирование может чуть замедлить производительность (из-за записи на диск). Если нужна максимальная производительность, отключите его в production коде.
