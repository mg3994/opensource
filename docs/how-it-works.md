# How Ejenix Works: Architecture, Inner Mechanics, and End-to-End Execution Model

This document provides a comprehensive technical deep dive into how **Ejenix** delivers over-the-air (OTA) updates for Flutter and Dart applications. It details the entire system from high-level architectural concepts down to bit-level binary encodings, instruction execution, cryptographic verification, and local device state transitions.

---

## High-Level Architecture

Ejenix compiles a safe subset of Dart to a compact register bytecode, packages and signs it into an Ed25519 CBOR bundle, hosts it on a lightweight self-hosted control plane, and executes it on device inside a sandboxed register Virtual Machine (VM).

```
   ┌────────────────────────────────────────────────────────────────────────┐
   │                           DEVELOPMENT & BUILD                          │
   └────────────────────────────────────────────────────────────────────────┘
    Dart Source (.dart)
           │
           ▼
    [ Compiler ]  ──────► Rejects forbidden APIs (dart:io, ffi, isolates)
           │
           ▼
    Register Bytecode + Constant Pool
           │
           ▼
    [ Bundle Serializer ] ───► Canonical CBOR Encoding
           │
           ▼
    [ Ed25519 Signer ] ──────► Signed Bundle (.bundle)
           │
   ┌───────┴────────────────────────────────────────────────────────────────┐
   │                          CONTROL PLANE SERVER                          │
   └───────┬────────────────────────────────────────────────────────────────┘
           │  ejenix push & promote
           ▼
    [ Control Plane REST API ] ───► Serves active bundle per (appId, channel, env)
           │                        Publishes rollout % and salt
           │
   ┌───────┴────────────────────────────────────────────────────────────────┐
   │                            CLIENT DEVICE                               │
   └───────┬────────────────────────────────────────────────────────────────┘
           │ HTTP GET /v1/apps/.../active (or Delta fetch)
           ▼
    [ Loader Subsystem ]
           │
           ├── 1. Verify Ed25519 Signature against public key in binary
           ├── 2. Structural Bytecode Verification (bounds, opcodes, jump targets)
           ├── 3. Atomic Write to Disk (write-to-sibling + flush + rename)
           └── 4. Check Rollout Eligibility (local Install ID hash against salt)
           │
           ▼
    [ Register VM Interpreter ]
           │
           ├── Instruction Budget Metering (Step limit per call)
           ├── Inline Caches (Polymorphic call dispatch)
           └── Sandboxed Host Capability Bridge
           │
           ▼
    [ Flutter Bridge / UI Engine ]
           │
           └── Swaps Widget Subtree in-place (No restart / no relaunch)
```

---

## 1. Compiler Subsystem (`packages/compiler`)

The Ejenix compiler translates Dart source code into register bytecode. It uses the official Dart `analyzer` package to parse source text into an Abstract Syntax Tree (AST), resolve symbols, enforce language subset rules, and lower code into bytecode functions.

### Supported Dart Language Subset

Ejenix supports a broad, expressive subset of Dart:
- **Classes & Generics**: Instance methods, static fields/methods, constructors, field initializers, type parameters.
- **Mixins & Inheritance**: Mixin application, `super` invocations, abstract interfaces.
- **Functions & Closures**: Anonymous functions, lexical scoping, closure environments capturing `this` or local variables.
- **Control Flow**: `if`/`else`, `for`/`in`, `while`, `do-while`, `switch`/`case` with pattern matching, labelled `break`/`continue`.
- **Expressions**: Arithmetic/logical operators, cascades (`..`), spreads (`...`), records, list/map/set literals.
- **Asynchronous & Generators**: `async`/`await`, `sync*` (yielding `Iterable`), `async*` (yielding `Stream`).

### Security Enforcements & Diagnostic Codes

To guarantee platform compliance and prevent illegal operations, the compiler rejects forbidden capabilities at build time:
- **Forbidden Packages/Libraries**: Imports of `dart:io`, `dart:ffi`, `dart:mirrors`, `dart:isolate`, `dart:developer`, or native process/channel tools (`Process`, `MethodChannel`, `DynamicLibrary`) trigger fatal compiler diagnostic errors.
- **Diagnostic Codes**: Diagnostic errors follow a structured format (`E0000` through `E0200`):
  - `E0000` - Syntax & Structural AST Errors
  - `E0001` - Forbidden Import / API Violation
  - `E0100` - Unsupported Expression / Language Feature
  - `E0101` - Unsupported Statement / Control Flow
  - `E0102` - Type/Class Hierarchy Violation
  - `E0200` - Capability/Host Bridge Binding Failure

### IR & Bytecode Code Generation

