# Local adapter patch

Source: `joakimeriksson/fm1-emu`, exact revision
`ebd6de3bbd303bf5011fa1c3d669144e87f03c27`, `crates/fm1-core`.
The upstream MIT licence is retained in `LICENSE`.

This branch changes only `crates/fm1-core/src/lib.rs` and `src/jit.rs`, plus this note. The workspace manifest and sibling crate dependencies remain upstream-owned.

- `src/lib.rs`: eager fallible native-cache initialization and an actual cache-ownership query. Interpreter semantics and native code generation are unchanged.
- `src/jit.rs`: on macOS only, lease sixteen isolated 64 MiB CPU caches from one process MAP_JIT region and return leases on destruction. No machine state is shared.

The 1 GiB arena is a virtual address reservation, not a claim about resident
memory. The process retains the arena mapping; used pages/cache capacity are
not automatically decommitted when a slot is returned. Sixteen simultaneous
CPU caches fit; exhaustion is an explicit backend-unavailable error. A dual
CPU firmware normally needs two caches per instance. A transactional replacement
temporarily needs another cache while preparing its new CPU.

The original core silently interpreted when executable mapping failed and did
not release its mapping on CPU destruction. Those semantics cannot implement
the wrapper's explicit native-backend contract or repeated hosted reloads.
Apple documents a single MAP_JIT region under Hardened Runtime; the arena also
supports independent instances under that restriction. A mutex only protects
slot allocation/return, outside emulation and code emission. Code generation
and execution retain upstream's thread-local write-protection handling.

The C adapter rejects `com.apple.security.cs.jit-write-allowlist`, which disables
the `pthread_jit_write_protect_np` API used by this pinned implementation.
Hardened Runtime needs `com.apple.security.cs.allow-jit`; App Sandbox and
Hardened Runtime are separate features. This adapter does not implement an iOS
JIT path.

Primary platform references:

- [Porting just-in-time compilers to Apple silicon](https://developer.apple.com/documentation/apple-silicon/porting-just-in-time-compilers-to-apple-silicon)
- [Allow execution of JIT-compiled code entitlement](https://developer.apple.com/documentation/bundleresources/entitlements/com.apple.security.cs.allow-jit)

Tests cover compiled execution/PCM equivalence with the interpreter, sixteen
simultaneous caches, a clean seventeenth allocation failure, returned-slot reuse,
actual secondary-CPU release, and repeated prepare/reboot replacement.
