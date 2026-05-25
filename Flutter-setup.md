# Complete Flutter Setup Guide for Windows

> Full Flutter installation and setup guide for Windows with Android Studio

---

# Table of Contents

1. Introduction
2. System Requirements
3. Install Flutter SDK
4. Add Flutter to PATH
5. Install Android Studio Plugins
6. Install Android SDK
7. Accept Android Licenses
8. Verify Installation
9. Setup Android Emulator
10. Create First Flutter App
11. VS Code Setup
12. Useful Flutter Commands
13. Common Errors & Fixes
14. Flutter Folder Structure
15. Flutter Learning Roadmap
16. Recommended Packages
17. Beginner Projects
18. Useful Resources

---

# 1. Introduction

Flutter is Google's cross-platform UI framework used to build:

* Android apps
* iOS apps
* Web apps
* Desktop apps

Using a single codebase.

Official Website:
https://flutter.dev

---

# 2. System Requirements

Recommended Requirements:

| Requirement     | Recommended              |
| --------------- | ------------------------ |
| OS              | Windows 10/11            |
| RAM             | 8GB minimum              |
| Recommended RAM | 16GB                     |
| Storage         | 20GB+ Free Space         |
| Processor       | Intel i5/i7 or Ryzen 5/7 |
| Virtualization  | Enabled                  |

---

# 3. Install Flutter SDK

## Step 1 — Download Flutter SDK

Open:

https://docs.flutter.dev/get-started/install/windows

Download:

* Flutter SDK for Windows

You will get:

```bash
flutter_windows_x.x.x-stable.zip
```

---

## Step 2 — Extract Flutter

Extract Flutter SDK to:

```bash
C:\flutter
```

IMPORTANT:
Do NOT place Flutter inside:

```bash
C:\Program Files
```

Avoid:

* Desktop
* Downloads
* Documents

Correct Structure:

```bash
C:\
 └── flutter
      ├── bin
      ├── cache
      ├── packages
      └── ...
```

---

# 4. Add Flutter to PATH

## Step 1

Search in Windows:

```bash
Environment Variables
```

Open:

```bash
Edit the system environment variables
```

---

## Step 2

Click:

```bash
Environment Variables
```

---

## Step 3

Under:

```bash
System Variables
```

Select:

```bash
Path
```

Click:

```bash
Edit
```

---

## Step 4

Click:

```bash
New
```

Add:

```bash
C:\flutter\bin
```

Click OK on all windows.

---

# 5. Verify Flutter Installation

Open NEW CMD or PowerShell:

```bash
flutter --version
```

Expected Output:

```bash
Flutter 3.x.x
Dart 3.x.x
```

---

# 6. Install Flutter Plugin in Android Studio

Open Android Studio.

Go to:

```bash
File → Settings → Plugins
```

Search:

```bash
Flutter
```

Install:

* Flutter Plugin
* Dart Plugin

Restart Android Studio.

---

# 7. Install Android SDK Components

Open Android Studio.

Go to:

```bash
More Actions → SDK Manager
```

---

## SDK Platforms

Install:

* Android 14 SDK
  or
* Android 15 SDK

---

## SDK Tools

Enable:

```bash
Android SDK Build-Tools
Android Emulator
Android SDK Platform-Tools
Android SDK Command-line Tools
Intel x86 Emulator Accelerator (if available)
```

Click:

```bash
Apply
```

---

# 8. Accept Android Licenses

Open terminal:

```bash
flutter doctor --android-licenses
```

Press:

```bash
y
```

for all prompts.

---

# 9. Run Flutter Doctor

Run:

```bash
flutter doctor
```

Expected Result:

```bash
[✓] Flutter
[✓] Android toolchain
[✓] Android Studio
[✓] Connected device
```

If any issue exists, Flutter will show fixes.

---

# 10. Setup Android Emulator

Open Android Studio.

Go to:

```bash
More Actions → Virtual Device Manager
```

---

## Create Emulator

Recommended Device:

```bash
Pixel 7
```

---

## Download System Image

Recommended:

```bash
Android 14
x86_64
```

Finish setup.

---

## Start Emulator

Click:

```bash
Play Button
```

The Android emulator should start.

---

# 11. Create First Flutter Project

Open terminal.

Run:

```bash
flutter create myapp
```

Go inside project:

```bash
cd myapp
```

