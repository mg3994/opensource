# How to Manage Versioning, Capability Evolution, and Schema Migration

This guide explains how to manage application versioning, capability breaking changes, and multi-version fleet compatibility in Ejenix.

---

## 1. Multi-Version Fleet Management Problem

In production, client devices run multiple app binary versions simultaneously (e.g., v1.0.0, v1.1.0, v1.2.0).

When you introduce a new `@Patchable` host capability in app version `1.2.0`, a patch compiled against that new capability cannot run on older client binaries (v1.0.0 or v1.1.0) because the native method binding does not exist in their binary code.

```
Patch v1.2.0 (Needs 'App.newFeature')
    │
    ├──► Client App v1.2.0 (Has 'App.newFeature') ──► Runs successfully
    │
    └──► Client App v1.0.0 (Lacks 'App.newFeature') ──► Rejects patch before staging!
```

---

## 2. Setting Capability Floors with `--min-sdk`

Ejenix resolves multi-version fleet safety using **Capability Floors**:

1. **Declare `sdkVersion` in Host App**:
   Pass semantic version to `EjenixPatchView`:
   ```dart
   EjenixPatchView(
     appId: 'com.acme.shop',
     sdkVersion: '1.2.0', // Current app binary version
     ...
   )
   ```

2. **Stamp Patch with `--min-sdk`**:
   When building a patch that requires capabilities introduced in v1.2.0:
   ```bash
   ejenix build patches/home_screen.dart \
     -o home.bundle \
     --signing-key release.key \
     --app-id com.acme.shop \
     --min-sdk 1.2.0
   ```

3. **Loader Filtering on Device**:
   When a client device fetches the active patch, the loader compares `bundle.minSdk` against `app.sdkVersion`:
   - If `app.sdkVersion (1.0.0) < bundle.minSdk (1.2.0)`, the device rejects the patch before staging and keeps running its cached/bundled patch.
   - Devices running `sdkVersion >= 1.2.0` accept and stage the patch.

---

## 3. Handling Breaking Changes to Capabilities

A capability signature change (e.g., modifying parameters or changing argument types) is a **breaking change**.

### Best Practices for Capability Evolution

| Goal | Recommended Approach |
|---|---|
| **Add new capability** | Decorate with `@Patchable`, bump `sdkVersion` in app, set `--min-sdk` on new patches. |
| **Add optional parameter** | Add optional named parameter with default value in host code. Safe for older patches. |
| **Rename or change parameter type** | **DO NOT** modify existing `@Patchable` selector in place. Deprecate old selector, register new selector (e.g., `App.fetchProductsV2`), bump `sdkVersion`, and use `--min-sdk`. |

---

## 4. Replay Prevention via Sequence Generations (`--generation`)

To prevent replay attacks or accidental downgrades:

```bash
# Release 1
ejenix build patches/home.dart -o home_v1.bundle --signing-key release.key --app-id com.acme.shop --generation 100

# Release 2 (Higher generation number)
ejenix build patches/home.dart -o home_v2.bundle --signing-key release.key --app-id com.acme.shop --generation 101
```

- Device records `highestAcceptedGeneration = 100`.
- When `home_v2.bundle` (`generation = 101`) arrives, device accepts it and updates `highestAcceptedGeneration = 101`.
- If an attacker attempts to re-serve `home_v1.bundle` (`generation = 100`), device rejects it immediately with status `rejected`.