1. **AST Lowering**: The compiler walks the AST and transforms constructs (e.g. loops, cascades, pattern matches) into basic control flow graphs.
2. **Register Allocation**: Each bytecode function maintains a frame with registers `r0` to `rN` (up to 256 per frame). Local variables, parameters, and temporary evaluation results are assigned register indices.
3. **Constant Pool Construction**: Strings, integers, doubles, type descriptors, and selector symbols are collected into a indexed constant pool.
4. **Label & Jump Resolution**: Jump instructions (`Jump`, `JumpIfTrue`, `JumpIfFalse`, `JumpIfNull`) initially emit placeholder offsets, which are backpatched once label targets are emitted.
5. **Exception Table Emission**: `try`/`catch`/`finally` blocks produce exception handler entries specifying start offset, end offset, target handler offset, and catch type index.

---

## 2. Register Bytecode & VM Interpreter (`packages/bytecode`, `packages/interpreter`)

The Ejenix virtual machine is a register-based interpreter engineered for low overhead, small binary footprint, and strict sandboxing.

### Bytecode Instruction Set Architecture (ISA)

The VM implements 62 opcodes operating on 8-bit or 16-bit register operands and constant pool indices:
- **Data Movement & Constants**: `LoadConst`, `LoadInt`, `LoadBool`, `Move`, `LoadNull`.
- **Object Operations**: `NewObject`, `GetField`, `SetField`, `GetStatic`, `SetStatic`.
- **Method & Function Dispatch**: `InvokeDirect`, `InvokeVirtual`, `InvokeStatic`, `InvokeClosure`.
- **Control Flow & Jumps**: `Jump`, `JumpIfTrue`, `JumpIfFalse`, `JumpIfNull`, `Return`, `Throw`.
- **Arithmetic & Logic**: `Add`, `Sub`, `Mul`, `Div`, `Mod`, `BitAnd`, `BitOr`, `BitXor`, `Shl`, `Shr`, `Negate`, `Not`, `Equals`, `LessThan`, etc.
- **Async & Yield**: `YieldSync`, `YieldAsync`, `Await`.
- **Host Capability Bridge**: `InvokeHost` (interfacing directly with host registered functions and Flutter UI bindings).

### Dynamic Dispatch & Inline Caches (ICs)

Dynamic method calls (`rResult = rReceiver.method(rArg1, rArg2)`) use an Inline Cache (IC) attached to each dispatch callsite instruction:
1. **Fast Path (IC Hit)**: On the initial call, the VM looks up the target method in the receiver's class table and caches the class ID and function pointer in the IC slot. Subsequent calls with the same receiver class execute via a direct pointer jump.
2. **Slow Path (IC Miss / Polymorphism)**: If the receiver class ID changes, the VM falls back to a class hierarchy lookup table, updates the cache (or transitions to a polymorphic lookup stub), and completes the call.

### Instruction Budgeting & Step Limits

To prevent infinite loops (`while (true) {}`) or runaway computation from freezing the Flutter UI thread, the interpreter enforces an instruction budget:
- **Logical Invocation Metering**: Every entry from the host into the VM is allocated an instruction budget (`kDefaultPatchStepLimit = 5,000,000` steps).
- **Step Counting**: Every opcode execution decrements the step counter.
- **Budget Overrun**: If steps reach 0, the VM throws `StepLimitExceededException`. This exception bypasses interpreted `catch` blocks and bubbles directly to the host loader, triggering fallback rendering and quarantining the faulty bundle.
- **Deterministic Metering**: Step counts are strictly instruction-based (not clock-time based), guaranteeing identical, reproducible execution across fast and slow hardware.
- **Host Allocation Ceilings**: Operations that invoke host runtime allocation carry fixed caps (e.g. `kMaxHostStringLength`, `kMaxHostElements`) throwing `HostBudgetExceededException` if exceeded.

---

## 3. Bundle Format & Cryptographic Verification (`packages/bundle`)

Patches are serialized into canonical CBOR (Concise Binary Object Representation) bundles signed with Ed25519 (RFC 8032).

### Binary Bundle Specification

A bundle artifact contains two primary sections: an Envelope Header and a Verified Body.

```
┌────────────────────────────────────────────────────────────────────────┐
│                          BUNDLE ENVELOPE HEADER                        │
├─────────────────┬──────────────────────────────────────────────────────┤
│ Magic Header    │ 4 bytes: 0x45 0x4A 0x4E 0x58 ("EJNX")                │
│ Format Version  │ uint16 (e.g., 1)                                     │
│ Signature       │ 64 bytes Ed25519 signature over canonical CBOR Body  │
├─────────────────┴──────────────────────────────────────────────────────┤
│                         CANONICAL CBOR BODY                            │
├─────────────────┬──────────────────────────────────────────────────────┤
│ appId           │ String (e.g., "com.acme.shop")                       │
│ channel         │ String (e.g., "home")                                │
│ environment     │ String (e.g., "production")                          │
│ minSdk          │ String / Semantic Version (e.g., "1.2.0")            │
│ generation      │ uint64 monotonic sequence number                     │
│ createdAt       │ ISO-8601 Timestamp String                            │
│ constantPool    │ List of Constants (Strings, Ints, Doubles, Types)    │
│ classes         │ Class Descriptors & Member Layouts                   │
│ functions       │ List of Compiled Bytecode Functions (Opcodes & Regs) │
└─────────────────┴──────────────────────────────────────────────────────┘
```

