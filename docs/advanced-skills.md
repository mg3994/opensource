# Advanced Ejenix Skills & Enterprise Workflows

This guide covers advanced operational skill cards for enterprise deployments, multi-channel screen management, binary delta updates, CI/CD automation, complex capability marshaling, and performance benchmarking.

---

## Table of Advanced Skills

12. [Skill 12: Multi-Screen Channel Architecture](#skill-12-multi-screen-channel-architecture)
13. [Skill 13: Binary Delta Updates (`ejenix build-delta`)](#skill-13-binary-delta-updates-ejenix-build-delta)
14. [Skill 14: GitHub Actions Automated CI/CD Pipeline](#skill-14-github-actions-automated-cicd-pipeline)
15. [Skill 15: Complex Type Marshaling in Capabilities](#skill-15-complex-type-marshaling-in-capabilities)
16. [Skill 16: Performance Profiling & Benchmarking (`ejenix bench`)](#skill-16-performance-profiling--benchmarking-ejenix-bench)

---

### Skill 12: Multi-Screen Channel Architecture

**Goal**: Manage and promote independent patchable screens (channels) under a single application ID and signing key.

**Concept**: The control plane keys active bundles on `(appId, channel, env)`. Different screens (`home`, `checkout`, `account`) promote and roll back independently.

**Commands**:
```bash
# Scaffold screens
ejenix scaffold home_screen --app-id com.acme.shop
ejenix scaffold checkout_screen --app-id com.acme.shop

# Build bundles
ejenix build patches/home_screen.dart -o home.bundle --signing-key release.key --app-id com.acme.shop
ejenix build patches/checkout_screen.dart -o checkout.bundle --signing-key release.key --app-id com.acme.shop

# Push & Promote independently
ejenix push home.bundle --app com.acme.shop
ejenix promote bnd_home123 --channel home --env production --app com.acme.shop

ejenix push checkout.bundle --app com.acme.shop
ejenix promote bnd_chk456 --channel checkout --env production --rollout 10 --app com.acme.shop
```

**Flutter Client Setup**:
```dart
// Home View
EjenixPatchView(appId: 'com.acme.shop', channel: 'home', ...)

// Checkout View
EjenixPatchView(appId: 'com.acme.shop', channel: 'checkout', ...)
```

---

### Skill 13: Binary Delta Updates (`ejenix build-delta`)

**Goal**: Produce ultra-compact VCDIFF-style delta patches against a base bundle version to minimize client download bandwidth.

**Command**:
```bash
ejenix build-delta \
  --base base_v1.bundle \
  patches/home_screen_v2.dart \
  -o delta_v1_to_v2.bundle \
  --signing-key release.key \
  --app-id com.acme.shop
```

*Output*:
```
✓ compiled 5 function(s) -> full bundle (751 bytes)
✓ wrote delta -> delta_v1_to_v2.bundle (142 bytes, 18.9% of full bundle)
```

**Workflow**:
When pushed to the control plane, client devices carrying `base_v1.bundle` fetch only the 142-byte delta, reconstruct `v2`, verify the Ed25519 signature, and stage `v2`.

---

### Skill 14: GitHub Actions Automated CI/CD Pipeline

**Goal**: Automate capability verification, patch compilation, signing, testing, and promotion via GitHub Actions.

Create `.github/workflows/ejenix-patch.yml`:

```yaml
name: Deploy Ejenix OTA Patch

on:
  push:
    branches: [ main ]
    paths:
      - 'patches/**'

jobs:
  deploy-patch:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: subosito/flutter-action@v2
        with:
          channel: 'stable'

      - name: Install Ejenix CLI
        run: |
          git clone https://github.com/ejenix/opensource.git ejenix_cli
          cd ejenix_cli && dart pub get
          dart pub global activate --source path packages/cli
          echo "$HOME/.pub-cache/bin" >> $GITHUB_PATH

      - name: Export Private Key Secret
        run: |
          echo "${{ secrets.EJENIX_PRIVATE_KEY }}" > release.key
          chmod 0600 release.key

      - name: Dump App Capability Manifest
        run: flutter test test/dump_capabilities_test.dart

      - name: Build and Sign Patch Bundle
        run: |
          ejenix build patches/home_screen.dart \
            -o home.bundle \
            --signing-key release.key \
            --app-id com.acme.shop \
            --generation ${{ github.run_number }} \
            --min-sdk 1.0.0

      - name: Verify Capabilities Gate
        run: |
          ejenix verify home.bundle \
            --key release.key.pub \
            --capabilities build/capabilities.json

      - name: Upload and Promote to Staging (100%)
        env:
          EJENIX_TOKEN: ${{ secrets.EJENIX_APP_API_KEY }}
        run: |
          BUNDLE_ID=$(ejenix push home.bundle --server https://ejenix.acme.com --app com.acme.shop --json | jq -r .bundleId)
          ejenix promote $BUNDLE_ID \
            --channel home \
            --env staging \
            --rollout 100 \
            --server https://ejenix.acme.com \
            --app com.acme.shop
```

---

### Skill 15: Complex Type Marshaling in Capabilities

**Goal**: Safely pass widgets, primitives, callbacks, and data maps across the host-patch capability boundary.

**Supported Types across Sandbox Boundary**:
- **Primitives**: `String`, `int`, `double`, `num`, `bool`, `null`
- **Collections**: `List<Object?>`, `Map<String, Object?>`
- **Widgets**: Any host `Widget` returned by a capability
- **Callbacks**: `VoidCallback` (0 args), `ValueChanged<T>` (1 arg)

**Example**:
```dart
// Host Code Definition:
class UserTile extends StatelessWidget {
  final String userName;
  final ValueChanged<String> onSelect;
  const UserTile({super.key, required this.userName, required this.onSelect});

  @override
  Widget build(BuildContext context) {
    return ListTile(
      title: Text(userName),
      onTap: () => onSelect(userName),
    );
  }
}

// In Patch SDK (patch_sdk/app.dart):
external Widget UserTile({required String userName, required ValueChanged<String> onSelect});

// Usage in Patch Source Code:
Widget buildUserList() {
  return UserTile(
    userName: 'Alice',
    onSelect: (name) => print('Selected user: $name'),
  );
}
```

---

### Skill 16: Performance Profiling & Benchmarking (`ejenix bench`)

**Goal**: Profile compiler throughput, VM instruction rate, and dev-loop latency.

**Command**:
```bash
ejenix bench
```

**Example Benchmark Output**:
```
  COMPILER
    fibonacci(30)         0.41 ms  ( 2,439 ops/sec )
    complex_screen       1.12 ms  (   892 ops/sec )

  INTERPRETER VM
    fibonacci(20)         0.85 ms  ( 1,176 ops/sec,  2.4 M instructions/sec )
    widget_tree_build     0.08 ms  ( 12,500 ops/sec )

  DEV LOOP
    watch_rebuild        14.20 ms  (  70.4 ops/sec )
```
