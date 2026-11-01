# iss-arm roadmap: native ARM64 for the Second Son PC port

Fork of [HayatoG/iss_pc](https://github.com/HayatoG/iss_pc) (upstream: playable
Windows/Linux x86-64 port of inFAMOUS Second Son, PS4 CUSA00223 + 1.07).
Upstream stays focused on x86-64 correctness and performance; this fork owns the
ARM64 question. Upstream sync: rebase/merge from `HayatoG/iss_pc` `developer`
regularly; ARM work lands here first, and anything generally useful goes back
upstream as PRs.

See [docs/ARM64.md](docs/ARM64.md) for the architecture investigation behind
this plan.

## Phase 0 — Origination (now)

- [x] Fork created as `ToolchainFern/iss-arm`, public.
- [ ] Architecture investigation published (`docs/ARM64.md`).
- [ ] Contributing guide and code-of-conduct basics (`CONTRIBUTING.md`).
- [ ] Call-for-collaborators issue pinned.
- [ ] Answer the open questions in `docs/ARM64.md` from static analysis of the
      port's loader and runtime (no game files needed for this).

## Phase 1 — Translation-layer spike

- [ ] Instrument what the loader actually executes: ISA surface used by the
      eboot + modules, indirect-branch patterns, any code patching at runtime.
- [ ] Prototype Path A (dynamic binary translation) targeting ARM64 Linux —
      the fastest route to a running experiment on contributor hardware.
- [ ] Spike Path B (static recompilation) far enough to judge feasibility for
      the iOS endgame.
- [ ] Decision record: which path(s) the fork commits to, with measured data.

## Phase 2 — Platform shells

- [ ] Android: collaborate with [erickmendes22/iss_android](https://github.com/erickmendes22/iss_android) (FEXCore-based) rather than duplicating it — shared CPU-translation research, touch-control notes, and test data flow both ways.
- [ ] macOS (Apple Silicon): MoltenVK renderer validation; interim Rosetta 2
      build (Path C) as a stepping stone if useful. This is iss-arm's primary platform target alongside iOS.
- [ ] iOS: only behind Path B; app scaffold, signing/sandbox/asset-delivery
      plan.
- [ ] ARM64 CI for the portable parts (runtime, scripts); execution tests gated
      on contributor hardware.

## Phase 3 — Playable

- [ ] First boot to title on at least one ARM64 target.
- [ ] Performance and correctness triage, same discipline as upstream
      (per-effect fixes behind switches, like upstream's BPxx series).
- [ ] Upstream PRs for anything that improves x86-64 too.

## Non-goals

- No game files, keys, firmware, or dumps in this repo — ever. Contributors use
  their own legally obtained copies.
- No promises on timelines. This is research-first; the translation layer is a
  genuinely hard problem and the maintainers are honest about that.
- No competing with upstream on x86-64. If upstream ever takes ARM64 on, this
  fork merges back and closes.