### Deterministic CBOR Encoding

To ensure reproducibility:
- Keys in map structures are strictly sorted lexicographically.
- Floating-point numbers and integers are formatted in minimal canonical IEEE 754 representation.
- Compiling the exact same Dart source with the same compiler version yields a **byte-identical** CBOR payload.

### Two-Tier Device Verification

When a client device receives a bundle, it performs two distinct verification steps *before* writing the bundle to permanent patch storage or staging it for execution:

1. **Cryptographic Ed25519 Signature Check**:
   - The device extracts the public key embedded inside the application's compiled binary (`_trustedKeys`).
   - It computes the SHA-512 digest of the canonical CBOR body and verifies the 64-byte Ed25519 signature.
   - If signature verification fails, the bundle is immediately discarded with status `rejected`. The control plane is never trusted for code authority.

2. **Structural Bytecode Verification**:
   - The loader parses the verified CBOR structure and inspects every function and class descriptor.
   - **Opcode Check**: Verifies that every instruction opcode is valid (0..61).
   - **Register Bounds Check**: Confirms all operand registers `rN` are within the function's declared register frame count.
   - **Jump Alignment Check**: Confirms jump targets point to valid instruction boundaries within the function's bytecode array.
   - **Handler Bounds Check**: Validates exception handler start, end, and target offsets.
   - **Class Graph Acyclicity Check**: Verifies that the class inheritance graph contains no circular references.
   - If structural verification fails, the bundle is rejected before execution can take place.

---

## 4. Binary Delta Updates (`packages/delta`)

To keep update payloads minimal (often under 1 KB), Ejenix supports VCDIFF-style binary delta encoding between bundles.

```
 Base Bundle (v1) [On Device] ──┐
                               ├──► [ Delta Engine ] ──► Target Bundle (v2) ──► Verify Signature
 Binary Delta Patch [Fetched] ──┘
```

1. **Delta Generation**: The server computes a binary diff between `Bundle_v1` and `Bundle_v2` using byte-level copy and insert instructions.
2. **Delta Application**: The device loader receives the delta bytes, reads `Bundle_v1` from its local cache, applies the delta operations, and reconstructs `Bundle_v2`.
3. **Post-Reconstruction Verification**: The reconstructed `Bundle_v2` is subjected to the full Ed25519 signature and structural bytecode checks. If verification succeeds, it is staged as the active patch.

---

## 5. Device Lifecycle & Self-Healing Loader (`packages/loader`)

The client loader manages local caching, staging, rendering, and crash recovery.

```
                        ┌────────────────────────┐
                        │   EjenixPatchView      │
                        └───────────┬────────────┘
                                    │ Mount / Resume
                                    ▼
                        ┌────────────────────────┐
                        │ Read Local State File  │
                        └───────────┬────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
     [ Active Patch ]     [ Bundled Fallback ]   [ Native Fallback ]
      (Cached on disk)     (Asset in app APK)    (fallbackBuilder)
              │                     │                     │
              └─────────────────────┼─────────────────────┘
                                    │
                                    ▼
                        ┌────────────────────────┐
                        │ Mount Interpreter VM   │
                        └───────────┬────────────┘
                                    │
                    ┌───────────────┴───────────────┐
                    │                               │
            Success │                               │ Failure / Crash
                    ▼                               ▼
       Render Patch Subtree              Increment Crash Counter
                                                    │
                                     Crash Count >= Threshold?
                                        ├── NO  ──► Render Fallback
                                        └── YES ──► [ QUARANTINE BUNDLE ]
                                                    Auto-Rollback to Last
                                                    Known Good Patch
```

### Atomic Disk Storage (Crash-Safe)

All loader updates to the local state file (`loader_state.json`) and active bundle files use atomic disk operations:
1. Write target bundle data to a temporary sibling file (`bundle.tmp`).
2. Flush file descriptors to physical disk (`flush()`).
3. Atomically rename `bundle.tmp` to `bundle.active`.
4. If a power loss or process crash occurs mid-write, the existing `bundle.active` remains completely untouched and uncorrupted.

### Fallback Rendering Hierarchy

When mounting a patchable screen, `EjenixPatchView` executes a three-tier fallback sequence:
1. **Cached Patch**: Attempts to load and verify the active patch stored on disk.
2. **Bundled Fallback Asset**: If no valid cached patch exists, loads the pre-compiled bundle shipped inside the app binary assets (`assets/screen.bundle`).
3. **Native Host Fallback (`fallbackBuilder`)**: If no valid patch asset exists or if execution fails, renders the native Dart widget tree provided by `fallbackBuilder`.

