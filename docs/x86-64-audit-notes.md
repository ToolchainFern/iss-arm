# x86-64 surface audit notes (provisional)

> **Status: PRELIMINARY — partial inventory, not a complete scope.**
> Read-only audit of upstream `HayatoG/iss_pc` at commit `37c05ba` (this fork's
> fork point), run 2026-11-01 with local analysis tooling on a contributor's
> Mac. No game files, dumps, keys, or firmware were involved. The SIMD/MSR
> sweep is unfinished and the F16C site count below is unverified — do **not**
> use this as a complete translation-layer scope estimate.

## Method

Cloned upstream at `37c05ba` and audited `port/src` — the loader plus the
reimplemented PS4 runtime in C. Catalogued every x86-64-specific site found:
inline assembly, CPUID/RDTSC/`arch_prctl`, signal-context handling, TLS,
stack/fiber manipulation, SIMD, and architecture guards.

## Inventory

| File and line | Category | Purpose | ARM64 implication |
|---|---|---|---|
| probe.c:24 | Architecture guard | Requires x86-64 and a GNU-compatible compiler. | Must reimplement or replace the guarded loader path. |
| probe.c:149 | Assembly / stack | `enter_on_stack` switches the host stack and calls the guest entry using x86-64 registers and calling convention. | Must reimplement guest entry and host/guest transition. |
| probe.c:130 | Generated x86 code | Emits an import-tracing stub that saves registers and XMM state, calls a logger, then jumps to its target. | Must reimplement instrumentation and guest/native transitions. |
| probe.c:224, :282–339 | Exception and signal state | Maps x86 GPRs, instruction pointer and XMM state from Windows or Linux fault contexts into the instruction emulator, then writes state back. | Must reimplement against translator-owned guest state; host ARM signal state is not guest x86 state. |
| probe.c:299–300, :1384–1410 | Red zone | Diagnostic clobbering and policy for rewriting guest red-zone use. | Preserve guest SysV red-zone semantics; approach depends on translator. |
| probe.c:355, :363–458 | Signal recovery / diagnostics | Recovers from guest faults and reports x86 instruction, stack, frame and argument registers. | Must adapt recovery and diagnostics to guest state. |
| probe.c:1465–1476 | Signal handling | Installs fault, trap, diagnostic and watchdog handlers. | Portable with care; coordinate host faults with translated guest state. |
| probe.c:1695 | Instruction rewrite | Applies the ISA rewrite table to the loaded image. | Must replace or integrate with translation; emitted stubs are x86 code. |
| probe.c:1774 | Assembly | Loads an x87 control word. | Must model the guest FP environment rather than set an ARM host x87 register. |
| runtime_thread.c:1–8, :80–96 | TLS / arch_prctl | Uses a FreeBSD-style guest TCB; Linux places its base in GS, Windows uses a TEB TLS slot. Guest FS loads are rewritten to GS. | Must provide per-guest-thread TLS-base semantics, independent of host TLS. |
| runtime_thread.c:517–550, :604 | Stack / FP state | Uses setjmp/longjmp for thread exit; saves/restores MXCSR and x87 control state. | Must preserve guest thread exit and FP-control behavior across host transitions. |
| runtime_gr2.c:119–212 | Fibers | Linux fibers use getcontext/makecontext/swapcontext with a guest entry and guest-provided stack. | Must reimplement or adapt fiber switching so guest CPU state and guest stacks switch together. |
| runtime_gr2.c:216–303 | Fibers | Windows implementation uses Win32 fibers. | Portable with care as a host mechanism; guest CPU state still needs explicit handling. |
| compat_win.c:14–35 | Assembly / nonlocal jump | Saves/restores Win64 callee-saved GPRs, XMM6–15, MXCSR and x87 control. | Windows x86 implementation is irrelevant on ARM, but equivalent guest recovery semantics need implementation. |
| runtime_freeze.c:204–205, :415–419 | Signal context | Captures RIP, RSP, RBP from a Linux signal context for freeze diagnostics. | Must read translated guest state for meaningful guest reports. |
| runtime_kernel.c:88–111 | TSC | Reads and calibrates host rdtsc; comments say guest direct reads must agree with the runtime clock. | Must supply a coherent guest TSC to both translated instructions and runtime calls. |
| cpu_check.c:2–28 | CPUID | Reads leaves 1, 7, 0x80000001; checks x86 feature bits incl. rewriteable instruction sets. | Must replace host CPUID checks with a guest feature policy matching translator capability. |
| isa_emulate.c:151–158, :211 | F16C / FP | Implements F16C conversion; reads MXCSR for rounding and denormal behavior. | Conversion logic reusable with care; guest MXCSR must come from guest state. |
| isa_rewrite.c:324–377, :550 | Emitted SIMD / rewrite | Generates x86 AVX stubs for F16C; explicitly excludes those sites from the INT3 trap path. | Must translate or replace the generated stubs; the existing stub assumes an x86 host with AVX. |
| runtime_memory.c:91, :762, :880–881; runtime_gr2.c:131, :227 | Host TLS | `__thread` for runtime bookkeeping, fault recovery, region cache, fiber state. | Portable with care; these alone do not imply guest FS-base semantics. |
| runtime.h:27; explicit sysv_abi / ms_abi in hook files | ABI | Calls and callbacks depend on x86 calling conventions. | Must bridge guest calls, callbacks, arguments, return values, preserved state. |
| iss_camera_pitch_hook.h:37 | Arch conditional / ABI | Uses ms_abi and x86 `target("no-avx,no-avx2")` attribute. | Must reimplement or remove x86-specific hook compilation and calling assumptions. |

