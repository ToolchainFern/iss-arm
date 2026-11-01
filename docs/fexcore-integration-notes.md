# FEXCore integration — research notes

Companion to `docs/ANDROID_PORT.md` (steps 1–2: the `guest_cpu` interface and
FEXCore integration). These are **research notes, not a tested integration** —
every claim below is either verified from FEX-Emu's own documentation or
explicitly marked as something to verify against the pinned FEXCore version.
Corrections welcome; that's the point of the doc.

## What FEXCore is (verified)

- FEXCore is the emulation library inside [FEX-Emu](https://fex-emu.com), an
  x86/x86-64 → ARM64 binary translator for Linux. It is built to be embedded:
  "portable library implementation in order to support easy integration into
  applications" (FEXCore readme).
- Translation is JIT through a custom IR with a tiered recompiler; it supports
  "all modern extensions of the x86(-64) instruction set, including AVX/AVX2"
  (fex-emu.com). Target host is AArch64, ARMv8.1 or newer.
- Relevant subsystems for this port:
  - **SignalDelegator** — translates between host and guest signal contexts.
  - **SyscallHandler** — translates the guest syscall ABI to the host kernel.
  - **Thunking** — lets guest code call *host* libraries (used for OpenGL/Vulkan
    forwarding, which matters for the renderer bridge in step 3).
  - **Per-app configuration** — performance/behavior tweaks per game (e.g.
    skipping costly memory-model emulation where safe).
- FEXCore can be built as a shared or static library; the project pins FEXCore
  versions in CI, so "versão fixada" (as the plan requires) matches how upstream
  FEX itself works.

## Blocker mapping (from ANDROID_PORT.md)

| Blocker in this codebase | FEXCore integration angle |
|---|---|
| `port/src/probe.c` requires x86-64, touches x86 registers/asm | This is where the `guest_cpu` interface boundary goes: keep loader/probing logic native, hand guest code ranges to FEXCore for execution. |
| `port/src/runtime_thread.c` uses `arch_prctl`/`ARCH_SET_GS` and x87 state | Guest thread state (including GS base for TLS) becomes FEXCore-managed. **To verify:** how the port's TLS expectations map onto FEXCore's thread context — likely the largest single integration question. |
| Signal handling / memory-fault diagnostics assume x86 contexts | Route through FEXCore's SignalDelegator; crash diagnostics must read the *guest* context via FEXCore rather than the raw `ucontext`. |
| Fibers (`getcontext`/`makecontext`/`swapcontext`) in `port/src/runtime_gr2.c` | Audit under FEXCore; if the fiber switch confuses the JIT's thread state, consider replacing with FEXCore-aware context handling. **To verify** on the pinned version. |
| `port/gpu/CMakeLists.txt` requires X11 outside Windows | Orthogonal to CPU work (step 3 of the plan); note FEXCore's thunking can forward Vulkan calls to host libraries, which may simplify the Android presenter bridge later. |
| No `guest_cpu`/FEXCore integration in the base | Propose: introduce the interface first with the existing x86-64 path behind it (plan step 1), so the PC build keeps working while the FEXCore backend lands incrementally. |

## Suggested version-pinning practice

- Pin the FEXCore commit in the build (as the plan says, "versão fixada") and
  record it next to the iss_pc base commit (`1898d449…`) already tracked in
  ANDROID_PORT.md, so any FEXCore-side regression is bisectable.
- Use FEXCore's per-app config for this game only — don't change global FEX
  behavior for the other ports in this family.

## Open questions (for the maintainer)

1. TLS/GS mapping — is there prior art in the GoW III / Bloodborne ports for
   how `ARCH_SET_GS`-style expectations were handled under FEXCore?
2. Does the game's x86-64 code use any instructions outside FEXCore's covered
   set? (The iss_pc base already emulates F16C conversions at startup — worth
   checking whether FEXCore covers those natively.)
3. Signal-based crash diagnostics: keep the port's own handler and translate,
   or defer entirely to FEXCore's delegator?

## Sources

- https://fex-emu.com (project homepage; AVX/IR/per-app-config claims)
- FEXCore readme (github.com/thebcd69/bcd-fex mirror of FEXCore/Readme.md;
  "portable library", ARMv8.1+ target)
- DeepWiki FEX-Emu pages: overview (IR/tiered recompiler), system integration
  (SignalDelegator, thunking)
