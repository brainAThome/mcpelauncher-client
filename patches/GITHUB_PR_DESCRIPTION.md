# PR Title: Fix NVIDIA crash (GL_MAX_IMAGE_UNITS=8) and FMOD audio init failure on Linux

## Summary

Two fixes for the `ng` branch that resolve:
1. **SIGABRT crash on NVIDIA GPUs** where `GL_MAX_IMAGE_UNITS = 8`
2. **No audio output** (FMOD_ERR_OUTPUT_INIT / error code 28) when SDL3 audio backend is used with PipeWire

Tested on: Kubuntu 24.04, NVIDIA RTX 4080 (driver 590.48), PipeWire 1.0.5, Minecraft Bedrock 1.26.0.2.

---

## Fix 1: NVIDIA GL_MAX_IMAGE_UNITS crash (`fake_egl.cpp`)

### Problem

Minecraft Bedrock uses compute shaders with `layout(binding = 9)` on `image2DArray` uniforms. NVIDIA desktop GPUs report `GL_MAX_IMAGE_UNITS = 8` (units 0-7), so binding index 9 causes a shader compilation failure → `std::terminate()` → SIGABRT in the "Rendering Pool" thread.

```
Signal 6 in thread "Rendering Pool" at [libminecraftpe.so + 0x5dc9898]
```

### Solution

- Hook `glShaderSource` to remap `binding = N` (where N >= 8) to `N % 8`, but **only on lines containing `image` uniforms** (image2D, image2DArray, etc.). Sampler and UBO bindings are not affected (they have much higher limits).
- Hook `glBindImageTexture` to remap `unit >= 8` to `unit % 8` correspondingly.
- Add `glCompileShader` and `glLinkProgram` error logging hooks for easier debugging.

### Why image-only?

`GL_MAX_IMAGE_UNITS` (typically 8 on NVIDIA) only applies to `image*` type uniforms. Sampler bindings (`GL_MAX_COMBINED_TEXTURE_IMAGE_UNITS`, typically 32+) and UBO bindings (`GL_MAX_UNIFORM_BUFFER_BINDINGS`, typically 84+) must NOT be remapped — doing so causes wrong textures to be sampled, resulting in incorrect colors.

---

## Fix 2: FMOD/AAudio audio initialization failure (`fake_audio.cpp`, `fake_audio.h`)

### Problem

When using the SDL3 audio backend (no host libfmod), FMOD fails to initialize with error code 28 (`FMOD_ERR_OUTPUT_INIT`). Root causes:

1. **`updateDefaults()` uses unvalidated return from `SDL_GetAudioDeviceFormat()`**: When this call fails (common with PipeWire), the `SDL_AudioSpec` struct contains garbage values, overwriting the sensible defaults (48000 Hz, 2 channels, 512 buffer) with invalid values like `freq=0`, `channels=11`.

2. **Missing AAudio builder hooks**: FMOD calls `AAudioStreamBuilder_setSampleRate`, `AAudioStreamBuilder_setChannelCount`, and `AAudioStreamBuilder_setFormat` to configure the stream, but these were not hooked — the values were silently discarded.

3. **Incorrect stream state**: `AAudioStream_getState` returned `AAUDIO_STREAM_STATE_CLOSED` for streams that were created but not yet started (`stream->s == NULL`). FMOD interprets this as a dead stream and aborts initialization. The correct state is `AAUDIO_STREAM_STATE_OPEN`.

### Solution

- Validate `SDL_GetAudioDeviceFormat` return value and sanitize `spec` fields before updating defaults
- Add hooks for `AAudioStreamBuilder_setSampleRate`, `setChannelCount`, `setFormat` → store on builder struct
- Add `sampleRate`, `channelCount`, `format` fields to `FakeAudioStreamBuilder` and propagate them to the stream in `openStream`
- Return `AAUDIO_STREAM_STATE_OPEN` instead of `CLOSED` when stream is created but not started

---

## Files Changed

| File | Changes |
|------|---------|
| `src/fake_egl.cpp` | Added `#include <regex>`, NVIDIA image binding remap hooks, shader/program error logging |
| `src/fake_audio.cpp` | Validated SDL defaults, added 3 AAudio builder hooks, propagate builder config to stream, fixed stream state |
| `src/fake_audio.h` | Added `sampleRate`, `channelCount`, `format` fields to `FakeAudioStreamBuilder` |

## How to Apply

```bash
cd mcpelauncher-client
git apply combined-nvidia-audio-fixes.patch
```

## Testing

- NVIDIA RTX 4080, driver 590.48.01, GL_MAX_IMAGE_UNITS=8 → no crash, game renders correctly
- PipeWire 1.0.5 audio → FMOD initializes successfully, in-game sound works
- Stable gameplay for extended sessions
