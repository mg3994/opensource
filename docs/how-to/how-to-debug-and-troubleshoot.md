# How to Debug and Troubleshoot Ejenix Patches

This guide provides systematic troubleshooting procedures for diagnosing and resolving issues across the Ejenix patch lifecycle.

---

## 1. Diagnostic Decision Tree & Status Codes (`onStatus`)

When an `EjenixPatchView` mounts or returns to the foreground, it reports status updates via its `onStatus` callback.

```
                           EjenixPatchView Mounts
                                     │
                                     ▼
                            [ Fetch / Inspect ]
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         ▼                           ▼                           ▼
[ PatchStatus.updated ]     [ PatchStatus.cached ]     [ PatchStatus.upToDate ]
 (New patch downloaded       (Running existing patch    (No newer patch staged;
  and rendered live)          stored in local cache)     network checked ok)
         │                           │                           │
         └───────────────────────────┼───────────────────────────┘
                                     │
                                     ▼
                      Did something fail during run?
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         ▼                           ▼                           ▼
[ PatchStatus.rejected ]   [ PatchStatus.incompatible ] [ PatchStatus.rolledBack ]
 (Signature check failed     (Missing host capability    (Patch crashed 3 times;
  or appId/env mismatch)     or minSdk floor mismatch)    quarantined on device)
```

### Detailed Status Code Reference

| Status Code | Description | Root Cause & Remediation |
|---|---|---|
| `updated` | Newer patch downloaded and rendered successfully. | Expected successful path. |
| `cached` | Loaded and rendered previously cached patch. | Offline launch or no active network connection. |
| `bundled` | Loaded patch bundled inside APK/IPA assets. | Initial launch fallback asset execution. |
| `upToDate` | Control plane checked; active bundle already staged. | Control plane checked successfully; no newer patch ID. |
| `rejected` | Bundle rejected during signature or structural check. | **1.** Key mismatch: `_trustedKeys` in host view doesn't match key used in `ejenix build`. **2.** `appId` mismatch. **3.** Bundle generation number is lower than device's `highestAcceptedGeneration`. |
| `incompatible` | Bundle cannot run on this binary build. | **1.** Patch invokes `@Patchable` capability missing from host app. **2.** `--min-sdk` on bundle is higher than `sdkVersion` declared by app. |
| `rolledBack` | Patch crashed repeatedly; device rolled back. | Patch threw exceptions on launch 3 consecutive times. Quarantine triggered; check debug console/Sentry for stack traces. |

---

## 2. Troubleshooting Common Scenarios

### Scenario 1: Patch Never Arrives / Device Shows Fallback

**Symptoms**:
- `ejenix promote` succeeded on control plane, but device continues rendering native fallback widget or `upToDate`.

**Checklist**:
1. **Empty Trust Anchor**: Check `_trustedKeys` in `lib/home_screen_view.dart`. If it is `[]` (empty list), the loader rejects all patches silently. Copy the 32 bytes from `release.key.pub`.
2. **Environment Mismatch**: Confirm `EjenixPatchView(env: 'staging')` matches `ejenix promote --env staging`. If host app is on `production`, it ignores staging promotes.
3. **App ID Mismatch**: Confirm `appId` on `EjenixPatchView`, `ejenix build --app-id`, and `ejenix app create --id` match down to case.
4. **Channel Mismatch**: Confirm `EjenixPatchView(channel: 'home')` matches `ejenix promote --channel home`.

### Scenario 2: Patch Throws `MissingHostCapabilityException`

**Symptoms**:
- `onStatus` reports `incompatible` or debug log displays `MissingHostCapabilityException: Selector 'App.foo' not found`.

**Fix**:
1. Confirm host app decorated the method/class with `@Patchable('App.foo')`.
2. Re-run `ejenix gen` to update `lib/app_capabilities.g.dart`.
3. Confirm host view passes `extend: appCapabilities(...)` to `EjenixPatchView`.
4. If the capability was added in a new native release, stamp the patch with `--min-sdk <new-version>`.

### Scenario 3: Instruction Budget Overrun (`StepLimitExceededException`)

**Symptoms**:
- Screen falls back to native widget; log reports `StepLimitExceededException: Execution exceeded step limit 5000000`.

**Fix**:
1. Check patch code for infinite loops (`while (true) {}`).
2. Check for unbounded sync iteration over huge datasets inside `build()`.
3. If legitimate heavy computation is required, raise `stepLimit:` on `EjenixPatchView(stepLimit: 10000000)`.

---

## 3. CLI Inspection Tools

Inspect bundle metadata, headers, constant pool, and disassembly offline:

```bash
# Inspect bundle metadata, minSdk, generation, and capability requirements
ejenix inspect home.bundle

# Output machine-readable JSON for automated inspection
ejenix inspect home.bundle --json
```

---

## 4. Server & HTTP Diagnostic Errors

| HTTP Status | Error Message | Solution |
|---|---|---|
| `401 Unauthorized` | Invalid or missing token | Set `export EJENIX_TOKEN="<ADMIN_KEY_OR_APP_API_KEY>"`. |
| `404 Not Found` | App or channel not found | Run `ejenix app create` or confirm `--app` parameter. |
| `409 Conflict` | Bundle ID already exists with different bytes | Bundle IDs are immutable content digests. Recompile under a new build or generation number. |
| `409 Conflict` | No previous bundle to roll back to | Expected when attempting `ejenix rollback` on a channel with only 1 historical release. Promote a second bundle first. |
