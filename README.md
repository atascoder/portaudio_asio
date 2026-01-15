# PortAudio ASIO Channel Redirection

## Overview

This repository contains a modified version of PortAudio with ASIO support for redirecting audio output from channels 1-2 to channels 3-4 on your ASIO audio interface (specifically tested with Soundcraft EVO 4).

### Problem Solved

PortAudio was outputting audio to ASIO channels 0-1 (displayed as channels 1-2) instead of the requested channels 2-3 (displayed as 3-4). This has been fixed through proper ASIO buffer index mapping.

## Quick Start

### Windows DLL Build

See [WINDOWS_BUILD_GUIDE.md](WINDOWS_BUILD_GUIDE.md) for detailed instructions.

**Quick version:**
```powershell
cd portaudio
mkdir build
cd build
cmake .. -G "Visual Studio 17 2022" -A x64 -DPA_USE_ASIO=ON -DCMAKE_PREFIX_PATH="C:\path\to\asio"
cmake --build . --config Release
```

### Linux/Unix Build

```bash
cd portaudio
mkdir build
cd build
cmake .. -G "Unix Makefiles"
make -j4
```

## 🔍 How to Debug (Все еще звук идет в неправильные каналы?)

1. **Скомпилируйте новую DLL** с файловым логированием
2. **Замените вашу текущую portaudio.dll** 
3. **Откройте лог** в: `C:\Users\<YourName>\AppData\Local\portaudio_asio_debug.log`

**Подробные инструкции:**
- [HOW_TO_DEBUG.md](HOW_TO_DEBUG.md) - Полное руководство по отладке
- [LOG_LOCATION.md](LOG_LOCATION.md) - Где найти лог файл и как его открыть

## Understanding the Fix

The issue and solution are explained in detail in [SOLUTION_EXPLANATION.md](SOLUTION_EXPLANATION.md).

**TL;DR:** After ASIO allocates buffers, we need to map them back to the correct physical channels by searching for the matching `channelNum`, not by assuming direct array index correspondence.

## Technical Details

- **Modified File**: [portaudio/src/hostapi/asio/pa_asio.cpp](portaudio/src/hostapi/asio/pa_asio.cpp)
- **Key Changes**: 
  - Hardcoded output channel selector to channels 3-4 (indices 2-3)
  - Implemented proper ASIO buffer index mapping after ASIOCreateBuffers()
  - Added comprehensive debug logging to file: `portaudio_asio_debug.log`

### For Deep Understanding

- [ANALYSIS.md](ANALYSIS.md) - Technical deep dive into the root cause
- [SOLUTION_EXPLANATION.md](SOLUTION_EXPLANATION.md) - Detailed explanation of the solution
- [WINDOWS_BUILD_GUIDE.md](WINDOWS_BUILD_GUIDE.md) - Compilation instructions

## Debug Output

When opening a stream, you'll see debug output like:

```
OpenStream: Setting outputChannelSelectors[0] = 2 (channel 3)
OpenStream: Setting outputChannelSelectors[1] = 3 (channel 4)
=== AFTER ASIOCreateBuffers ===
asioBufferInfos[2]: OUTPUT, channelNum=2, buffers=(0x..., 0x...)
asioBufferInfos[3]: OUTPUT, channelNum=3, buffers=(0x..., 0x...)
Output channel 0 (phys 2) -> ASIO buffer index 2
Output channel 1 (phys 3) -> ASIO buffer index 3
```

This shows the channel mapping is working correctly.

## Compilation Variants

The code can be easily modified for different channel mappings by changing the hardcoded values around line 2121 in `pa_asio.cpp`:

```cpp
outputChannelSelectors[i] = 2 + i;  /* Channels 3, 4, 5, ... */
```

Change `2 + i` to any desired starting channel.

## GitHub Actions

Automated Windows DLL builds are available via GitHub Actions. See [WINDOWS_BUILD_GUIDE.md](WINDOWS_BUILD_GUIDE.md#option-3-github-actions-automated).

## Testing Your Build

1. Compile the library
2. Copy the DLL/SO to your application
3. Run your audio application
4. Monitor debug output for channel mapping information
5. Verify audio comes from channels 3-4 on your interface

## Compatibility

- **Tested**: Soundcraft EVO 4 audio interface
- **Should work**: Any ASIO device with 4+ channels
- **Requires**: Windows (for ASIO) or appropriate ASIO driver on other platforms

## Project Structure

```
portaudio_asio/
├── portaudio/          # PortAudio library (submodule with modifications)
├── asio/               # ASIO SDK (submodule)
├── .github/workflows/  # CI/CD configurations
├── SOLUTION_EXPLANATION.md
├── WINDOWS_BUILD_GUIDE.md
├── ANALYSIS.md
└── README.md
```

## Key Commits

- **36dba6e**: CRITICAL FIX - Implement proper ASIO buffer index mapping
- **5079026**: Add detailed debug logging for channel mapping
- **f9e5f82**: Improve channel mapping logic for any channel count
- **5c214cb**: Initial ASIO channel redirection to channels 3-4

## References

- [PortAudio Official](http://www.portaudio.com/)
- [ASIO Documentation](https://en.wikipedia.org/wiki/Audio_Stream_Input/Output)
- Modified code with detailed comments in `portaudio/src/hostapi/asio/pa_asio.cpp`

## Support

For issues or questions:
1. Check [SOLUTION_EXPLANATION.md](SOLUTION_EXPLANATION.md) for detailed technical information
2. Review [WINDOWS_BUILD_GUIDE.md](WINDOWS_BUILD_GUIDE.md) troubleshooting section
3. Check [ANALYSIS.md](ANALYSIS.md) for root cause explanation
4. Review debug output when running your application
