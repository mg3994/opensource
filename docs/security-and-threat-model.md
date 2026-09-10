# Ejenix Security Architecture, Threat Model, and Store Compliance

This document outlines the **security model**, **threat analysis**, **sandboxing guarantees**, and **app store policy compliance** for Ejenix.

---

## 1. Core Cryptographic Security Model

Ejenix operates on a strict **zero-trust server model**:

```
                       Control Plane (UNTRUSTED)
                          │   Serves signed bytes
                          ▼
                       Client Device (TRUSTED)
                          │
                          ├── Verified against release.key.pub (Embedded in binary)
                          └── Structural Bytecode Check
```

- **Cryptographic Trust Anchor**: The application trusts *only* the Ed25519 public key (`release.key.pub`) compiled into the host binary (`_trustedKeys`).
- **Untrusted Control Plane**: The control plane server stores and serves patches, but is never trusted to authorize code. Even if a control plane server is completely compromised, an attacker cannot execute unauthorized code on client devices because they lack the private key (`release.key`).
- **Signature Scheme**: Every patch bundle carries a 64-byte Ed25519 digital signature (RFC 8032) computed over a canonical, deterministic CBOR payload body.

---

## 2. Threat Analysis & Defensive Safeguards

| Threat Vector | Severity | Ejenix Safeguard & Defense |
|---|---|---|
| **Control Plane Server Compromise** | Critical | Devices verify Ed25519 signatures locally against the compiled public key. Untrusted bytes are rejected immediately with status `rejected`. |
| **Man-In-The-Middle (MITM) Tampering** | Critical | Any byte modification invalidates the canonical CBOR payload SHA-512 digest and fails Ed25519 signature verification. |
| **Replay Attacks (Stale Patch Injection)** | High | Every bundle carries a monotonic `generation` integer. Devices track `highestAcceptedGeneration` and reject any bundle with a lower generation number. |
| **Malformed / Corrupted Bytecode Injection** | High | Before staging, the device loader performs **Structural Verification**: checking valid opcodes (0..61), register bounds, jump target alignment, exception handler bounds, and class hierarchy acyclicity. |
| **Infinite Loop / UI Thread Denial of Service** | Medium | Every host invocation is metered with an instruction budget (`StepLimitExceededException` at 5,000,000 steps). |
| **Unbounded Memory Allocation** | Medium | Host runtime string and collection allocations carry hard ceilings (`kMaxHostStringLength`, `kMaxHostElements`), throwing `HostBudgetExceededException`. |
| **Privilege Escalation / System Access** | Critical | Compiler permanently bans `dart:io`, `dart:ffi`, `dart:mirrors`, `dart:isolate`, `Process`, `MethodChannel`, and `DynamicLibrary` at compile time. |
| **Device Privacy Tracking / User Profiling** | Low | Staged rollouts calculate eligibility locally on device using `sha256(installId + salt) mod 100`. Zero device telemetry or registry is sent to the server. |

---

## 3. Sandboxing & Boundary Controls

### Compile-Time API Prohibition

The Ejenix compiler (`packages/compiler`) analyzes AST nodes prior to emitting bytecode. Attempts to import or reference forbidden APIs raise fatal compiler diagnostics (`E0001` / `E0100`):

```
Forbidden APIs (Rejected at Compile Time):
  • dart:io             • Process / Process.run
  • dart:ffi            • MethodChannel / BinaryMessenger
  • dart:mirrors        • DynamicLibrary / dlopen
  • dart:isolate        • System environment / OS bridges
```

### Host Capability Allow-List (`HostRegistry`)

Patches run in an isolated execution environment. They cannot access native platform APIs directly. They can only invoke host selectors explicitly registered via `HostRegistry`:
- **Standard UI Surface**: Pre-registered Flutter widgets (191 standard Flutter UI components).
- **Custom App Capabilities**: Registered using `@Patchable` annotations and generated via `ejenix gen`.

---

## 4. Replay Prevention Mechanics

To ensure a stale or malicious server cannot force a device to downgrade to an older, vulnerable patch:

1. Every patch build is stamped with a monotonic sequence number (`--generation <N>`).
2. The client loader records `highestAcceptedGeneration` in its atomic local state file (`loader_state.json`).
3. If `fetchedBundle.generation < state.highestAcceptedGeneration`, the loader rejects the update with `PatchStatus.rejected`.

---

## 5. Store Review & Policy Compliance (Apple & Google Play)

Ejenix is engineered to comply with public app store guidelines and is running live in production on both the **Apple App Store** and **Google Play Store**.

### Apple App Store Guidelines Analysis

- **Developer Program License Agreement (DPLA) §3.3.1(B) - Executable Code**:
  - *Requirement*: Downloaded interpreted code must not change the primary purpose of the application, bypass signing/sandboxing, or create an app store.
  - *Ejenix Compliance*: Ejenix bytecode runs inside an interpreter packaged inside the reviewed binary. It operates strictly within the application process sandbox, touches no OS security boundaries, and uses no dynamic linking (`dlopen`).
- **App Store Review Guideline 2.5.2**:
  - *Requirement*: Apps must be self-contained and not download or execute code that introduces new features or functionality outside review.
  - *Ejenix Compliance*: Patches can only call capabilities registered by the reviewed host binary. A patch cannot introduce new native capabilities that were not present in the submitted build.

### Google Play Store Policy Analysis

- **Device and Network Abuse Policy**:
  - *Requirement*: Code running in a virtual machine or interpreter must not facilitate policy violations.
  - *Ejenix Compliance*: The capability allow-list is fixed at build time. The public key is compiled into the app, preventing arbitrary remote code execution outside the host capability boundary.

---

## 6. Developer Compliance Guidelines

To maintain full store compliance, developers should follow these rules:

1. **Bug Fixes & UI Polish**: Use OTA patches for fixing broken UI flows, copy changes, layout fixes, and merchandising adjustments.
2. **Feature Releases**: For major new capabilities or structural purpose changes, submit a standard app store release.
3. **Capability Bumping**: When adding a new `@Patchable` capability, bump `sdkVersion` in the app and set `--min-sdk` on patches so older app builds reject patches expecting missing native code.
