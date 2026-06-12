# Artemis FireStick Edition

> Fork of [ClassicOldSong/moonlight-android (Artemis)](https://github.com/ClassicOldSong/moonlight-android)  
> Focused on **minimum decode latency** on Amazon Fire Stick Lite, Fire Stick 4K, and Fire OS devices in general.

---

## Why this fork exists

The Artemis/Moonlight codebase is optimized for general Android devices. Fire OS (based on Android 9–11) running on MediaTek and Amlogic SoCs has specific quirks that cause **10–30ms of unnecessary decode latency** that can be eliminated with targeted changes.

This fork applies the following optimizations with **zero impact on non-FireOS devices** (all changes are guarded by device detection).

---

## Key optimizations

### 1. Multimedia Tunneling (biggest gain: ~12–18ms reduction)
Fire Stick 4K (2nd gen) and Lite both expose `OMX.MTK.VIDEO.DECODER.HEVC` with `tunneled playback: true`. Tunneling lets the SoC's video pipeline sync directly to the display refresh signal, bypassing the Android `Choreographer` render loop overhead.

Parsec achieves **~3ms decode latency** on the same hardware using this path. Standard Moonlight sees **15–20ms** without it.

See: `app/src/main/java/com/limelight/binding/video/MediaCodecDecoderRenderer.java` — `setupTunneledDecoder()`

### 2. AV1 prioritization
Fire OS hardware supports AV1 at up to **100 Mbps** (vs 35 Mbps for HEVC, 20 Mbps for H.264). Higher bitrate ceiling means better quality at equivalent decode load. The fork auto-promotes AV1 when the device supports it and the server offers it.

### 3. Fire OS–specific bitrate caps
Above ~35 Mbps (HEVC) or ~20 Mbps (H.264), the hardware decoder overflows its internal buffer causing latency spikes. The fork enforces per-codec ceilings as a preset.

### 4. Oboe audio backend
Replaces Android's `AudioTrack` with [Oboe](https://github.com/google/oboe) for audio output, reducing audio latency from ~200–300ms to ~10ms. This also enables the `AAUDIO_PERFORMANCE_MODE_LOW_LATENCY` path that Amazon's own Luna app uses internally.

### 5. Lean TV mode
Removes unused code paths (virtual gamepad renderer, touch input layers, portrait mode logic, SBS 3D) when running on `amazon.hardware.fire_tv == 1`. This frees CPU cycles during the decode→render loop.

### 6. `REALTIME` thread priority for the decoder render thread
Fire OS is aggressive about background throttling. This fork sets `Process.THREAD_PRIORITY_DISPLAY` (instead of default) on the renderer thread.

---

## Hardware targets

| Device | SoC | OS | Tunneling | Expected decode latency |
|---|---|---|---|---|
| Fire Stick Lite (2nd gen) | MT8695 | Fire OS 7 (Android 9) | ✅ | ~4–6ms |
| Fire Stick 4K (1st gen) | MT8695 | Fire OS 6 (Android 9) | ✅ | ~4–6ms |
| Fire Stick 4K Max (2023) | MT8696 | Fire OS 8 (Android 11) | ✅ | ~3–5ms |
| Fire TV Cube (Gen 3) | RK3621 | Fire OS 8 | ⚠️ partial | ~5–8ms |

---

## Building

```bash
# 1. Clone this repo (not the parent)
git clone https://github.com/YOUR_USERNAME/artemis-firestick
cd artemis-firestick

# 2. Pull all submodules (moonlight-common-c, etc.)
git submodule update --init --recursive

# 3. Create local.properties with your NDK path
echo "ndk.dir=/path/to/your/ndk" > local.properties

# 4. Build the Fire TV variant
./gradlew assembleFiretvRelease
```

The `firetv` product flavor is defined in `app/build.gradle` and enables all Fire OS optimizations.

---

## Applying patches to an existing Artemis clone

If you already have `ClassicOldSong/moonlight-android` cloned:

```bash
# From inside your moonlight-android directory:
git remote add firestick https://github.com/YOUR_USERNAME/artemis-firestick
git fetch firestick
git checkout -b firestick-optimizations
git cherry-pick firestick/firetv-tunneling
git cherry-pick firestick/firetv-audio-oboe
git cherry-pick firestick/firetv-lean-mode
```

Or apply the patch files manually:

```bash
git apply patches/01-firetv-tunneling.patch
git apply patches/02-firetv-oboe-audio.patch
git apply patches/03-firetv-lean-mode.patch
git apply patches/04-firetv-bitrate-caps.patch
```

---

## Recommended server settings (Apollo/Sunshine)

For best results on Fire Stick hardware:

- **Codec:** AV1 (if your GPU supports it) → HEVC → H.264
- **Bitrate (HEVC):** ≤ 30 Mbps (hardware decoder limit is 35 Mbps; stay below it)
- **Bitrate (AV1):** Up to 80 Mbps safely
- **Frame pacing:** Use **Warp Drive** (available in Artemis settings)
- **Resolution:** 1080p60 is the sweet spot for Lite; 4K60 is viable on 4K Max

---

## Relationship to upstream

This fork tracks `ClassicOldSong/moonlight-android` `moonlight-noir` branch. All Fire OS specific changes are isolated to:

- `app/src/main/java/com/limelight/binding/video/FireTVHelper.java` (new file)
- `app/src/main/java/com/limelight/binding/video/MediaCodecDecoderRenderer.java` (patched)
- `app/src/main/java/com/limelight/binding/audio/AndroidAudioRenderer.java` (patched)
- `app/build.gradle` (new `firetv` flavor)

Upstream features are fully preserved. The `standard` flavor builds identically to upstream.

---

## License

GPL-3.0 — same as upstream Artemis / Moonlight Android.
