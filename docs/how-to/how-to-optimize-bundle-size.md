# How to Optimize Ejenix Patch Bundle Size

This guide provides strategies for minimizing Ejenix bytecode bundle sizes and optimizing network transfer bandwidth across client devices.

---

## 1. Bundle Size Basics

Ejenix compiles Dart source code into a compact CBOR bundle containing:
- Envelope header & Ed25519 signature (68 bytes)
- Indexed Constant Pool (strings, type descriptors, selector names)
- Class layouts & member descriptors
- Bytecode opcodes & register frames

*Typical full bundle size*: **600 bytes – 1.5 KB** for standard UI screens.

---

## 2. Strategies for Size Optimization

### Strategy 1: Use Host Capability Widgets Rather Than Custom Interpreted Implementations

Widgets rendered via registered host capabilities (`HostRegistry`) execute as native compiled Dart and take zero bytecode space in the patch.

```dart
// ❌ Custom interpreted widget logic (Larger bytecode bundle):
Widget buildCustomButton(String label, VoidCallback onPressed) {
  return Container(
    padding: EdgeInsets.all(12),
    decoration: BoxDecoration(color: Colors.blue, borderRadius: BorderRadius.circular(8)),
    child: GestureDetector(
      onTap: onPressed,
      child: Text(label, style: TextStyle(color: Colors.white, fontWeight: FontWeight.bold)),
    ),
  );
}

// ✅ Host capability widget (Tiny bytecode footprint):
// Expose `PrimaryButton` via `@Patchable` in host binary and call it in patch:
Widget buildCustomButton(String label, VoidCallback onPressed) {
  return PrimaryButton(label: label, onPressed: onPressed);
}
```

### Strategy 2: Share Constant Pool Identifiers

The constant pool deduplicates strings and selector symbols. Using consistent naming conventions across patch functions reduces constant pool overhead.

### Strategy 3: Leverage Binary Delta Updates (`ejenix build-delta`)

For patch updates to existing screens, ship a binary delta patch instead of a full bundle.

```bash
ejenix build-delta \
  --base base_screen_v1.bundle \
  patches/home_screen_v2.dart \
  -o patch_v2.delta \
  --signing-key release.key \
  --app-id com.acme.shop
```

*Comparison*:
- Full Bundle `v2`: **850 bytes**
- Binary Delta `v1 -> v2`: **140 bytes** (83% bandwidth reduction)

---

## 3. Measuring & Auditing Bundle Sizes

Audit bundle sizes and constant pool entries using `ejenix inspect`:

```bash
ejenix inspect home.bundle
```

*Example Output*:
```
Bundle Header:
  Magic: EJNX (v1)
  App ID: com.acme.shop
  Channel: home
  Generation: 2
  Min SDK: 1.0.0
  Body Size: 642 bytes
  Signature: Valid (64 bytes)

Constant Pool (14 entries):
  [0] "com.acme.shop"
  [1] "home"
  [2] "App.fetchProducts"
  [3] "PrimaryButton"

Classes (1):
  HomeScreenState

Functions (3):
  main() -> 42 opcodes, 8 registers
  build() -> 86 opcodes, 12 registers
```
