# Ejenix Skills & Operational Reference Guide

This guide provides modular, actionable **Skill Cards** for software engineers and AI coding agents operating **Ejenix**. Each skill card specifies its objective, prerequisites, CLI commands, parameters, and verification steps.

---

## Table of Skills

1. [Skill 1: Deploying Control Plane Infrastructure](#skill-1-deploying-control-plane-infrastructure)
2. [Skill 2: Generating Signing Keys & Security Policy](#skill-2-generating-signing-keys--security-policy)
3. [Skill 3: Registering Applications & Managing Auth Tokens](#skill-3-registering-applications--managing-auth-tokens)
4. [Skill 4: Scaffolding Patchable Screens in Flutter](#skill-4-scaffolding-patchable-screens-in-flutter)
5. [Skill 5: Exposing Host Capabilities (`@Patchable` + `ejenix gen`)](#skill-5-exposing-host-capabilities-patchable--ejenix-gen)
6. [Skill 6: Compiling, Signing, & Inspecting Patches](#skill-6-compiling-signing--inspecting-patches)
7. [Skill 7: Uploading & Promoting Patches (Staged Canary Rollout)](#skill-7-uploading--promoting-patches-staged-canary-rollout)
8. [Skill 8: Instant Emergency Rollback](#skill-8-instant-emergency-rollback)
9. [Skill 9: Local Development Watch Loop (`ejenix watch`)](#skill-9-local-development-watch-loop-ejenix-watch)
10. [Skill 10: CI/CD Compatibility Gating & Min-SDK Floor](#skill-10-cicd-compatibility-gating--min-sdk-floor)
11. [Skill 11: Diagnostics & Troubleshooting](#skill-11-diagnostics--troubleshooting)

---

### Skill 1: Deploying Control Plane Infrastructure

**Goal**: Spin up a self-hosted control plane server.

```bash
# Docker / Local Container:
./deploy.sh --target docker

# Google Cloud Platform (Cloud Run + GCS):
./deploy.sh --target gcp --bucket <bucket-name>

# Azure Container Apps:
./deploy.sh --target azure --resource-group <rg> --storage-account <storage>

# Kubernetes:
./deploy.sh --target kubernetes

# Bare-Metal systemd service:
sudo ./deploy.sh --target bare-metal
```

**Verification**:
```bash
curl -fsS http://localhost:8080/v1/health
# Returns: {"status":"ok","version":"..."}
```

---

### Skill 2: Generating Signing Keys & Security Policy

**Goal**: Create an Ed25519 signing keypair and enforce security rules.

```bash
# Generate keypair
ejenix keygen -o release.key

# Secure private seed
echo "*.key" >> .gitignore
chmod 0600 release.key
```

**Rules**:
- `release.key` (private seed): Store in CI/CD Secret Store (e.g., GitHub Secrets). **NEVER commit.**
- `release.key.pub` (public key): Safe to commit and embed in Flutter client code.

---

### Skill 3: Registering Applications & Managing Auth Tokens

**Goal**: Register an application ID on the control plane and manage scoped access tokens.

```bash
# Register application (using Admin Key)
export EJENIX_TOKEN="<ADMIN_KEY>"

ejenix app create \
  --id com.acme.shop \
  --name "Acme Shop" \
  --key release.key.pub \
  --server http://localhost:8080
```

*Output includes an **App API Key** scoped exclusively to `com.acme.shop` for CI operations.*

**List Registered Apps**:
```bash
ejenix app list --server http://localhost:8080
```

---

### Skill 4: Scaffolding Patchable Screens in Flutter

**Goal**: Scaffold an interpreted patch screen and a host view component.

```bash
# Run from Flutter project root
ejenix scaffold home_screen --app-id com.acme.shop
```

Creates:
- `patches/home_screen.dart` (the patch source)
- `lib/home_screen_view.dart` (the Flutter host widget wrapping `EjenixPatchView`)

**Required Post-Scaffold Actions**:
1. Copy standard Flutter Patch SDK:
   ```bash
   mkdir -p patch_sdk
   cp <ejenix-repo>/flutter_bridge/patch_sdk/flutter.dart patch_sdk/
   ```
2. In `lib/home_screen_view.dart`:
   - Set `_trustedKeys` to public key bytes from `release.key.pub`.
   - Set `controlPlane` URL and `env` (`staging` or `production`).
   - Set `fallbackBuilder` to render native fallback widget.

---

### Skill 5: Exposing Host Capabilities (`@Patchable` + `ejenix gen`)

**Goal**: Expose custom host widgets, repositories, and services to patches safely.

**1. Annotate Host Code**:
```dart
import 'package:ejenix_flutter/ejenix_flutter.dart';

@patchable
class CustomButton extends StatelessWidget { ... }

class CartService {
  @Patchable('App.cartTotal')
  double getCartTotal() => 99.99;
}
```

**2. Generate Bindings**:
```bash
ejenix gen lib/services/cart_service.dart \
  --out-sdk patch_sdk/app.dart \
  --out-capabilities lib/app_capabilities.g.dart
```

**3. Pass Capabilities to Host View**:
```dart
EjenixPatchView(
  ...
  extend: appCapabilities(cartServiceInstance),
)
```

---

### Skill 6: Compiling, Signing, & Inspecting Patches

**Goal**: Build a signed CBOR bundle from Dart source code, verify it offline, and inspect disassembly.

**Build & Sign**:
```bash
ejenix build patches/home_screen.dart \
  -o home.bundle \
  --signing-key release.key \
  --app-id com.acme.shop \
  --generation 1 \
  --min-sdk 1.0.0
```

**Verify Offline**:
```bash
ejenix verify home.bundle --key release.key.pub
```

**Inspect Disassembly & Metadata**:
```bash
ejenix inspect home.bundle
```

---

### Skill 7: Uploading & Promoting Patches (Staged Canary Rollout)

**Goal**: Push a bundle to the control plane, promote it to an environment, and configure staged canary rollout.

**1. Upload Bundle**:
```bash
export EJENIX_TOKEN="<APP_API_KEY>"

ejenix push home.bundle \
  --server http://localhost:8080 \
  --app com.acme.shop
# Returns Bundle ID, e.g., bnd_8f3a1...
```

**2. Canary Staged Rollout (5% Fleet Share)**:
```bash
ejenix promote bnd_8f3a1... \
  --channel home \
  --env staging \
  --rollout 5 \
  --server http://localhost:8080 \
  --app com.acme.shop
```

**3. Widen Rollout (100% Fleet Share)**:
```bash
ejenix promote bnd_8f3a1... \
  --channel home \
  --env staging \
  --rollout 100 \
  --server http://localhost:8080 \
  --app com.acme.shop
```

---

### Skill 8: Instant Emergency Rollback

**Goal**: Revert the active release on a channel back to the previously active bundle.

```bash
ejenix rollback \
  --channel home \
  --env production \
  --server http://localhost:8080 \
  --app com.acme.shop
```

*Restores previous bundle to 100% of devices immediately.*

---

### Skill 9: Local Development Watch Loop (`ejenix watch`)

**Goal**: Enable sub-second live updates during local development.

```bash
ejenix watch patches/home_screen.dart \
  --signing-key release.key \
  --app-id com.acme.shop \
  --watch patch_sdk \
  --port 8787
```

*Starts local dev server at `http://localhost:8787` serving live patch bundles on file saves.*

---

### Skill 10: CI/CD Compatibility Gating & Min-SDK Floor

**Goal**: Block patches in CI if they call host capabilities missing from the target binary version.

**1. Generate Capability Manifest in Test**:
```dart
// test/dump_capabilities_test.dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:ejenix_flutter/ejenix_flutter.dart';
import '../lib/app_capabilities.g.dart';

void main() {
  test('dump capability manifest', () {
    Directory('build').createSync(recursive: true);
    File('build/capabilities.json')
        .writeAsStringSync(capabilityManifest(appCapabilities(dummyRepo)));
  });
}
```

**2. Run CI Capability Gate**:
```bash
flutter test test/dump_capabilities_test.dart

ejenix verify home.bundle \
  --key release.key.pub \
  --capabilities build/capabilities.json
```
*Exits with code `65` if the patch calls any capability missing from `build/capabilities.json`.*

---

### Skill 11: Diagnostics & Troubleshooting

**Symptom & Fix Matrix**:

| Symptom | Probable Cause | Corrective Action |
|---|---|---|
| `onStatus` reports `rejected` | Key mismatch or wrong `appId` | Confirm `release.key.pub` matches app binary key & `appId` matches exactly. |
| `onStatus` reports `incompatible` | Missing host capability or `minSdk` > `sdkVersion` | Update native app build to register capability or bump `--min-sdk`. |
| `onStatus` reports `rolledBack` | Patch crashed 3 consecutive times | Check crash logs, fix patch code, and push a higher generation bundle. |
| Patch never updates (`upToDate`) | `env` mismatch (`staging` vs `production`) | Check `EjenixPatchView(env:)` matches `ejenix promote --env`. |
| Screen renders blank / white | Missing `fallbackBuilder` | Provide native fallback widget in `EjenixPatchView(fallbackBuilder:)`. |
| 409 Conflict on `rollback` | No previous bundle exists on server | Promote at least two bundles before testing rollback. |
| 401 Unauthorized | Missing or invalid auth token | `export EJENIX_TOKEN="<ADMIN_KEY_OR_APP_API_KEY>"`. |
