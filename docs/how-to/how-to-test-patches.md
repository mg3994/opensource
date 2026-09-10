# How to Test Ejenix Patches and Fallback Mechanics

This guide outlines testing methodologies for verifying Ejenix patches, testing native fallback components, and validating crash-loop self-healing mechanics.

---

## 1. Testing Patch Code in Flutter Unit/Widget Tests

Patch files (`patches/home_screen.dart`) are valid Dart code against `patch_sdk/`. You can write standard Flutter widget tests for patch components:

```dart
// test/home_patch_test.dart
import 'package:flutter_test/flutter_test.dart';
import 'package:flutter/material.dart';
import '../patches/home_screen.dart' as patch;

void main() {
  testWidgets('Patch screen renders main title and button', (tester) async {
    await tester.pumpWidget(
      MaterialApp(
        home: patch.main(),
      ),
    );

    expect(find.text('Patched Home Screen'), findsOneWidget);
  });
}
```

---

## 2. CI Capability Verification Test

Ensure patches do not call capabilities missing from the target application build using automated test manifest dumping:

```dart
// test/dump_capabilities_test.dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:ejenix_flutter/ejenix_flutter.dart';
import '../lib/app_capabilities.g.dart';

void main() {
  test('dump capability manifest for CI verification', () {
    Directory('build').createSync(recursive: true);
    File('build/capabilities.json')
        .writeAsStringSync(capabilityManifest(appCapabilities(dummyRepo)));
  });
}
```

Run in CI pipeline:

```bash
flutter test test/dump_capabilities_test.dart

ejenix verify home.bundle \
  --key release.key.pub \
  --capabilities build/capabilities.json
```

---

## 3. Testing Offline & Control Plane Disruption

Test that your application functions smoothly when the control plane is unreachable or offline:

1. Disconnect network or point `controlPlane` URL to an invalid endpoint (`http://10.255.255.1:8080`).
2. Launch the app.
3. Verify that `EjenixPatchView` gracefully falls back:
   - Renders cached patch if present (`PatchStatus.cached`).
   - Renders bundled fallback asset if present (`PatchStatus.bundled`).
   - Renders native `fallbackBuilder` widget if no patch exists.

---

## 4. Simulating Crash-Loop Rollback & Quarantine

Validate the loader's automatic self-healing crash-loop rollback:

### 1. Build a Intentionally Crashing Patch

Create a patch that throws an unhandled exception during `build()`:

```dart
// patches/crash_patch.dart
import '../patch_sdk/flutter.dart';

Widget main() {
  throw Exception('Simulated patch launch crash');
}
```

### 2. Build & Promote Patch

```bash
ejenix build patches/crash_patch.dart -o crash.bundle --signing-key release.key --app-id com.acme.shop --generation 999
ejenix push crash.bundle --app com.acme.shop
ejenix promote bnd_crash --channel home --env staging --app com.acme.shop
```

### 3. Launch Application & Observe Recovery

1. Launch application: The loader fetches `crash.bundle` and attempts to mount it. The patch throws an exception.
2. The loader catches the error, increments the crash counter in `loader_state.json`, and renders `fallbackBuilder`.
3. Relaunch application 3 consecutive times:
   - On the 3rd crash, the loader marks `bnd_crash` as **Quarantined**.
   - The loader purges `crash.bundle` from local storage and permanently rolls back to the previous patch (or native fallback).
   - `onStatus` reports `PatchStatus.rolledBack`.
