# Crop2Bounds — Android APK Build Guide

> Autocrop images to content bounds · No background removal · Share-to-app support

---

## What you get

| Feature | Status |
|---|---|
| Crop to transparency bounds | ✅ |
| Crop to color bounds (color trim) | ✅ |
| Padding control | ✅ |
| Output format (PNG / WebP / JPG / match) | ✅ |
| **Share image TO app** (from Gallery, Files, etc.) | ✅ |
| **Share result OUT** (Android native share sheet) | ✅ |
| Download / ZIP | ✅ |
| AI background removal | ✅ optional, on-device |

---

## Prerequisites

Install these **once** on your machine:

1. **Node.js 18+** — https://nodejs.org
2. **Android Studio** (latest stable) — https://developer.android.com/studio
   - During install, make sure **Android SDK**, **Android SDK Platform**, and **Android Virtual Device** are checked
3. After Android Studio is installed, set the env variable:
   ```bash
   export ANDROID_SDK_ROOT=$HOME/Library/Android/sdk   # macOS
   export ANDROID_SDK_ROOT=%LOCALAPPDATA%\Android\Sdk  # Windows
   ```

---

## Build steps

### Step 1 — Install dependencies

```bash
cd crop2bounds-android/capacitor-project
npm install
```

### Step 2 — Add Android platform

```bash
npx cap add android
```

This creates the `android/` folder with a full Gradle project.

### Step 3 — Patch AndroidManifest.xml

After `cap add android`, **replace** the generated manifest with the one provided:

```bash
cp ../app/src/main/AndroidManifest.xml android/app/src/main/AndroidManifest.xml
```

This adds the critical **Share intent filter** that makes Crop2Bounds appear in
Android's share sheet when a user shares an image from their Gallery.

### Step 4 — Patch MainActivity.java

```bash
cp ../app/src/main/java/com/crop2bounds/MainActivity.java \
   android/app/src/main/java/com/crop2bounds/app/MainActivity.java
```

This handles the incoming share intent and forwards the content URI to the
web layer via Capacitor's event bridge.

### Step 5 — Copy FileProvider config

```bash
mkdir -p android/app/src/main/res/xml
cp ../app/src/main/res/xml/file_paths.xml \
   android/app/src/main/res/xml/file_paths.xml
```

### Step 6 — Sync web assets

```bash
npx cap sync android
```

### Step 7 — Build the debug APK

```bash
cd android
./gradlew assembleDebug
```

The APK will be at:
```
android/app/build/outputs/apk/debug/app-debug.apk
```

---

## Install on your Android device

### Option A — ADB (USB)

```bash
# Enable "USB debugging" in Android Settings → Developer Options first
adb install android/app/build/outputs/apk/debug/app-debug.apk
```

### Option B — Copy APK

1. Copy `app-debug.apk` to your phone
2. Tap it in Files app
3. Allow "Install unknown apps" when prompted

---

## Workflow

Nothing is processed until you say so — settings never re-render uploaded images behind your back.

1. **Add images** — pick, drop, or share them. They are staged as thumbnails; nothing is processed yet.
2. **Configure** — crop mode, background colour/tolerance, padding, output format, and (optionally) load the AI background-removal model. This panel appears once images are staged.
3. **Process** — press the button to run all staged images. Changing any setting afterwards marks the batch dirty; press again to reprocess with the new settings. No re-upload needed.
4. **Output** — every result has its own Copy / Share / Save buttons and per-image re-trim + Remove BG controls; the toolbar adds ZIP download and Clear all.

The AI model is a one-time download (~40 MB, browser-cached) from step 2. Load it when you want it; it is not required for plain cropping, and each result card decides whether to use it.

---

## How sharing works

```
User opens Gallery → selects image → taps Share
  └─> Android shows share sheet
        └─> "Crop to bounds" (Crop2Bounds) appears
              └─> User taps it
                    └─> MainActivity.handleIntent() fires
                          └─> content:// URI forwarded to JS
                                └─> handleSharedUri() reads file
                                      └─> image is STAGED (not processed yet)
                                            └─> user configures, presses Process
                                                  └─> result card appears
                                                        └─> user taps "↑ Share" or "↓ Save"
```

---

## Build a release APK (for distribution)

To build a signed release APK:

1. Generate a keystore:
   ```bash
   keytool -genkey -v -keystore crop2bounds.keystore \
     -alias crop2bounds -keyalg RSA -keysize 2048 -validity 10000
   ```

2. Add signing config to `android/app/build.gradle`:
   ```groovy
   android {
     signingConfigs {
       release {
         storeFile file('../../crop2bounds.keystore')
         storePassword 'YOUR_STORE_PASSWORD'
         keyAlias 'crop2bounds'
         keyPassword 'YOUR_KEY_PASSWORD'
       }
     }
     buildTypes {
       release {
         signingConfig signingConfigs.release
         minifyEnabled false
       }
     }
   }
   ```

3. Build:
   ```bash
   cd android && ./gradlew assembleRelease
   ```

---

## Project structure

```
crop2bounds-android/
├── capacitor-project/          ← Web app (Capacitor root)
│   ├── index.html              ← The app (crop-only, no BG removal)
│   ├── capacitor.config.json   ← Capacitor config
│   └── package.json            ← npm deps
│
└── app/
    └── src/main/
        ├── AndroidManifest.xml ← Share intent filter
        ├── java/com/crop2bounds/
        │   └── MainActivity.java  ← Intent handler
        └── res/
            ├── xml/file_paths.xml ← FileProvider paths
            └── values/strings.xml
```

---

## Troubleshooting

| Problem | Fix |
|---|---|
| `ANDROID_SDK_ROOT not set` | Export the variable (see Step 0) |
| `gradlew: permission denied` | `chmod +x android/gradlew` |
| Share sheet doesn't show app | Check `<intent-filter>` in manifest; re-install APK |
| White screen on launch | Check `webDir` in capacitor.config.json points to index.html directory |
| Image not loading from share | Ensure `READ_MEDIA_IMAGES` permission granted in Settings |