No `_mm*` intrinsic calls were found by the targeted search in `port/src` — that
is **not** a conclusion that SIMD is absent. Generated byte sequences and
assembly explicitly use XMM/YMM instructions. A complete SIMD / direct-MSR
inventory was not finished.

## F16C interception precedent

At startup, CPU feature detection plus the `TLG_ISA_REWRITE` policy select
instruction sets to rewrite (probe.c:1384 onward); at probe.c:1695 the loader
applies the matching ISA table to the loaded image. For F16C sites,
isa_rewrite.c:324 emits an x86 stub that steps below the red zone, saves guest
GPRs and flags, copies the relevant YMM source into a buffer, calls
`isa_emulate_f16c`, restores destination and preserved state, and returns to
guest code. The software conversion honors guest MXCSR rounding and denormal
settings. These sites bypass the INT3 fallback (isa_rewrite.c:550: they are at
least five bytes and YMM state is not in that trap frame). Useful interception
precedent — but the emitted stub itself assumes an x86 host with AVX; it is not
an ARM translation layer. (The "13 sites" count was not independently verified.)

## Provisional `guest_cpu` boundary

Justified by the inspected code; function signatures still open:

- Guest entry, guest-to-host calls, host-to-guest callbacks (guest ABI + stack).
- Read/write guest GPRs, instruction pointer, flags; vector and FP-control state
  where traps, diagnostics, or emulation require it.
- Per-guest-thread TLS-base initialization and access.
- Guest thread and fiber state creation, switching, nonlocal exit.
- Associate host faults with guest state; resume or report a guest fault.
- Consistent guest TSC and advertised CPU-feature policy.
- Loader registration of executable guest regions and import/hook interception points.

Open design questions: exact signatures; whether signal-context translation is
needed at all; whether the existing ISA rewrite table is retained.

## Preliminary risk ranking

| # | Path A: dynamic translation | Path B: static recompilation |
|---|---|---|
| 1 | Guest/host ABI transitions and callbacks | Discovering all executable guest entry points and indirect targets |
| 2 | Guest faults vs host signal mapping (host PC ≠ guest RIP) | Indirect control flow and callbacks must reach recompiled targets |
| 3 | Per-thread guest TLS and rewritten FS/GS access | Guest TLS semantics in compiled host code and callbacks |
| 4 | Fiber switching with guest register + stack state | Fiber switching preserving recompiled guest execution state |
| 5 | Coherent direct guest TSC reads and runtime clock | Finding and replacing every relevant TSC instruction consistently |
| 6 | Guest FP control, rounding, vector state | Accurate x87/MXCSR/vector semantics across compiled code |
| 7 | Loader patches and generated x86 stubs | Existing patches and emitted stubs can't execute as ARM code |
| 8 | Red-zone preservation across host calls and faults | Recompiled stack layout preserving observable guest stack behavior |
| 9 | CPUID feature policy matching actual translation support | Feature policy matching actually recompiled instructions |
| 10 | Nonlocal exits across translated and native frames | Nonlocal exits/diagnostics retaining guest state and control flow |

## Open items

- Complete the SIMD intrinsic and direct-MSR inventory.
- Verify the F16C site count against the rewrite table.
- Settle `guest_cpu` function signatures; decide on signal-context translation;
  decide the fate of the ISA rewrite table under translation.