### Crash-Loop Rollback & Quarantine

To automatically recover from patches that crash or loop on launch:
- **Crash Counter**: The state file tracks consecutive execution failures for the current bundle ID.
- **Threshold**: If a patch fails 3 consecutive times without completing its initial mount/render cycle, the loader marks the bundle ID as **Quarantined**.
- **Self-Healing Action**: The loader automatically purges the quarantined bundle, rolls back local state to the previous active bundle ID (or native fallback), and logs status `rolledBack`.
- **Quarantine Persistence**: Quarantined bundle IDs are stored permanently on device so the control plane cannot re-stage the same broken bundle ID on subsequent network fetches.

### Freshness Protection (Monotonic Generation Numbers)

To protect against replay attacks (where an attacker or stale server re-serves an older, validly signed bundle):
- Every bundle carries a monotonic `generation` integer (e.g. `42`).
- The device state file records `highestAcceptedGeneration`.
- If a fetched bundle has `generation < highestAcceptedGeneration`, the loader rejects it immediately with status `rejected`.

---

## 6. Control Plane Server & Staged Rollout (`packages/server`)

The control plane is an AOT-compiled HTTP REST server providing bundle storage, channel configuration, staged rollout calculation, dashboard UI, and metrics.

### Keying & Multi-Tenant Channels

The control plane organizes releases under three keys: `(appId, channel, env)`.
- `appId`: Unique application identifier (e.g. `com.acme.shop`).
- `channel`: Screen or surface name (e.g. `home`, `checkout`, `profile`).
- `env`: Environment target (e.g. `staging`, `production`, `review`).

Each channel maintains an active bundle pointer and a previous bundle pointer (enabling single-command `ejenix rollback`).

### Salt-Based Local Install Staged Rollout Algorithm

Ejenix supports privacy-preserving staged rollout (e.g., `--rollout 5` for a 5% canary rollout):

```
                        Server publishes:
                  { rollout: 5%, salt: "s3cr3t" }
                                │
                                ▼
                       Client Device Hash
           sha256( local_install_id + salt ) mod 100
                                │
                 ┌──────────────┴──────────────┐
                 ▼                             ▼
            Result < 5                    Result >= 5
                 │                             │
                 ▼                             ▼
       [ Eligible for Patch ]      [ Keep Existing Patch ]
```

- **Zero Telemetry**: Devices do not register their IDs with the server.
- **Monotonicity**: Increasing the rollout percentage (e.g. 5% -> 25%) uses the exact same salt. Devices already eligible at 5% remain eligible at 25%, guaranteeing no device loses a patch during widening.
- **Uniform Distribution**: SHA-256 hashing ensures even percentage distribution across the install base.

---

## 7. Flutter Bridge & Capability Codegen (`flutter_bridge`, `packages/cli`)

The bridge connects the sandboxed bytecode interpreter to host Flutter widgets and native services.

### Capability Sandboxing Model

A patch cannot execute native code directly; it can only invoke registered host selectors exposed via `HostRegistry`.
- **Standard Flutter Surface**: `flutter_bridge` pre-registers 191 standard Flutter widgets and framework bindings (e.g., `Scaffold`, `Text`, `Column`, `ListView`, `Container`, `GestureDetector`, `Colors`, `EdgeInsets`).
- **App Capabilities (`extend`)**: Host applications expose custom widgets, state repositories, and services using `@Patchable` annotations.

### `@Patchable` Annotation & Codegen Engine (`ejenix gen`)

```
   Host Dart Code (@Patchable)
           │
           ▼
     ejenix gen
           │
     ┌─────┴─────────────────────────────────────────┐
     ▼                                               ▼
patch_sdk/app.dart                       lib/app_capabilities.g.dart
(Typed Stubs for Patches)                (Host Registration Bindings)
```

1. **Host Definition**:
   ```dart
   class ProductRepo {
     @Patchable('App.getProducts')
     List<Product> fetch() => ...;
   }
   ```
2. **Codegen Output**:
   - `patch_sdk/app.dart`: Contains `external` typed signatures used by patch developers during compilation.
   - `lib/app_capabilities.g.dart`: Contains `appCapabilities(...)` which registers selector handlers in `HostRegistry`.

### In-Place UI Swapping

`EjenixPatchView` embeds as a standard Flutter `StatefulWidget`. When a new patch is staged:
1. The loader emits an updated module instance.
2. `EjenixPatchView` triggers Flutter's `setState()`.
3. The widget tree rebuilds the patch subtree instantly in memory.
4. The user experiences an immediate UI update without app reboots or lost navigation context.
