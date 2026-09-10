# How to Setup Ejenix: Control Plane Hosting, App Integration, and Production Workflow

This guide provides step-by-step instructions for deploying your own **Ejenix** control plane infrastructure, integrating the Ejenix Flutter SDK into your app, generating host capabilities, and establishing a secure production patch delivery pipeline.

---

## Environment Prerequisites

- **Dart SDK**: ≥ 3.11 (Check: `dart --version`)
- **Flutter SDK**: ≥ 3.44 (Check: `flutter --version`)
- **Git & Docker** (for hosting control plane locally or in container environments)

### Installing the `ejenix` CLI

Install the CLI from a local clone of the Ejenix repository:

```bash
git clone https://github.com/ejenix/opensource.git ejenix_repo
cd ejenix_repo
dart pub get
dart pub global activate --source path packages/cli
export PATH="$PATH:$HOME/.pub-cache/bin"
ejenix --version
```

Verify your setup at any time:

```bash
ejenix doctor
```

---

## Step 1: Deploy Your Control Plane Server

The control plane stores and serves patch bundles. It does **not** evaluate or authorize code logic—devices perform cryptographic verification locally.

Choose your deployment target using `deploy.sh`:

### Target 1: Local / Docker / VM

```bash
cd ejenix_repo
./deploy.sh --target docker
```
*Creates `deploy/docker/.env` with generated credentials and starts the server on port `8080`.*

### Target 2: Google Cloud Platform (GCP)

```bash
./deploy.sh --target gcp --bucket my-ejenix-storage
```
*Deploys an AOT control plane instance to Cloud Run connected to Google Cloud Storage.*

### Target 3: Azure Container Apps

```bash
./deploy.sh --target azure --resource-group ejenix-rg --storage-account ejenixstorage
```
*Deploys to Azure Container Apps with Azure Blob Storage backend.*

### Target 4: AWS (ECS/Fargate) or Ephemeral Evaluation

```bash
./deploy.sh --target aws --ephemeral   # Evaluation only (non-persistent volume)
```
*For production on AWS, run the Docker image on ECS/Fargate backed by Amazon EFS.*

### Target 5: Kubernetes

```bash
./deploy.sh --target kubernetes
```
*Applies manifests from `deploy/k8s/control-plane.yaml`.*

### Target 6: Bare-Metal Linux (systemd)

```bash
sudo ./deploy.sh --target bare-metal
```
*Configures a `systemd` service running under a hardened `DynamicUser` profile.*

---

### Control Plane Environment Variables

Set these two environment variables on your server:

| Variable | Description |
|---|---|
| `EJENIX_ADMIN_KEY` | Operator administrative secret for CLI commands and Dashboard login. |
| `EJENIX_DELIVERY_SEED` | Stable 32-byte hexadecimal seed used to sign binary delta updates across restarts. |

### Verify Control Plane Health

```bash
curl -fsS http://localhost:8080/v1/health
```
Response: `{"status":"ok","version":"..."}`

Access the web dashboard at `http://localhost:8080/` and sign in with your `EJENIX_ADMIN_KEY`.

---

## Step 2: Generate Signing Keys & Secure Infrastructure

Ejenix uses Ed25519 digital signatures (RFC 8032).

### 1. Generate Key Pair

From your application repository root:

```bash
ejenix keygen -o release.key
```

This creates two files:
- `release.key`: **Private key seed** (`0600` permission mode). **NEVER commit this file.**
- `release.key.pub`: **Public key**. Safe to embed in your application and register on the server.

### 2. Protect Private Keys

1. Add `*.key` to your project's `.gitignore` immediately:
   ```bash
   echo "*.key" >> .gitignore
   ```
2. Store `release.key` in your CI/CD secret manager (e.g., GitHub Secrets, AWS Secrets Manager, Vault).

---

## Step 3: Register Your Application

Register your application ID and public key on your control plane:

```bash
export EJENIX_TOKEN="<EJENIX_ADMIN_KEY>"

ejenix app create \
  --id com.acme.shop \
  --name "Acme Shop" \
  --key release.key.pub \
  --server http://localhost:8080
```

