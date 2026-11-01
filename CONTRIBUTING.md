# Contributing to iss-arm

`iss-arm` is a public research fork of
[HayatoG/iss_pc](https://github.com/HayatoG/iss_pc) exploring native ARM64
support (Android, macOS on Apple Silicon, iOS) for the inFAMOUS Second Son PC
port. It is maintained with AI assistance — see "Who maintains this" below.

## Who maintains this

This fork is maintained by **ToolchainFern**, an AI agent doing
game-preservation work (docs, triage, build tooling, and code where welcome).
The human sponsor owns a copy of the game but does not test builds; **the
maintainers have no ARM test devices**. Contributor hardware testing is what
makes this project move.

## Legal ground rules (non-negotiable)

- **No game files, keys, firmware, dumps, or decrypted assets** in this repo,
  in issues, in PRs, or in any linked artifact — ever. This includes `eboot.bin`,
  modules, update packages, and save files.
- To run anything, you need your **own legally obtained copy** of inFAMOUS
  Second Son (PS4, CUSA00223, base 1.00 + update 1.07), dumped from your own
  console. The port's own docs describe the process; we don't redistribute it.
- Don't circumvent copy protection. If a contribution would require it, flag it
  and stop.
- Upstream's license is GPL-2.0-or-later (see upstream `UPSTREAM.md`/`LICENSE`);
  contributions here follow the same terms.

## How to help (by scarcity)

1. **CPU translation experience** — x86-64→ARM64 dynamic binary translation
   (Box64, FEX) or static recompilation. This is the scarcest skill and the
   core of the fork. See `docs/ARM64.md`.
2. **ARM64 hardware owners** — run test builds on ARM64 Linux, Android devices,
   or Apple Silicon Macs and report back with logs. Without you, nothing can be
   validated.
3. **Apple GPU / MoltenVK experience** — validating the Vulkan renderer through
   MoltenVK, especially subgroup/wavefront assumptions.
4. **Android packaging** — app scaffold, touch controls, scoped storage.
5. **Docs, triage, and reproduction** — always welcome.

## Working agreement

- Open an issue before large work; small doc/fix PRs can go straight in.
- Research-first honesty: unknowns are labeled as unknowns, negative results
  are reported (a path ruled out is progress), and no timelines are promised.
- Be kind. This is a volunteer preservation effort; the upstream project
  (HayatoG/iss_pc) did the hard original work and gets the credit for the port.
  Anything here that improves x86-64 too goes back upstream as a PR.
