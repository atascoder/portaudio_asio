# Изменения в PortAudio для вывода на каналы 3 и 4

## Модификация кода

### 1. pa_asio.cpp - Жёсткое задание выходных каналов

**Файл:** `portaudio/src/hostapi/asio/pa_asio.cpp`

**Что было изменено:**
- Добавлена логика в функцию `OpenStream()` для автоматического переназначения выходных каналов с каналов 1-2 на каналы 3-4
- При создании 2-канального выходного потока код теперь автоматически переназначает эти каналы на каналы 3 и 4 (индексы 2 и 3)
- Добавлена правильная обработка памяти и освобождение выделенных буферов

**Детали реализации:**
```cpp
/* HARDCODED: Force output to channels 3 and 4 (indices 2 and 3) */
if( outputChannelCount == 2 )
{
    // Выделяем память для селекторов каналов
    if( !outputChannelSelectors )
    {
        outputChannelSelectors = (int*)malloc( outputChannelCount * sizeof(int) );
        if( !outputChannelSelectors )
        {
            result = paInsufficientMemory;
            goto error;
        }
    }
    // Устанавливаем жесткие селекторы каналов 2 и 3 (каналы 3 и 4)
    outputChannelSelectors[0] = 2;  /* Channel 3 */
    outputChannelSelectors[1] = 3;  /* Channel 4 */
    PA_DEBUG(("OpenStream: Forcing output to channels 3 and 4 (indices 2 and 3)\n"));
}
```

### 2. build-windows.yml - GitHub Actions workflow

**Файл:** `.github/workflows/build-windows.yml`

**Что было изменено:**
Создан правильный workflow для сборки PortAudio DLL на Windows в GitHub Actions с поддержкой:
- Двух архитектур (x64 и Win32)
- CMake для конфигурации и сборки
- Visual Studio 2022
- Правильной обработки артефактов (DLL и LIB файлы)

**Основные шаги:**
1. Checkout репозитория с подмодулями
2. Setup CMake
3. Конфигурация CMake для каждой архитектуры (x64 и Win32)
4. Сборка в режиме Release
5. Копирование артефактов (portaudio.dll и portaudio.lib)
6. Загрузка артефактов в GitHub

## Как использовать

### При компиляции на Windows:
```bash
mkdir build
cd build
cmake .. -G "Visual Studio 17 2022" -A x64 -DCMAKE_BUILD_TYPE=Release
cmake --build . --config Release
```

### В GitHub Actions:
При push на ветку `main` или pull request будет автоматически:
1. Скомпилирована DLL для x64 архитектуры
2. Скомпилирована DLL для Win32 архитектуры
3. Загружены артефакты для скачивания

## Результаты сборки

После успешной сборки будут доступны:
- `portaudio_x64.dll` / `portaudio_x64.lib` - 64-битная версия
- `portaudio_win32.dll` / `portaudio_win32.lib` - 32-битная версия

## Важно

Все выходные потоки с 2 каналами будут автоматически выводиться на каналы 3 и 4 вашего аудиоинтерфейса вместо каналов 1 и 2.
