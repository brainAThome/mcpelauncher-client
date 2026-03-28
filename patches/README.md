# mcpelauncher-client — NVIDIA & Audio Fixes for Linux

## Patches for [minecraft-linux/mcpelauncher-client](https://github.com/minecraft-linux/mcpelauncher-client) (`ng` branch)

> **PR:** [#133](https://github.com/minecraft-linux/mcpelauncher-client/pull/133)  
> **Author:** brainAThome  
> **Date:** 2026-03-15  
> **Status:** Tested & Verified  

---

## Table of Contents

- [Overview](#overview)
- [Affected Hardware & Software](#affected-hardware--software)
- [Fix 1: NVIDIA GL_MAX_IMAGE_UNITS Crash](#fix-1-nvidia-gl_max_image_units-crash)
  - [Symptoms](#11-symptoms)
  - [Root Cause Analysis](#12-root-cause-analysis)
  - [The OpenGL Image Unit Limit](#13-the-opengl-image-unit-limit)
  - [Why Only Image Bindings](#14-why-only-image-bindings)
  - [Implementation Details](#15-implementation-details)
  - [Code Flow Diagram](#16-code-flow-diagram)
- [Fix 2: FMOD Audio Initialization Failure](#fix-2-fmod-audio-initialization-failure)
  - [Symptoms](#21-symptoms)
  - [Root Cause Analysis](#22-root-cause-analysis)
  - [The AAudio Bridge Architecture](#23-the-aaudio-bridge-architecture)
  - [Implementation Details](#24-implementation-details)
  - [Audio Pipeline Diagram](#25-audio-pipeline-diagram)
- [Files Changed](#files-changed)
- [Building from Source](#building-from-source)
- [Applying the Patches](#applying-the-patches)
- [Verification & Testing](#verification--testing)
- [Troubleshooting](#troubleshooting)
- [Minecraft Version Update Guide (Step-by-Step)](#minecraft-version-update-guide-step-by-step)
  - [Step 1: Download the New Minecraft Version](#step-1-download-the-new-minecraft-version)
  - [Step 2: Update the DRM Mod](#step-2-update-the-drm-mod)
  - [Step 3: Verify/Rebuild the Patched Binary](#step-3-verifyrebuild-the-patched-binary)
  - [Step 4: Test](#step-4-test)
  - [Step 5: Troubleshooting Version Updates](#step-5-troubleshooting-version-updates)
  - [Quick Reference: Important Paths](#quick-reference-important-paths)
  - [Quick Reference: Important URLs](#quick-reference-important-urls)
  - [Quick Reference: Important Commands](#quick-reference-important-commands)
  - [Version History](#version-history)
  - [WARNING: Two Different Launchers!](#warning-two-different-launchers)
- [Technical Background](#technical-background)

---

## Overview

This patch set resolves two critical issues in `mcpelauncher-client` that affect **NVIDIA GPU users** and **PipeWire audio users** on Linux:

| # | Issue | Impact | Affected Users |
|---|-------|--------|----------------|
| 1 | SIGABRT crash on NVIDIA GPUs | Game crashes immediately when entering a world | All NVIDIA desktop GPUs (GL_MAX_IMAGE_UNITS = 8) |
| 2 | No audio output via SDL3 backend | No sound in-game, FMOD error code 28 | PipeWire users, some PulseAudio setups |

Both fixes are non-invasive, self-contained in the GL/audio interception layers, and do not modify any game logic or external dependencies.

---

## Affected Hardware & Software

### Verified Test System

| Component | Details |
|-----------|---------|
| **OS** | Kubuntu 24.04.4 LTS (Noble), Kernel 6.17.0-19-generic |
| **GPU** | NVIDIA GeForce RTX 4080, Driver 590.48.01 |
| **CPU** | Intel Core i9-10850K @ 3.60 GHz |
| **Audio** | PipeWire 1.0.5 (with PulseAudio/ALSA compatibility) |
| **Audio Devices** | Yamaha ZG01, Astro A50, Razer Kiyo Pro, HDMI Audio |
| **Minecraft** | Bedrock Edition 1.26.10.4 (Android x86_64 via mcpelauncher) |
| **Compiler** | Ubuntu Clang 18.1.3 |

### Potentially Affected GPUs (Fix 1)

Any NVIDIA desktop GPU reporting `GL_MAX_IMAGE_UNITS = 8`:

- GeForce RTX 4090, 4080, 4070 Ti/Super, 4060 Ti/Super
- GeForce RTX 3090, 3080, 3070, 3060 (Ti)
- GeForce RTX 2080, 2070, 2060 (Ti/Super)
- GeForce GTX 1080, 1070, 1060
- All Quadro/Tesla cards with proprietary NVIDIA driver

AMD and Intel GPUs typically have `GL_MAX_IMAGE_UNITS >= 8` and may also be affected on some driver versions.

### Potentially Affected Audio Setups (Fix 2)

- Any Linux system using **PipeWire** as audio server
- Systems where `SDL_GetAudioDeviceFormat(SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK, ...)` returns failure
- Some **PulseAudio** setups where the default playback device format query fails

---

## Fix 1: NVIDIA GL_MAX_IMAGE_UNITS Crash

### 1.1 Symptoms

- Game crashes with **Signal 6 (SIGABRT)** immediately after entering a world
- Crash occurs in the **"Rendering Pool"** thread
- Stack trace points to `libminecraftpe.so` → `std::terminate()` → `__gnu_cxx::__verbose_terminate_handler()`
- The main menu renders correctly; the crash triggers when compute shaders are loaded for world rendering

**Example crash log:**
```
01:17:22 Error [Crash] Signal 6 in thread "Rendering Pool" at
  [libminecraftpe.so + 0x5dc9898]
  [libminecraftpe.so + 0x5dc9830]
  ...
  gsignal
  abort
  __gnu_cxx::__verbose_terminate_handler()
  __cxxabiv1::__terminate(void (*)())
  std::terminate()
```

### 1.2 Root Cause Analysis

Minecraft Bedrock Edition uses **GLSL compute shaders** for deferred rendering pipeline operations on the world. These shaders declare `image2DArray` uniforms with binding indices exceeding 7:

```glsl
layout(binding = 9, rgba16f) uniform writeonly image2DArray outputBuffer;
```

The OpenGL specification defines `GL_MAX_IMAGE_UNITS` as the maximum number of image units available. On NVIDIA desktop GPUs with the proprietary driver, this value is **8** (units 0–7). A `binding = 9` declaration references unit index 9, which is **out of range**.

**What happens:**
1. Minecraft loads a compute shader with `binding = 9` for an `image2DArray`
2. `glCompileShader()` fails because binding 9 exceeds `GL_MAX_IMAGE_UNITS` (8)
3. Minecraft does not check the compile status (the game was designed for mobile GPUs where `GL_MAX_IMAGE_UNITS` is typically higher or the binding is valid)
4. `glLinkProgram()` fails as well
5. The game attempts to use the invalid program → undefined behavior → `std::terminate()` → SIGABRT

### 1.3 The OpenGL Image Unit Limit

OpenGL defines several independent binding point limits:

| Limit Constant | Typical NVIDIA Value | Applies To |
|----------------|---------------------|------------|
| `GL_MAX_IMAGE_UNITS` | **8** | `image2D`, `image2DArray`, `image3D`, `imageCube`, etc. |
| `GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS` | 32–192 | `sampler2D`, `sampler2DArray`, `samplerCube`, etc. |
| `GL_MAX_UNIFORM_BUFFER_BINDINGS` | 84 | `uniform` blocks |
| `GL_MAX_SHADER_STORAGE_BUFFER_BINDINGS` | 96 | `buffer` blocks |

**Critical insight:** Only `image*` type uniforms are limited to 8 units. Sampler bindings, UBO bindings, and SSBO bindings have much higher limits and must **never** be remapped.

### 1.4 Why Only Image Bindings

An earlier version of this fix incorrectly remapped **all** `binding = N` declarations (including samplers and UBOs). This caused:

- **Wrong texture colors**: Sampler bindings shifted by `% 8` point to the wrong texture units, causing textures to be sampled from incorrect sources
- **Visual glitches**: UBO bindings shifted to wrong data buffers

The fix must specifically target only GLSL lines containing `image` type declarations. The line-based approach is reliable because:

1. GLSL requires `layout(binding = N)` on the same line as the uniform declaration
2. `image2D`, `image2DArray`, `image3D`, etc. all contain the substring `"image"`
3. Sampler types (`sampler2D`, `samplerCube`) do **not** contain `"image"`

### 1.5 Implementation Details

**File:** `src/fake_egl.cpp`

#### Hook: `glShaderSource`

```
For each shader source string:
  For each line:
    IF line contains "image":
      Find all binding = N patterns via regex
      IF N >= 8:
        Replace with N % 8
    ELSE:
      Pass line through unchanged
```

The regex pattern `(binding\s*=\s*)(\d+)` matches the binding qualifier in GLSL layout declarations. The `\s*` accounts for varying whitespace styles.

#### Hook: `glBindImageTexture`

```
IF unit parameter >= 8:
  Remap to unit % 8
Pass to real glBindImageTexture
```

This keeps the API call consistent with the shader modifications — the same remapping applied at GLSL source level must also be applied at the `glBindImageTexture` call site.

#### Hooks: `glCompileShader` / `glLinkProgram`

These add **error logging** only. After calling the real function, they check `GL_COMPILE_STATUS` / `GL_LINK_STATUS` and log any errors via `glGetShaderInfoLog` / `glGetProgramInfoLog`. This aids debugging but does not modify behavior.

### 1.6 Code Flow Diagram

```
Minecraft calls glShaderSource(shader, source_code)
        │
        ▼
┌─────────────────────────────────────┐
│  FakeEGL: glShaderSource hook       │
│                                     │
│  For each line in source:           │
│    Does line contain "image"?       │
│    ┌──YES──┐         ┌──NO──┐       │
│    │ Scan for         │ Pass  │      │
│    │ binding=N        │ through│     │
│    │ If N>=8:         │       │      │
│    │   N = N % 8      │       │      │
│    └───────┘         └───────┘      │
│                                     │
│  Call real glShaderSource(modified)  │
└─────────────────────────────────────┘
        │
        ▼
Minecraft calls glCompileShader(shader)
        │
        ▼
┌─────────────────────────────────────┐
│  FakeEGL: glCompileShader hook      │
│  Call real glCompileShader           │
│  Check GL_COMPILE_STATUS             │
│  If failed: log error via            │
│    glGetShaderInfoLog                │
└─────────────────────────────────────┘
        │
        ▼
Minecraft calls glBindImageTexture(unit=9, ...)
        │
        ▼
┌─────────────────────────────────────┐
│  FakeEGL: glBindImageTexture hook   │
│  unit >= 8? → unit = unit % 8 = 1  │
│  Call real glBindImageTexture(1,..) │
└─────────────────────────────────────┘
```

---

## Fix 2: FMOD Audio Initialization Failure

### 2.1 Symptoms

- **No sound** in Minecraft despite system audio working perfectly
- The log shows: `FMOD_System_Init returned 28` (= `FMOD_ERR_OUTPUT_INIT`)
- The log shows garbage values from audio queries:
  ```
  getSampleRate: 0
  getChannelCount: 11
  getChannelCount: -495316880
  ```
- Other applications (browsers, video players, system sounds) all produce audio correctly

### 2.2 Root Cause Analysis

The audio system in mcpelauncher uses a three-layer architecture:

```
Minecraft (Android FMOD) → AAudio fake API → SDL3 Audio → PipeWire/PulseAudio
```

Three separate bugs in the AAudio fake layer combine to cause audio failure:

#### Bug A: Unvalidated `SDL_GetAudioDeviceFormat` Return Value

In `updateDefaults()`, the original code:

```cpp
SDL_AudioSpec spec;            // ← Uninitialized stack variable
int sampleFrames;              // ← Uninitialized
SDL_GetAudioDeviceFormat(..., &spec, &sampleFrames);  // ← Return value ignored
defaultSampleRate = spec.freq;       // ← Could be 0 or garbage
defaultNumChannels = spec.channels;  // ← Could be 11 or garbage
defaultBufSize = sampleFrames;       // ← Could be random value
```

When `SDL_GetAudioDeviceFormat()` fails (common with PipeWire on SDL3), `spec` and `sampleFrames` contain **uninitialized stack values**. These garbage values overwrite the sensible defaults (48000 Hz, 2 channels, 512 frames).

#### Bug B: Missing AAudio Builder Configuration Hooks

FMOD calls the following AAudio functions to configure the audio stream before opening it:

- `AAudioStreamBuilder_setSampleRate(builder, 48000)`
- `AAudioStreamBuilder_setChannelCount(builder, 2)`
- `AAudioStreamBuilder_setFormat(builder, AAUDIO_FORMAT_PCM_I16)`

These functions were **not hooked** in the fake AAudio layer. The calls silently resolved to no-ops, and the configured values were discarded. When the stream was opened, it used the (potentially garbage) default values instead of what FMOD requested.

#### Bug C: Incorrect Stream State for Newly Created Streams

`AAudioStream_getState()` returned `AAUDIO_STREAM_STATE_CLOSED` when `stream->s == NULL`. This occurs for streams that have been **created** (`openStream`) but **not yet started** (`requestStart`).

According to the AAudio API contract:
- After `openStream()`: state should be `AAUDIO_STREAM_STATE_OPEN`
- After `requestStart()`: state should be `AAUDIO_STREAM_STATE_STARTED`
- After `close()`: state should be `AAUDIO_STREAM_STATE_CLOSED`

FMOD checks the stream state after creating it. Receiving `CLOSED` immediately after creation causes FMOD to believe the stream has failed and abort initialization.

### 2.3 The AAudio Bridge Architecture

mcpelauncher translates Android's AAudio API calls to SDL3 audio:

```
┌──────────────────────────────────────────────────────────────────┐
│                    Minecraft (libminecraftpe.so)                  │
│                    Uses FMOD (Android build)                     │
│                    FMOD uses AAudio API                           │
└──────────────────┬───────────────────────────────────────────────┘
                   │ AAudio function calls
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                    FakeAudio (fake_audio.cpp)                     │
│                                                                  │
│  AAudio_createStreamBuilder()  → new FakeAudioStreamBuilder      │
│  AAudioStreamBuilder_setSampleRate(48000)  → builder.sampleRate   │
│  AAudioStreamBuilder_setChannelCount(2)    → builder.channelCount │
│  AAudioStreamBuilder_setFormat(PCM_I16)    → builder.format       │
│  AAudioStreamBuilder_openStream()   → new FakeAudioStream         │
│  AAudioStream_getState()            → OPEN (not started yet)      │
│  AAudioStream_requestStart()        → SDL_OpenAudioDeviceStream() │
│  AAudioStream_getState()            → STARTED                     │
│                                                                  │
│  Data flow (during playback):                                    │
│  SDL3 callback → dataCallback(stream, buffer, frames)            │
│               → SDL_PutAudioStreamData()                         │
└──────────────────┬───────────────────────────────────────────────┘
                   │ SDL3 Audio API
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│                    SDL3 Audio Layer                               │
│                    SDL_OpenAudioDeviceStream()                    │
│                    SDL_ResumeAudioDevice()                        │
└──────────────────┬───────────────────────────────────────────────┘
                   │ Native audio API
                   ▼
┌──────────────────────────────────────────────────────────────────┐
│              PipeWire / PulseAudio / ALSA                        │
│              → Hardware Audio Device                             │
└──────────────────────────────────────────────────────────────────┘
```

### 2.4 Implementation Details

**Files:** `src/fake_audio.cpp`, `src/fake_audio.h`

#### Change 1: Validate `SDL_GetAudioDeviceFormat` (`fake_audio.cpp`)

**Before:**
```cpp
void FakeAudio::updateDefaults() {
    SDL_AudioSpec spec;
    int sampleFrames;
    SDL_GetAudioDeviceFormat(SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK, &spec, &sampleFrames);
    defaultSampleRate = ReadEnvInt("AUDIO_SAMPLE_RATE", spec.freq);
    defaultNumChannels = spec.channels;
    defaultBufSize = sampleFrames;
    FmodUtils::setSampleRate(defaultSampleRate);
}
```

**After:**
```cpp
void FakeAudio::updateDefaults() {
    SDL_AudioSpec spec = {};         // Zero-initialized
    int sampleFrames = 0;           // Zero-initialized
    if (SDL_GetAudioDeviceFormat(SDL_AUDIO_DEVICE_DEFAULT_PLAYBACK, &spec, &sampleFrames)) {
        if (spec.freq > 0) defaultSampleRate = ReadEnvInt("AUDIO_SAMPLE_RATE", spec.freq);
        if (spec.channels > 0 && spec.channels <= 8) defaultNumChannels = spec.channels;
        if (sampleFrames > 0) defaultBufSize = sampleFrames;
    }
    // If SDL call fails: defaults remain at 48000 Hz, 2 ch, 512 frames
    FmodUtils::setSampleRate(defaultSampleRate);
}
```

#### Change 2: Add AAudio Builder Hooks (`fake_audio.cpp`)

Three new hooks that store FMOD's requested configuration:

```cpp
syms["AAudioStreamBuilder_setSampleRate"] = (void *)+[](FakeAudioStreamBuilder *builder, int32_t sampleRate) {
    builder->sampleRate = sampleRate;
};
syms["AAudioStreamBuilder_setChannelCount"] = (void *)+[](FakeAudioStreamBuilder *builder, int32_t channelCount) {
    builder->channelCount = channelCount;
};
syms["AAudioStreamBuilder_setFormat"] = (void *)+[](FakeAudioStreamBuilder *builder, aaudio_format_t format) {
    builder->format = format;
};
```

#### Change 3: Propagate Builder Values to Stream (`fake_audio.cpp`)

In `AAudioStreamBuilder_openStream`, after creating the `FakeAudioStream`:

```cpp
if (builder->sampleRate > 0) (*stream)->sampleRate = builder->sampleRate;
if (builder->channelCount > 0) (*stream)->channelCount = builder->channelCount;
if (builder->format != AAUDIO_FORMAT_UNSPECIFIED) (*stream)->format = builder->format;
```

#### Change 4: Fix Stream State (`fake_audio.cpp`)

```cpp
// Before: returned AAUDIO_STREAM_STATE_CLOSED when stream->s == NULL
// After:  returns AAUDIO_STREAM_STATE_OPEN (stream created but not started)
if (!stream->s) {
    return AAUDIO_STREAM_STATE_OPEN;
}
```

#### Change 5: Add Builder Fields (`fake_audio.h`)

Three new fields in `FakeAudioStreamBuilder`:

```cpp
int32_t sampleRate = 0;                          // 0 = use default
int32_t channelCount = 0;                        // 0 = use default
aaudio_format_t format = AAUDIO_FORMAT_UNSPECIFIED; // unspecified = use default
```

### 2.5 Audio Pipeline Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│ FMOD Initialization Sequence (with fix)                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ 1. AAudio_createStreamBuilder(&builder)                             │
│    └─ SDL_Init(SDL_INIT_AUDIO)                                      │
│    └─ updateDefaults():                                             │
│       └─ SDL_GetAudioDeviceFormat() → validate return ✓             │
│       └─ defaults: 48000 Hz, 2 ch, 512 frames (kept if SDL fails)  │
│                                                                     │
│ 2. AAudioStreamBuilder_setSampleRate(builder, 48000)     ◄── NEW    │
│    └─ builder->sampleRate = 48000                                   │
│                                                                     │
│ 3. AAudioStreamBuilder_setChannelCount(builder, 2)       ◄── NEW    │
│    └─ builder->channelCount = 2                                     │
│                                                                     │
│ 4. AAudioStreamBuilder_setFormat(builder, PCM_I16)       ◄── NEW    │
│    └─ builder->format = AAUDIO_FORMAT_PCM_I16                       │
│                                                                     │
│ 5. AAudioStreamBuilder_openStream(builder, &stream)                 │
│    └─ new FakeAudioStream{...}                                      │
│    └─ stream->sampleRate = builder->sampleRate (48000)   ◄── NEW    │
│    └─ stream->channelCount = builder->channelCount (2)   ◄── NEW    │
│    └─ stream->format = builder->format (PCM_I16)         ◄── NEW    │
│                                                                     │
│ 6. AAudioStream_getState(stream)                                    │
│    └─ stream->s == NULL → return AAUDIO_STREAM_STATE_OPEN ◄── FIX   │
│    └─ (before: returned CLOSED → FMOD aborted)                     │
│                                                                     │
│ 7. AAudioStream_requestStart(stream)                                │
│    └─ SDL_OpenAudioDeviceStream(spec: 48000Hz, 2ch, S16LE)         │
│    └─ SDL_ResumeAudioDevice()                                       │
│    └─ return AAUDIO_OK                                              │
│                                                                     │
│ 8. FMOD_System_Init → returns 0 (OK)  ✓                            │
│    (before: returned 28 = FMOD_ERR_OUTPUT_INIT)                     │
│                                                                     │
│ 9. Audio playback active via SDL3 → PipeWire → hardware             │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Files Changed

```
 src/fake_audio.cpp |  28 ++++++++++----
 src/fake_audio.h   |   3 ++
 src/fake_egl.cpp   | 105 +++++++++++++++++++++++++++++++++++++++++++++++++++++
 3 files changed, 128 insertions(+), 8 deletions(-)
```

| File | Lines Added | Lines Removed | Purpose |
|------|:-----------:|:-------------:|---------|
| `src/fake_egl.cpp` | +105 | -0 | NVIDIA shader binding remap, GL error logging |
| `src/fake_audio.cpp` | +20 | -8 | SDL validation, AAudio builder hooks, stream state fix |
| `src/fake_audio.h` | +3 | -0 | Builder struct fields for sample rate, channels, format |

---

## Building from Source

### Prerequisites

```bash
# Ubuntu/Kubuntu 24.04
sudo dpkg --add-architecture i386
sudo apt update
sudo apt install -y \
    git cmake clang libssl-dev libcurl4-openssl-dev \
    libpng-dev libx11-dev libxi-dev libxcursor-dev \
    libpulse-dev libuv1-dev libzip-dev libprotobuf-dev \
    protobuf-compiler qtbase5-dev qtwebengine5-dev \
    qtdeclarative5-dev libqt5svg5-dev qml-module-qtquick2 \
    qml-module-qtquick-layouts qml-module-qtquick-controls \
    qml-module-qtquick-controls2 qml-module-qtquick-window2 \
    qml-module-qtquick-dialogs qml-module-qt-labs-settings \
    qml-module-qt-labs-folderlistmodel
```

### Clone & Build

```bash
# Clone the ng manifest (this is the official PPA source)
git clone --recursive --depth 1 -b ng \
    https://github.com/minecraft-linux/mcpelauncher-manifest.git \
    ~/mcpelauncher-build

cd ~/mcpelauncher-build

# Apply patches (if not already applied)
cd mcpelauncher-client
git apply ~/mcpelauncher-build/patches/combined-nvidia-audio-fixes.patch
cd ..

# Configure
mkdir -p build && cd build
CC=clang CXX=clang++ cmake .. \
    -DCMAKE_BUILD_TYPE=Release \
    -DUSE_SDL3_AUDIO=ON

# Build
cmake --build . --target mcpelauncher-client -j$(nproc)

# Install
sudo cp mcpelauncher-client/mcpelauncher-client /usr/bin/mcpelauncher-client
```

### Verify Installation

```bash
file /usr/bin/mcpelauncher-client
# Expected: ELF 64-bit LSB pie executable, x86-64, ...
```

---

## Applying the Patches

### Individual Patches

```bash
cd mcpelauncher-client/

# NVIDIA fix only:
git apply 01-nvidia-gl-max-image-units-fix.patch

# Audio fix only:
git apply 02-audio-fmod-sdl3-init-fix.patch
```

### Combined Patch

```bash
cd mcpelauncher-client/
git apply combined-nvidia-audio-fixes.patch
```

### Reverting

```bash
cd mcpelauncher-client/
git checkout -- src/fake_egl.cpp src/fake_audio.cpp src/fake_audio.h
```

---

## Verification & Testing

### Launch Command

```bash
# Via Launcher UI (recommended — handles Google credentials automatically):
# Just click "Minecraft Bedrock Launcher" in your app menu or desktop

# Via CLI (for debugging):
mcpelauncher-client \
    -dg ~/.local/share/mcpelauncher/versions/1.26.10.4/ \
    -m ~/.local/share/mcpelauncher/mods/mcpelauncher-updates/26.10.4.2/x86_64/
```

### Expected Log Output

#### NVIDIA Fix Active
```
[FakeEGL] Remapped shader image binding >= 8 to val % 8 (NVIDIA fix, image-only)
[FakeEGL] Remapped glBindImageTexture unit 9 -> 1 (NVIDIA fix)
```

These messages appear once when the first affected shader is loaded (typically when entering a world).

#### Audio Fix Active
```
[FMOD] Failed to load host libfmod: '...', use pulseaudio/sdl3 backend with android fmod if available
Found hook: _ZN4FMOD6System4initEijPv @ 0x...
Found hook: _ZN4FMOD6System9setOutputE15FMOD_OUTPUTTYPE @ 0x...
```

No `FMOD_System_Init returned 28` error should appear. Audio should work in-game.

#### Error Indicators (Things That Should NOT Appear)
```
❌ FMOD_System_Init returned 28
❌ FMOD_ERR_OUTPUT_INIT
❌ Signal 6 in thread "Rendering Pool"
❌ Shader compile error: ...
❌ Program link error: ...
❌ getSampleRate: 0
❌ getChannelCount: 11
```

### Verification Checklist

- [ ] Game launches without crash
- [ ] Main menu renders with correct colors
- [ ] Entering a world does not crash (NVIDIA fix)
- [ ] Textures display with correct colors (not blue/red swapped)
- [ ] In-game sound effects are audible (Audio fix)
- [ ] Background music plays
- [ ] Game stable for extended play (5+ minutes)

---

## Troubleshooting

### Game still crashes with Signal 6

1. Check if the new binary is actually installed:
   ```bash
   ls -la /usr/bin/mcpelauncher-client
   # Check the modification date — should be recent
   ```

2. Check if the shader fix is logging:
   ```bash
   mcpelauncher-client -dg ... 2>&1 | grep -i "Remapped\|shader\|compile error"
   ```

3. If you see `Shader compile error: ...` in the log, the binding remap may need adjustment. Report the full shader error message.

### No audio despite fix

1. Verify system audio works:
   ```bash
   wpctl status
   speaker-test -t sine -f 440 -l 1
   ```

2. Check if SDL3 audio backend is being used (not host libfmod):
   ```bash
   mcpelauncher-client ... 2>&1 | grep -i "fmod\|audio\|sdl3"
   ```

3. Try forcing the sample rate via environment variable:
   ```bash
   AUDIO_SAMPLE_RATE=48000 mcpelauncher-client -dg ...
   ```

### Colors are wrong (blue/red swapped)

This indicates the shader binding remap is too broad. Verify you have the **latest version** of the patch that only remaps bindings on `image*` declaration lines. An earlier version remapped all bindings which caused this issue.

### Build fails

Ensure you have `clang` and `clang++` installed and are building against the `ng` branch:
```bash
clang --version   # Should be 18.x or newer
git -C mcpelauncher-client log --oneline -1  # Should show the fix commit
```

---

## Minecraft Version Update Guide (Step-by-Step)

> **This section documents the complete process for when Minecraft Bedrock gets a new version.**
> Follow these steps IN ORDER. Last verified: 2026-03-28 with version 1.26.10.4.

### Overview: What Needs Updating

When Mojang releases a new Minecraft Bedrock version, **three things** need to be checked/updated:

| # | Component | Location | What to check |
|---|-----------|----------|---------------|
| 1 | **DRM Mod** | `~/.local/share/mcpelauncher/mods/mcpelauncher-updates/` | Does a new DRM mod version exist that supports the new MC version? |
| 2 | **Patched Binary** | `/usr/bin/mcpelauncher-client` | Did APT overwrite our NVIDIA-patched binary? |
| 3 | **Profile Config** | `~/.local/share/mcpelauncher/profiles/profiles.ini` | Does the profile point to the correct DRM mod version? |

### Common Symptoms

| Symptom | Likely Cause | Fix |
|---------|-------------|-----|
| "Die Version ist veraltet" in-game | MC servers expect a newer version than you have | Update game via Launcher UI → download new version |
| SIGSEGV (Signal 11) on world load | DRM mod too old for this MC version | Update DRM mod (Step 2) |
| SIGABRT (Signal 6) on world load | APT overwrote the NVIDIA-patched binary | Rebuild & install patched binary (Step 3) |
| "Loaded 0 mods" in CLI launch | Missing `-m` flag when launching from CLI | Use `-m <mod-path>` flag (see CLI launch above) |
| Game launches but no multiplayer | DRM mod loaded wrong PlayFab library | Update DRM mod (Step 2) |

---

### Step 1: Download the New Minecraft Version

The Launcher UI handles this automatically:
1. Open **"Minecraft Bedrock Launcher"** (NOT the Flatpak "Minecraft"!)
2. The launcher will detect the new version and offer to download it
3. Click download/play — the new version APK gets extracted to `~/.local/share/mcpelauncher/versions/<version>/`

**Verify:**
```bash
ls ~/.local/share/mcpelauncher/versions/
# Should show the new version directory
```

---

### Step 2: Update the DRM Mod

#### 2a. Check if a new DRM mod is needed

```bash
curl -sL "https://github.com/minecraft-linux/mcpelauncher-moddb/raw/main/moddb.json" | python3 -c "
import json, sys
data = json.load(sys.stdin)
for mod in data:
    if mod.get('name') == 'mcpelauncher-updates':
        for v in mod.get('versions', []):
            archs = v.get('archs', {})
            x86 = archs.get('x86_64', {})
            extra = x86.get('extraVersions', [])
            x86_versions = [e for e in v.get('extraVersions', []) if 'x86_64' in e.get('codes', {})]
            version_names = [e['version_name'] for e in x86_versions]
            print(f'DRM Mod v{v[\"version\"]}: supports {version_names}')
"
```

Check if your new MC version is listed in the extraVersions. If NOT, you need to wait for the mod developers to release an update.

#### 2b. Download and install the new DRM mod

```bash
# Replace NEW_VERSION with the version from moddb.json (e.g., 26.10.4.2)
NEW_VERSION="26.10.4.2"

# Download
cd /tmp
wget "https://github.com/minecraft-linux/mcpelauncher-updates/releases/download/v${NEW_VERSION}/mcpelauncher-updates-x86_64-release.zip"

# Extract
mkdir -p ~/.local/share/mcpelauncher/mods/mcpelauncher-updates/${NEW_VERSION}/x86_64
cd ~/.local/share/mcpelauncher/mods/mcpelauncher-updates/${NEW_VERSION}/x86_64
unzip /tmp/mcpelauncher-updates-x86_64-release.zip
```

#### 2c. Create mod.json

The zip doesn't include a mod.json. Create one from the moddb.json data:

```bash
# Extract the mod.json from moddb.json for this version
curl -sL "https://github.com/minecraft-linux/mcpelauncher-moddb/raw/main/moddb.json" | python3 -c "
import json, sys
NEW_VERSION = '${NEW_VERSION}'
data = json.load(sys.stdin)
for mod in data:
    if mod.get('name') == 'mcpelauncher-updates':
        for v in mod.get('versions', []):
            if v['version'] == NEW_VERSION:
                mod_json = {
                    'lib': 'libmcpelauncher-updates.so',
                    'minecraft': v.get('minecraft', '>= 1.21.121'),
                    'extraVersions': []
                }
                for ev in v.get('extraVersions', []):
                    if 'x86_64' in ev.get('codes', {}):
                        mod_json['extraVersions'].append({
                            'versionName': ev['version_name'],
                            'arch': 'x86_64',
                            'versionCode': ev['codes']['x86_64']
                        })
                print(json.dumps(mod_json, indent=4))
" > ~/.local/share/mcpelauncher/mods/mcpelauncher-updates/${NEW_VERSION}/x86_64/mod.json
```

#### 2d. Update the profile

Edit `~/.local/share/mcpelauncher/profiles/profiles.ini`:
```ini
[Default]
mods\1\path=/home/sven/.local/share/mcpelauncher/mods/mcpelauncher-updates/NEW_VERSION/x86_64/
mods\size=1
version=googleplay
```

Or via sed:
```bash
OLD_MOD_VERSION="26.10.4.2"  # current version
NEW_MOD_VERSION="XX.XX.XX"   # new version from moddb.json
sed -i "s|${OLD_MOD_VERSION}|${NEW_MOD_VERSION}|g" ~/.local/share/mcpelauncher/profiles/profiles.ini
```

**Verify:**
```bash
grep "mods" ~/.local/share/mcpelauncher/profiles/profiles.ini
# Should show the new path
```

---

### Step 3: Verify/Rebuild the Patched Binary

APT updates can overwrite our NVIDIA-patched binary. Check first:

```bash
strings /usr/bin/mcpelauncher-client | grep "NVIDIA fix"
```

**If you see 2 lines** → Binary is fine, skip to Step 4.  
**If you see nothing** → APT overwrote it. Rebuild:

```bash
cd ~/mcpelauncher-build

# Update source
git pull --recurse-submodules

# Re-apply NVIDIA+audio fix
cd mcpelauncher-client
# Check if branch still exists:
git log --oneline nvidia-audio-fixes 2>/dev/null | head -3
# Cherry-pick the fix onto the new upstream:
git cherry-pick $(git log --oneline nvidia-audio-fixes | head -1 | awk '{print $1}')
cd ..

# Build
cd build
CC=clang CXX=clang++ cmake .. -DCMAKE_BUILD_TYPE=Release -DUSE_SDL3_AUDIO=ON
cmake --build . --target mcpelauncher-client -j$(nproc)

# Install
sudo cp mcpelauncher-client/mcpelauncher-client /usr/bin/mcpelauncher-client

# Verify
strings /usr/bin/mcpelauncher-client | grep "NVIDIA fix"
# Should show 2 lines
```

#### Re-protect against future APT overwrites

```bash
# Check if divert is still active:
dpkg-divert --list '*mcpelauncher-client*'

# If not set up:
sudo dpkg-divert --divert /usr/bin/mcpelauncher-client.dpkg-dist \
    --rename --add /usr/bin/mcpelauncher-client
# Then copy patched binary back:
sudo cp ~/mcpelauncher-build/build/mcpelauncher-client/mcpelauncher-client /usr/bin/mcpelauncher-client
```

---

### Step 4: Test

```bash
# Quick test from CLI (shows all log output):
mcpelauncher-client \
    -dg ~/.local/share/mcpelauncher/versions/<NEW_MC_VERSION>/ \
    -m ~/.local/share/mcpelauncher/mods/mcpelauncher-updates/<NEW_MOD_VERSION>/x86_64/ \
    2>&1 | tee /tmp/mc-test.log

# Check for:
grep "Loaded.*mod" /tmp/mc-test.log          # Should NOT be "Loaded 0 mods" for second line
grep "NVIDIA fix" /tmp/mc-test.log            # Should show remapping messages
grep "Game version" /tmp/mc-test.log          # Should show the new version
grep "Google credentials" /tmp/mc-test.log    # DRM mod is working
```

Then test from the Launcher UI (desktop icon "Minecraft Bedrock Launcher") for full functionality including Google credentials.

---

### Step 5: Troubleshooting Version Updates

#### SIGSEGV (Signal 11) with new version

This almost always means the DRM mod doesn't support this MC version yet:
```bash
# Check moddb.json for updates:
curl -sL "https://github.com/minecraft-linux/mcpelauncher-moddb/raw/main/moddb.json" | python3 -m json.tool | grep -A2 "version_name"

# Check GitHub issues:
# https://github.com/minecraft-linux/mcpelauncher-manifest/issues
# Look for "Implement version X.XX.XX.X" issues
```

If no DRM mod is available yet: **downgrade to the last working MC version** by changing `version=` in profiles.ini to a specific version (e.g., `version=1.26.10.4`).

#### "Die Version ist veraltet" but correct version installed

1. Make sure you're using **"Minecraft Bedrock Launcher"** (mcpelauncher-ui-qt), NOT the Flatpak "Minecraft" (com.mojang.Minecraft)
2. Check installed versions: `ls ~/.local/share/mcpelauncher/versions/`
3. Check profile: `cat ~/.local/share/mcpelauncher/profiles/profiles.ini` — `version=googleplay` means auto-latest
4. The Launcher UI should detect and offer the newest version for download

#### Cherry-pick fails during rebuild

If `git cherry-pick` has conflicts:
```bash
# Check what the original fix commits look like:
git log --oneline nvidia-audio-fixes
# Manually apply from patch files:
git apply ~/mcpelauncher-build/patches/combined-nvidia-audio-fixes.patch
# Or apply individually:
git apply ~/mcpelauncher-build/patches/01-nvidia-gl-max-image-units-fix.patch
git apply ~/mcpelauncher-build/patches/02-audio-fmod-sdl3-init-fix.patch
```

---

### Quick Reference: Important Paths

| Item | Path |
|------|------|
| Patched binary | `/usr/bin/mcpelauncher-client` |
| APT backup binary | `/usr/bin/mcpelauncher-client.dpkg-dist` |
| Build directory | `~/mcpelauncher-build/` |
| Build output | `~/mcpelauncher-build/build/mcpelauncher-client/mcpelauncher-client` |
| Patch files | `~/mcpelauncher-build/patches/` |
| Game versions | `~/.local/share/mcpelauncher/versions/` |
| DRM mods | `~/.local/share/mcpelauncher/mods/mcpelauncher-updates/` |
| Profile config | `~/.local/share/mcpelauncher/profiles/profiles.ini` |
| Launcher UI config | `~/.config/Minecraft Linux Launcher/Minecraft Linux Launcher UI.conf` |
| Desktop entry (local) | `~/.local/share/applications/mcpelauncher-ui-qt.desktop` |
| Desktop entry (system) | `/usr/share/applications/mcpelauncher-ui-qt.desktop` |
| Desktop shortcut | `~/Schreibtisch/Minecraft Bedrock.desktop` |

### Quick Reference: Important URLs

| Resource | URL |
|----------|-----|
| DRM Mod Database | `https://github.com/minecraft-linux/mcpelauncher-moddb/raw/main/moddb.json` |
| DRM Mod Releases | `https://github.com/minecraft-linux/mcpelauncher-updates/releases` |
| Version Database | `https://raw.githubusercontent.com/minecraft-linux/mcpelauncher-versiondb/master/versions.json` |
| Launcher Issues | `https://github.com/minecraft-linux/mcpelauncher-manifest/issues` |
| Our Patches PR | `https://github.com/minecraft-linux/mcpelauncher-client/pull/133` |

### Quick Reference: Important Commands

```bash
# Check patched binary is in place:
strings /usr/bin/mcpelauncher-client | grep "NVIDIA fix"

# Check dpkg-divert protection:
dpkg-divert --list '*mcpelauncher*'

# Check current profile:
cat ~/.local/share/mcpelauncher/profiles/profiles.ini

# Check installed game versions:
ls ~/.local/share/mcpelauncher/versions/

# Check installed DRM mod versions:
ls ~/.local/share/mcpelauncher/mods/mcpelauncher-updates/

# Check latest available DRM mod:
curl -sL "https://github.com/minecraft-linux/mcpelauncher-moddb/raw/main/moddb.json" | python3 -m json.tool | grep '"version"'

# Check latest available MC version:
curl -sL "https://raw.githubusercontent.com/minecraft-linux/mcpelauncher-versiondb/master/versions.json" | python3 -c "
import json,sys
non_beta = [e for e in json.load(sys.stdin) if not e.get('beta')]
e = non_beta[-1]
print(f'Latest: {e[\"version_name\"]} (x86_64: {e[\"codes\"].get(\"x86_64\",\"?\")})')
"
```

---

### Version History

| Date | MC Version | DRM Mod | Notes |
|------|-----------|---------|-------|
| 2026-03-15 | 1.26.0.2 | 26.0.2 | Initial NVIDIA+Audio fix, binary built & patched |
| 2026-03-26 | 1.26.3.1 | 26.0.2 | APT overwrote binary (package v1.7.1.23570272227), rebuilt |
| 2026-03-28 | 1.26.10.4 | 26.10.4.2 | DRM mod updated (old 26.0.2 caused SIGSEGV), binary rebuilt from upstream 531a244 + cherry-pick 9a3f27e |

---

### WARNING: Two Different Launchers!

There are potentially **two Minecraft launchers** on this system:

| Launcher | Type | Desktop Name | Works? |
|----------|------|-------------|--------|
| `mcpelauncher-ui-qt` | APT package | **"Minecraft Bedrock Launcher"** | ✅ Yes — with NVIDIA fix + DRM mod |
| `com.mojang.Minecraft` | Flatpak | **"Minecraft"** | ❌ Different program entirely (Java Edition launcher) |

**Always use "Minecraft Bedrock Launcher"**, not the Flatpak "Minecraft"!

The Flatpak can be removed if not needed:
```bash
sudo flatpak uninstall com.mojang.Minecraft
```

---

## Technical Background

### How mcpelauncher Works

mcpelauncher runs the Android version of Minecraft Bedrock Edition (`libminecraftpe.so`) natively on Linux by:

1. **Linker** (`mcpelauncher-linker`): Loads the Android ELF shared libraries into the Linux process
2. **libc-shim**: Translates Android libc (Bionic) calls to Linux glibc
3. **JNI VM** (`libjnivm`): Provides a fake Java Native Interface for the game's JNI calls
4. **FakeEGL** (`fake_egl.cpp`): Intercepts OpenGL ES calls and forwards them to the host's desktop OpenGL
5. **FakeAudio** (`fake_audio.cpp`): Intercepts Android AAudio API calls and bridges them to SDL3 audio
6. **DRM Mod** (`mcpelauncher-updates`): Patches the game's DRM verification for legitimate offline use

### OpenGL ES vs Desktop OpenGL

Minecraft Bedrock is compiled for **OpenGL ES 3.0+** (Android). Desktop Linux typically provides **OpenGL 4.6** (desktop profile). The mcpelauncher bridges these by:

- Rewriting GLSL `#version 300 es` to `#version 410` (in `gl_core_patch.cpp`)
- Stubbing incompatible functions (e.g., `glInvalidateFramebuffer`)
- Disabling problematic extensions (e.g., `glDrawElementsInstancedOES` for Mesa)
- **This patch adds:** Remapping image bindings that exceed desktop GPU limits

### AAudio on Android vs Linux

On real Android, AAudio is a native C API that directly accesses the audio HAL. On mcpelauncher, it's implemented as a shim:

- `AAudio_createStreamBuilder` → allocates a `FakeAudioStreamBuilder` struct
- `AAudioStreamBuilder_set*` → stores configuration in the builder
- `AAudioStreamBuilder_openStream` → creates a `FakeAudioStream` with the configuration
- `AAudioStream_requestStart` → calls `SDL_OpenAudioDeviceStream()` and starts playback
- The data callback receives PCM frames from FMOD and feeds them to SDL3

---

## Development

This patch set was developed using **AI-assisted programming** ("vibecoding") with **Claude Opus 4.6** (Anthropic) via GitHub Copilot in VS Code. The AI performed root cause analysis, implemented the fixes, and authored the documentation. All changes were practically tested and verified in-game by [brainAThome](https://github.com/brainAThome).

---

## License

These patches are distributed under the same license as [mcpelauncher-client](https://github.com/minecraft-linux/mcpelauncher-client) (GPLv3).
