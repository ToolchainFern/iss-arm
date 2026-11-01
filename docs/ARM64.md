# ARM64 native support — architecture investigation

**Status:** research document, v0.1 (2026-10-31). Based on upstream
[HayatoG/iss_pc](https://github.com/HayatoG/iss_pc) README and `docs/ROADMAP.md`
as of 2026-10-31. This fork (`ToolchainFern/iss-arm`) exists to investigate and
implement native ARM64 execution for the iss_pc core. Nothing here ships game
code, assets, keys, or firmware — contributors use their own dumps
(CUSA00223 base 1.00 + update 1.07, ~33 GB).

## How iss_pc works today (verified from upstream docs)

- **Loader:** `port/src/probe.c` loads the PS4 `eboot.bin`; `port/scripts/prepare.py`
  and `link_modules.py` prepare and link the eboot with the game's modules and
  the reimplemented console modules.
- **Runtime:** `port/src/runtime_*.c` reimplements the PS4 system libraries in C;
  `runtime_infamous.c` holds the 15 imports specific to this game. TLS is handled
  (`mov reg, fs:[0]` works in any register).
- **CPU execution:** the game's original **x86-64 code runs directly on the host
  processor** — no CPU emulator. An AVX-capable CPU is required. Notably, the
  codebase already emulates 13 half-precision conversions (`VCVTPH2PS`/`VCVTPS2PH`)
  at startup on CPUs without F16C (ROADMAP item, #51) — so there is precedent for
  instruction-level emulation hooks at load time.
- **GPU:** translated to Vulkan via a shadPS4-derived core plus a shader
  recompiler. Known sensitive areas: wave64 ballot lowering on 32-lane GPUs,
  shared-memory (LDS) barriers, `ReadFirstLane` semantics, float atomics on
  NVIDIA.
- **Input/audio:** SDL3 controllers (DualSense/DualShock 4/Xbox/8BitDo), motion
  sensor emulation, own AudioOut implementation (no firmware files).
- **Launcher:** Qt Quick (default) + Tk fallback; not portable to mobile as-is.

## The ARM64 problem

Every ARM64 target in scope — iPhone/iPad (iOS), Apple Silicon Macs, and
virtually all Android devices — cannot execute the game's x86-64 code natively.
The port's founding assumption ("the game's original code runs natively on the
processor") does not hold there. ARM support therefore requires a **CPU
translation layer**, which is the central engineering problem of this fork. The
rest of the port (C runtime, SDL3, Vulkan renderer) is portable in principle.

## Candidate paths

### Path A — Dynamic binary translation (user-mode, Box64-style)

Translate x86-64 → ARM64 at load/runtime inside the port process, à la Box64
(which already runs x86-64 Linux binaries on ARM64 Linux).

- Pros: no per-game ahead-of-time step; the existing `prepare.py`/`link_modules.py`
  flow is largely untouched; contributors can prototype on ARM64 Linux today.
- Cons: translation overhead on a game that is already CPU-bound (20–30 FPS on an
  RTX 2060); the port's custom ELF loading, TLS handling, and signal usage must
  survive under the translator; **iOS is blocked** — no JIT/RWX pages for App
  Store apps, so DBT is not shippable on iOS (sideloaded JIT experiments are
  research-only).
- Best fit: Android and ARM64 Linux prototyping.

### Path B — Static (ahead-of-time) recompilation

Translate the game's x86-64 binary to ARM64 once, ship ARM64 code. Precedent in
this scene: the Bloodborne Android rebuild tool reportedly recompiles the PS4
game to native ARM64.

- Pros: the only shippable path for **iOS**; no runtime translation overhead;
  the translated binary is inspectable and debuggable.
- Cons: the hard problems are real — indirect branches, data/code separation,
  any self-modifying code, and the 15 game-specific imports plus the PS4 ABI
  surface must be handled; a static translator for this eboot is a project of
  its own.
- Best fit: iOS, and long-term Android.

### Path C — Interim macOS via Rosetta 2 (not native, documented for honesty)

Build the existing x86-64 port for macOS x86-64 and run it under Rosetta 2 on
Apple Silicon, with Vulkan routed through MoltenVK.

- Pros: smallest possible step; validates the MoltenVK renderer path early.
- Cons: not native ARM; translation overhead; still needs the macOS platform
  shell. Treated as a stepping stone, not the goal.

## Platform notes

| Target | CPU translation | GPU | Platform shell | Notes |
|---|---|---|---|---|
| Android (ARM64) | Path A or B — see sibling [erickmendes22/iss_android](https://github.com/erickmendes22/iss_android) (FEXCore-based); we collaborate, not duplicate | Vulkan native | New: touch controls, scoped storage, app packaging | Largest tester base; Path A prototype feasible now |
| macOS (Apple Silicon) | Path A, B, or interim C | Vulkan → MoltenVK | macOS build of runtime + launcher | Watch: wave64/subgroup lowering vs Apple 32-wide SIMD groups; MoltenVK feature coverage vs the port's Vulkan 1.3 usage |
| iOS | **Path B only** (no JIT) | Vulkan → MoltenVK | New: touch controls, sandbox, signing, ~33 GB asset delivery | Hardest target; static recomp is the gating item |
| ARM64 Linux | Path A | Vulkan native | Existing scripts mostly reusable | Best contributor testbed |

## What is already portable

- The C runtime reimplementing PS4 libraries (`runtime_*.c`) — compiles for
  ARM64 in principle.
- SDL3 input (ships iOS/Android/macOS backends).
- Save system, config, logging — platform-agnostic.
- The shader recompiler's *logic*; its *output* (SPIR-V → via MoltenVK to MSL on
  Apple) needs validation, especially subgroup/wavefront assumptions.

## What needs building

1. CPU translation layer (Path A and/or B) — the core of this fork.
2. Platform shells: Android app, macOS app, iOS app (packaging, file access,
   lifecycle).
3. Touch control layout (camera, spray/aim, touchpad gestures) for mobile.
4. MoltenVK validation for the Vulkan renderer on Apple GPUs.
5. ARM64 CI (build the portable parts; execution tests need contributor hardware
   — the fork maintainers have no ARM test devices).

## Open questions

- Does the eboot use self-modifying code or runtime code generation? (Decides
  how painful Path B is.)
- How much of the game's x86-64 ISA surface is actually exercised? (Bounds the
  translator's required coverage; the F16C precedent suggests the real surface
  may be smaller than the ISA.)
- Can the 15 game-specific imports and the PS4 ABI surface be satisfied
  identically under translation?
- What is the measured overhead of Path A on an ARM64 target for this
  workload? (Unknown until a prototype runs on contributor hardware.)

## Contributing

See [CONTRIBUTING.md](../CONTRIBUTING.md). CPU-emulation experience (Box64, FEX,
dynarmic), Apple-GPU/MoltenVK experience, and Android packaging experience are
the scarcest skills here. ARM64 hardware owners willing to run test builds are
equally valuable — the maintainers currently have no ARM test devices.
