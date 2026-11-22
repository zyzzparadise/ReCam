# MediaTek NV21 / live-frame experimental build

This branch is an experimental compatibility patch for the Redmi Note 14 4G
(tanzanite) on Android 16 / HyperOS 3. It is **not** verified on the handset.

## Changes

- Treat camera output format `0x11` (NV21 / YCrCb_420_SP) as a candidate for
  substitution. A write is allowed **only** when gralloc describes a true
  semi-planar VU layout: 1-byte luma samples, 2-byte chroma samples, identical
  chroma row strides, Cr followed immediately by Cb, and chroma located after
  the complete luma region. If either lock API or these guards fail, the
  original camera buffer is passed through unchanged.
- Retry the Android graphic-buffer mapper when AHardwareBuffer_lockPlanes
  supplies a layout that fails the NV21 safety checks.
- Do **not** inject the static payload colour bars when a frame is absent or
  stale. Previously these bars could mask what the live feeder was doing.
  The Android feeder's animated test pattern remains available.
- Add a **read-only** `center_i420` diagnostic to each served lane in
  `recam status`: the Y, Cb, and Cr values of the latest shared-memory frame.
  This helps distinguish actual red and blue inputs from fallback video.

## Build

The `.github/workflows/mediatek-test-build.yml` workflow builds Android
ARM64 binaries and a complete Windows x64 ReCam desktop package on GitHub
Actions. Look for its **Android ARM64 test** and **Windows x64 ReCam test**
artifacts. Do not mix old and new Android binaries.

## Testing carefully

1. Stop ReCam from the old desktop app, and **leave the device booted**.
2. Extract the new Windows build into a fresh folder and launch its
   `ReCam.exe` with ADB and FFmpeg available on PATH.
3. Pick the FFmpeg input:
   `-re -f lavfi -i color=c=red:s=640x480:r=15`.
   Start ReCam, then open the target camera app.
4. While that camera is open, run:
   `adb shell "su -c '/data/local/tmp/recam status'"`
   and check `center_i420`, plus video preview and recordings separately.
5. Repeat with `color=c=blue:s=640x480:r=15`.
   For standard limited-range YUV, red normally has a much higher Cr value
   than blue; blue has a much higher Cb value than red. The exact numeric
   values depend on conversion settings.
6. Run `adb logcat -d -v time -s ReCam:*` to see substitutions and
   any `pass ... invalid NV21 plane layout` or `lock failed` results.

If the camera crashes, stop ReCam. This patch makes no changes to firmware,
boot images or SELinux settings, but all `cameraserver` injection remains
experimental and can crash the camera process.

The shared ring structure/version are unchanged; `center_i420` is an
optional, read-only JSON field in the CLI status command.
