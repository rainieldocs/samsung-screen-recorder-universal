# Samsung Screen Recorder – Universal

[Version](https://img.shields.io/badge/version-99.9.99-blue)

[Android](https://img.shields.io/badge/Android-7.0%2B-green)

A **modified Samsung Screen Recorder** that works on **all Samsung devices (Android 7.0+)** — **no root, no system signature required**.

## ✨ Features

- Works on **all Samsung devices**
- **No root required**
- Screen recording with audio
- Audio options: **Mute / System / System + Mic**
- Video quality: **Low / Medium / High**
- Optional **touch indicators**
- **Hide floating button** (clean recordings)
- Uses **official MediaProjection API**
- Stable **foreground service**

## 📱 Requirements

- Android 7.0+
- Samsung phone or tablet
- Screen, audio, and storage permissions

## 📥 Installation

**ADB**

```bash
adb install screenrecorder_customized-aligned-debugSigned.apk

```

**Manual**

1. Download the APK
2. Enable *Install unknown apps*
3. Install the APK

## 🚨 IMPORTANT (Required)

**Press and hold the Screen Recorder tile in Quick Settings.**

When it opens:

1. Open **Recent apps**
2. Tap the **Recorder app icon**
3. **Allow all permissions**

> Skipping this step will cause the app to fail.
> 

**Also allow "Appear on top"** (Settings › Apps › Record screen › Appear on top), or run:

```bash
adb shell appops set com.sec.app.screenrecorder SYSTEM_ALERT_WINDOW allow
```

The floating button and the "Saving…" dialog need it.

**Optional: skip the "Start recording?" prompt.** With patch 0003 applied (see "One UI 8 / Android 16 fixes" below), Android asks for screen-capture consent before each recording. To allow screen capture for this app without a prompt:

```bash
adb shell appops set com.sec.app.screenrecorder PROJECT_MEDIA allow
```

This lets the app capture the screen at any time without asking you. To undo it:

```bash
adb shell appops set com.sec.app.screenrecorder PROJECT_MEDIA default
```

## 🛠️ One UI 8 / Android 16 fixes

Three patches in [`patches/`](patches/) fix problems seen on One UI 8.5 (Android 16). Each one changes the smali output of `apktool d`.

| Patch | Problem | Fix |
|---|---|---|
| [`0001`](patches/0001-mediastore-drop-samsung-only-columns.patch) | `MediaUtil.insertToDB` fails with `IllegalArgumentException: Invalid column video_codec_info`. The video is saved but not added to MediaStore, so Gallery shows it only after a later background scan, and the "done" notification cannot open it. | Remove the three Samsung-only columns (`recordingtype`, `video_codec_info`, `audio_codec_info`). All other columns are standard MediaStore columns and are kept. |
| [`0002`](patches/0002-loading-dialog-use-application-overlay.patch) | The app crashes when you stop a recording: `BadTokenException: permission denied for window type 2003`. `TYPE_SYSTEM_ALERT` (2003) is not allowed for apps on Android 8+, even with "Appear on top". The preview and the "done" notification never appear. | Show the "Saving…" dialog as `TYPE_APPLICATION_OVERLAY` (2038), the same type the floating button uses. |
| [`0003`](patches/0003-release-media-projection-after-recording.patch) | The app never calls `MediaProjection.stop()`, so the red screen-sharing icon stays in the status bar after a recording ends. | Release the projection and forget the consent result in `RecordingThreadService.onDestroy()`. Android asks for consent again before the next recording, as [Android 14+ requires](https://developer.android.com/about/versions/14/behavior-changes-14). Use the `PROJECT_MEDIA` command above to skip the prompt. |

Apply them to the APK:

```bash
apktool d screenrecorder_customized-aligned-debugSigned.apk -o sr
for p in patches/*.patch; do patch -d sr -p1 < "$p"; done
apktool b sr -o sr-unsigned.apk
zipalign -f -p 4 sr-unsigned.apk sr-aligned.apk
apksigner sign --ks ~/.android/debug.keystore --ks-pass pass:android --out sr-patched.apk sr-aligned.apk
```

A rebuilt APK has a different signature, so uninstall the old one first (`adb uninstall com.sec.app.screenrecorder`).

To find and fix the next problem the same way, see [docs/modifying.md](docs/modifying.md).

Tested on Galaxy XCover7 Pro (SM-G556B), One UI 8.5, Android 16: recording, Gallery, "done" notification, preview share, and no prompt with `PROJECT_MEDIA` allowed.

## ▶️ How to Use

1. Open **Screen Recorder**
2. Grant permissions
3. Start recording (3-second countdown)
4. Stop via floating button or notification

## 📁 Output Location

```
/storage/emulated/0/DCIM/Video screenshots/

```

## ⚠️ Limitations

- DRM apps (Netflix, etc.) cannot be recorded
- Persistent notification during recording (Android requirement)
- Internal audio may be limited on some apps

## ℹ️ Info

- **Package**: `com.sec.app.screenrecorder`
- **Version**: 99.9.99
- **Min SDK**: 24 (Android 7.0)
- **Target SDK**: 28

## ⚠️ Disclaimer

Unofficial modification of Samsung Screen Recorder.

Not affiliated with Samsung. Use at your own risk.

**Original app**: Samsung Screen Recorder v4.0.23

**Modified by**: [rrrainielll](https://github.com/rrrainielll)