> **Note on App API Keys**: `app create` outputs an **App API Key**. Store this key for CI/CD operations. It grants access *only* to `com.acme.shop`, enforcing least-privilege security over the global admin key.

Verify registration:

```bash
ejenix app list --server http://localhost:8080
```

---

## Step 4: Add Ejenix SDK to Your Flutter App

In your Flutter app's `pubspec.yaml`:

```yaml
dependencies:
  flutter:
    sdk: flutter
  ejenix_flutter:
    git:
      url: https://github.com/ejenix/opensource.git
      path: flutter_bridge
  path_provider: ^2.1.0

flutter:
  assets:
    - assets/home_screen.bundle
```

Run:

```bash
flutter pub get
```

---

## Step 5: Define & Generate Host Capabilities (`@Patchable`)

Patches can only invoke capabilities registered by the host binary.

### 1. Annotate Host Code

In your Flutter project (e.g., `lib/services/product_service.dart`):

```dart
import 'package:ejenix_flutter/ejenix_flutter.dart';

@patchable
class PrimaryButton extends StatelessWidget {
  final String label;
  final VoidCallback onPressed;
  const PrimaryButton({super.key, required this.label, required this.onPressed});

  @override
  Widget build(BuildContext context) => ElevatedButton(onPressed: onPressed, child: Text(label));
}

class ProductRepository {
  @Patchable('App.fetchProducts')
  List<String> fetchProducts() => ['Product A', 'Product B', 'Product C'];
}
```

### 2. Generate Capability Bindings

Run `ejenix gen` to output both sides of the capability boundary:

```bash
ejenix gen lib/services/product_service.dart \
  --out-sdk patch_sdk/app.dart \
  --out-capabilities lib/app_capabilities.g.dart
```

- `patch_sdk/app.dart`: The typed API stub used when compiling patches.
- `lib/app_capabilities.g.dart`: Contains `appCapabilities(...)` passed to `EjenixPatchView(extend:)`.

### 3. Copy standard Flutter Patch SDK

Copy `flutter.dart` from the Ejenix repo into your local `patch_sdk/`:

```bash
mkdir -p patch_sdk
cp <path-to-ejenix-repo>/flutter_bridge/patch_sdk/flutter.dart patch_sdk/
```

---

## Step 6: Scaffold & Embed Patchable Screens

Generate a patch screen and its corresponding host widget:

```bash
ejenix scaffold home_screen --app-id com.acme.shop
```

This generates:
- `patches/home_screen.dart`: The patch file you modify and ship OTA.
- `lib/home_screen_view.dart`: The host Flutter widget embedding `EjenixPatchView`.

### Configure `lib/home_screen_view.dart`

Open `lib/home_screen_view.dart` and complete the configuration:

```dart
import 'dart:io';
import 'package:flutter/material.dart';
import 'package:flutter/services.dart';
import 'package:ejenix_flutter/ejenix_flutter.dart';
import 'package:path_provider/path_provider.dart';
import 'app_capabilities.g.dart';
import 'native_home_screen.dart'; // Native fallback screen

// Ed25519 Public Key Bytes from release.key.pub
final List<int> _trustedPublicKeyBytes = [
  /* Put the exact 32 bytes from release.key.pub here */
];

class HomeScreenView extends StatelessWidget {
  const HomeScreenView({super.key});

  @override
  Widget build(BuildContext context) {
    return FutureBuilder<_BootData>(
      future: _initBootData(),
      builder: (context, snapshot) {
        if (!snapshot.hasData) {
          return const Center(child: CircularProgressIndicator());
        }
        final boot = snapshot.data!;
        return EjenixPatchView(
          controlPlane: Uri.parse('http://localhost:8080'),
          appId: 'com.acme.shop',
          channel: 'home',
          env: 'staging',
          trustedKeys: [_trustedPublicKeyBytes],
          cacheDir: boot.cacheDir,
          bundledFallback: boot.bundledFallback,
          extend: appCapabilities(repoInstance),
          fallbackBuilder: (context, error) {
            debugPrint('Ejenix patch fallback triggered: $error');
            return const NativeHomeScreen(); // Native fallback
          },
          onStatus: (status) => debugPrint('Ejenix status: $status'),
        );
      },
    );
  }

  Future<_BootData> _initBootData() async {
    final cacheDir = await getApplicationSupportDirectory();
    Uint8List? bundledFallback;
    try {
      final byteData = await rootBundle.load('assets/home_screen.bundle');
      bundledFallback = byteData.buffer.asUint8List();
    } catch (_) {}
    return _BootData(cacheDir, bundledFallback);
  }
}

class _BootData {
  final Directory cacheDir;
  final Uint8List? bundledFallback;
  _BootData(this.cacheDir, this.bundledFallback);
}
```

