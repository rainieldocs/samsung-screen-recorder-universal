# How to modify the APK

This repo has no source code, only the APK. This page shows how the patches in [`patches/`](../patches/) were made, so you can fix the next problem the same way.

The loop is: reproduce the bug, read the log, find the code, change the smali, rebuild, install, test.

## Tools

```bash
brew install apktool jadx          # decode/rebuild, and readable Java for reading
# Android SDK build-tools: zipalign, apksigner, aapt2
# adb from Android platform-tools
```

| Tool | Use |
|---|---|
| `jadx` | Decompile to Java. Read only: the Java output does not rebuild. |
| `apktool` | Decode to smali, then rebuild. All edits are made here. |
| `adb logcat` | Find the exception and the code path. |
| `zipalign`, `apksigner` | Align and sign the rebuilt APK. |

## 1. Inspect the APK first

```bash
shasum -a 256 screenrecorder_customized-aligned-debugSigned.apk
apksigner verify --print-certs screenrecorder_customized-aligned-debugSigned.apk
aapt2 dump badging screenrecorder_customized-aligned-debugSigned.apk | grep -E "package|targetSdk|uses-permission"
```

The APK has no `INTERNET` permission, no native libraries and one `classes.dex`.

## 2. Find the bug in logcat

The app logs with the tag `JDog->`. Its crash handler also logs through that tag, so a crash looks like `E/JDog-> ... [CrashHandler]`.

```bash
adb logcat -c                                   # clear, then reproduce the bug
adb logcat -d -v time | grep -E "JDog|AndroidRuntime|MediaProvider"
```

What each patch came from:

| Patch | Log line |
|---|---|
| 0001 | `E/JDog-> [RecordingThreadService] java.lang.IllegalArgumentException: Invalid column video_codec_info` |
| 0002 | `BadTokenException: Unable to add window ... permission denied for window type 2003` at `RecordingThreadService.createLoadingDialog` |
| 0003 | No error. `adb shell dumpsys media_projection` still listed `com.sec.app.screenrecorder` after recording stopped. |

## 3. Read the code with jadx

```bash
jadx -d jx screenrecorder_customized-aligned-debugSigned.apk
grep -rn "insertToDB\|releaseMediaProjection\|setType(" jx/sources/com/sec/app/screenrecorder
```

Read the Java to understand the logic. Then find the same method in the smali, which is what you edit.

## 4. Check your assumption on the device

Prove the cause before you edit. Examples from these patches:

- **0001:** ask MediaProvider whether each column exists. A missing column returns `Error while accessing provider`.

  ```bash
  for c in recordingtype resolution video_codec_info audio_codec_info width height duration; do
    printf "%-18s " $c
    adb shell "content query --uri content://media/external/video/media --projection $c" 2>&1 | head -1
  done
  ```

- **0002:** `grep -rn "0x7d3\|0x7f6" smali/` showed that the dialog used `TYPE_SYSTEM_ALERT` (`0x7d3`) and the working floating button used `TYPE_APPLICATION_OVERLAY` (`0x7f6`).
- **0003:** `grep -rn releaseMediaProjection jx/` found the method but no caller.

## 5. Edit the smali

```bash
apktool d screenrecorder_customized-aligned-debugSigned.apk -o sr
cp sr/smali/com/sec/app/screenrecorder/util/MediaUtil.smali MediaUtil.orig.smali   # keep a copy for the diff
```

Rules for smali edits:

- Change as little as possible: delete calls, or change a constant.
- When you add code, use only registers that the method already declares in `.locals`, and overwrite them only where their old value is no longer used. Patch 0003 uses `v0` at the start of `onDestroy`, before the original code sets it.
- Keep the labels (`:cond_0`). An `if-eqz` that jumps to the next line is valid.

## 6. Rebuild, sign, install

```bash
apktool b sr -o sr-unsigned.apk
zipalign -f -p 4 sr-unsigned.apk sr-aligned.apk
apksigner sign --ks ~/.android/debug.keystore --ks-pass pass:android --out sr-patched.apk sr-aligned.apk
adb uninstall com.sec.app.screenrecorder        # first time only: the signature changed
adb install -r sr-patched.apk
```

Sign every later build with the same keystore. Then `adb install -r` updates the app and keeps its permissions and Quick Settings tile.

After an uninstall, grant the permissions again:

```bash
P=com.sec.app.screenrecorder
for p in RECORD_AUDIO READ_EXTERNAL_STORAGE WRITE_EXTERNAL_STORAGE POST_NOTIFICATIONS; do
  adb shell pm grant $P android.permission.$p
done
adb shell appops set $P SYSTEM_ALERT_WINDOW allow
```

## 7. Test and write the patch

Test on the device with `adb logcat` running. Check that the original log line is gone. Then write a patch for each fix, with paths relative to the `apktool d` output:

```bash
F=smali/com/sec/app/screenrecorder/util/MediaUtil.smali
diff -u --label a/$F --label b/$F MediaUtil.orig.smali sr/$F > patches/0004-short-name.patch
```

Before you open a pull request, check that the patches rebuild exactly what you tested:

```bash
apktool d -f screenrecorder_customized-aligned-debugSigned.apk -o check
for p in patches/*.patch; do patch -d check -p1 < "$p"; done
diff -rq check/smali sr/smali && echo "patches match the tested build"
```