Run app:

```bash
flutter run
```

---

# 12. VS Code Setup (Optional)

Download VS Code:

https://code.visualstudio.com/

Install Extensions:

* Flutter
* Dart

Recommended Extensions:

* Error Lens
* GitLens
* Material Icon Theme

---

# 13. Useful Flutter Commands

## Check Installation

```bash
flutter doctor
```

---

## Create App

```bash
flutter create appname
```

---

## Run App

```bash
flutter run
```

---

## Get Packages

```bash
flutter pub get
```

---

## Upgrade Flutter

```bash
flutter upgrade
```

---

## Clean Project

```bash
flutter clean
```

---

## Check Connected Devices

```bash
flutter devices
```

---

## Build APK

```bash
flutter build apk
```

---

## Build Release APK

```bash
flutter build apk --release
```

---

# 14. Common Errors & Fixes

---

## ERROR: adb not found

Add this path:

```bash
C:\Users\YOUR_USERNAME\AppData\Local\Android\Sdk\platform-tools
```

to Environment Variables PATH.

Restart terminal.

---

## ERROR: Android Licenses Not Accepted

Run:

```bash
flutter doctor --android-licenses
```

Accept all licenses.

---

## ERROR: Emulator Slow

Enable:

* Intel VT-x
* AMD-V

from BIOS.

---

## ERROR: SDK Not Found

Open Android Studio → SDK Manager.

Install:

* SDK Platform
* Build Tools
* Command Line Tools

---

## ERROR: Flutter Command Not Found

Verify PATH contains:

```bash
C:\flutter\bin
```

Restart PC.

---

# 15. Recommended Flutter Folder Structure

```bash
project/
│
├── lib/
│   ├── screens/
│   ├── widgets/
│   ├── services/
│   ├── models/
│   ├── providers/
│   ├── utils/
│   └── main.dart
│
├── assets/
│   ├── images/
│   └── fonts/
│
├── android/
├── ios/
├── web/
├── test/
│
└── pubspec.yaml
```

---

# 16. Flutter Learning Roadmap

---

## Phase 1 — Dart Basics

Learn:

* Variables
* Data Types
* Functions
* Loops
* Conditions
* OOP
* Async/Await
* Futures
* Streams

---

## Phase 2 — Flutter Basics

Learn:

* Widgets
* StatelessWidget
* StatefulWidget
* Layouts
* Navigation
* Forms
* Lists
* GridView

---

## Phase 3 — State Management

Learn:

* Provider
* Riverpod
* Bloc

Recommended:

```bash
Riverpod
```

---

## Phase 4 — Backend Integration

Learn:

* REST APIs
* Dio Package
* Firebase
* Authentication
* JWT
* Local Storage

---

## Phase 5 — Advanced Flutter

Learn:

* Animations
* Push Notifications
* Deep Linking
* App Architecture
* Clean Architecture
* Offline Support

---

# 17. Recommended Packages

---

## State Management

```bash
provider
flutter_riverpod
bloc
```

---

## Networking

```bash
dio
http
```

---

## Local Storage

```bash
shared_preferences
hive
sqflite
```

---

## Firebase

```bash
firebase_core
firebase_auth
cloud_firestore
firebase_messaging
```

---

## UI

```bash
flutter_screenutil
cached_network_image
shimmer
```

---

# 18. Beginner Flutter Projects

1. Calculator App
2. Todo App
3. Notes App
4. Expense Tracker
5. Weather App
6. Chat Application
7. E-Commerce App
8. PDF Reader App
9. AI Chat App
10. Habit Tracker

---

# 19. Recommended YouTube Channels

Flutter Official:
https://www.youtube.com/@flutterdev

CodeWithAndrea:
https://codewithandrea.com/

The Net Ninja:
https://www.youtube.com/@NetNinja

Flutter Mapp:
https://www.youtube.com/@FlutterMapp

---

# 20. Final Verification

Run:

```bash
flutter doctor
```

If everything shows green checkmarks:

```bash
[✓] Flutter
[✓] Android toolchain
[✓] Android Studio
[✓] Connected device
```

Then Flutter setup is completed successfully.

---

# Congratulations 🚀

Your Flutter development environment is now ready.

Next Step:

* Learn Dart
* Build small apps
* Learn state management
* Integrate APIs
* Build production apps