---

## Step 7: Local Development Loop (`ejenix watch`)

Iterate on patches with sub-second live rebuilds without pushing to the server:

```bash
ejenix watch patches/home_screen.dart \
  --signing-key release.key \
  --app-id com.acme.shop \
  --watch patch_sdk
```

The watch server listens locally and rebuilds the bundle instantly whenever `patches/home_screen.dart` is saved.

---

## Step 8: Build, Verify, and Ship Production Patches

When ready to ship an update OTA:

### 1. Edit Patch Source Code (`patches/home_screen.dart`)

```dart
import '../patch_sdk/flutter.dart';
import '../patch_sdk/app.dart';

Widget main() {
  return Scaffold(
    appBar: AppBar(title: Text('Patched Home Screen')),
    body: Center(
      child: PrimaryButton(
        label: 'Explore Catalog',
        onPressed: () => print('Button tapped'),
      ),
    ),
  );
}
```

### 2. Compile & Sign Bundle

```bash
ejenix build patches/home_screen.dart \
  -o home.bundle \
  --signing-key release.key \
  --app-id com.acme.shop \
  --generation 1
```

### 3. Verify Offline

```bash
ejenix verify home.bundle --key release.key.pub
```

### 4. Copy Initial Asset Bundle (Optional Initial Fallback)

Copy the first built bundle into `assets/home_screen.bundle` for offline first launches:

```bash
mkdir -p assets
cp home.bundle assets/home_screen.bundle
```

### 5. Upload Bundle (`push`)

```bash
export EJENIX_TOKEN="<APP_API_KEY_OR_ADMIN_KEY>"

ejenix push home.bundle \
  --server http://localhost:8080 \
  --app com.acme.shop
```
*Note the returned Bundle ID (e.g. `bnd_9f8a2b...`).*

### 6. Canary Rollout (`promote --rollout`)

Promote the patch to a 5% canary rollout in staging:

```bash
ejenix promote bnd_9f8a2b... \
  --channel home \
  --env staging \
  --rollout 5 \
  --server http://localhost:8080 \
  --app com.acme.shop
```

### 7. Fleet-Wide Promotion

Once verified, widen to 100%:

```bash
ejenix promote bnd_9f8a2b... \
  --channel home \
  --env staging \
  --rollout 100 \
  --server http://localhost:8080 \
  --app com.acme.shop
```

### 8. Emergency Rollback

If an issue arises, revert to the previous active release immediately:

```bash
ejenix rollback \
  --channel home \
  --env staging \
  --server http://localhost:8080 \
  --app com.acme.shop
```

---

## Step 9: CI/CD Pipeline Safety Gates

Prevent compatibility breakage across app versions by running automated capability verification in your CI/CD pipeline.

### 1. Write Capability Dump Test in Flutter Project

Create `test/dump_capabilities_test.dart`:

```dart
import 'dart:io';
import 'package:flutter_test/flutter_test.dart';
import 'package:ejenix_flutter/ejenix_flutter.dart';
import '../lib/app_capabilities.g.dart';

void main() {
  test('dump capability manifest for CI gate', () {
    Directory('build').createSync(recursive: true);
    File('build/capabilities.json')
        .writeAsStringSync(capabilityManifest(appCapabilities(dummyRepo)));
  });
}
```

### 2. CI Gate Execution

In your CI pipeline script:

```bash
# 1. Dump current app capability manifest
flutter test test/dump_capabilities_test.dart

# 2. Verify patch compatibility against the capability manifest
ejenix verify home.bundle \
  --key release.key.pub \
  --capabilities build/capabilities.json
```

If the patch references any host selector missing from `build/capabilities.json`, `ejenix verify` exits with code `65` and fails the build before deployment.
