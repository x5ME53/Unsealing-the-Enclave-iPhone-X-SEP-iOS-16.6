# Unsealing the Enclave — iPhone X SEP, iOS 16.6

Reverse engineering the Secure Enclave Processor (SEP) firmware from the
**iPhone X (A11), iOS 16.6, build 20G75**. This repo is a public research
notebook: setup guides anyone can reproduce, component-by-component notes, and
findings as they accumulate.

I'm documenting this as I learn, so expect the notes to grow, get corrected, and
occasionally be wrong-then-fixed. If that's useful to you, great — that's the
point of keeping it public.

> **Goal:** learn iOS/SEP reverse engineering thoroughly enough to do real
> vulnerability research and coordinated disclosure / bug bounty work. This is
> a learning log, not a dump of novel exploits.

---

## What the SEP is

The Secure Enclave Processor is a physically separate coprocessor inside the
SoC, with its own CPU, ROM, RAM, and OS (sepOS). The main application processor
running iOS cannot read its memory; the two communicate only through a
constrained **mailbox** message interface. The SEP holds the secrets that must
survive a full iOS compromise: filesystem encryption keys, biometric templates,
and passcode / anti-bruteforce enforcement.

## Why this target

SEP firmware ships encrypted, with the payload key wrapped by the SoC's hardware
GID key. That key is only recoverable on devices vulnerable to the **checkm8**
bootrom exploit — **A11 and earlier**. The iPhone X (A11) is therefore the most
recent iPhone with publicly decryptable SEP firmware, and its layout still maps
onto the foundational public research, which makes it a strong target for
learning.

---

## Repo layout

| Path | What's in it |
| --- | --- |
| [`docs/`](docs/) | Reproducible guides (extraction, tooling, Ghidra setup) |
| [`notes/`](notes/) | Component-by-component reverse engineering notes |
| [`findings/`](findings/) | Write-ups of specific behaviours / bugs / curiosities |
| [`scripts/`](scripts/) | Helper scripts (Ghidra scripts, automation) |
| `log.md` | Dated research log — the running journal |

*(Folders get created as content lands; not all exist yet.)*

---

## Guides

- **[Extracting and Loading SEP Firmware into Ghidra](docs/extracting-sep-firmware.md)**
  — full reproducible pipeline: IPSW → decrypt → split → Ghidra. Start here if
  you want to follow along on your own device.

## Notes

- `notes/00-boot.md` — the SEP bootloader *(planned)*
- `notes/01-kernel.md` — sepOS microkernel: entry point, init chain, mailbox
  *(in progress)*
- `notes/02-sepos.md` — the SEPOS app layer *(planned)*

## Findings

- *(nothing here yet — the honest state of a new research repo)*

---

## Current status

- [x] Firmware extracted, decrypted, and split (A11 / 20G75)
- [x] Kernel module loaded into Ghidra, auto-analysis clean
- [x] Entry point identified (`Reset`): interrupt masking → CPU/FPU setup →
      data init → handoff to init chain
- [ ] String pass to bootstrap function labels
- [ ] Locate and map the mailbox message handler / dispatch loop
- [ ] Reconstruct the mailbox protocol (message types + fields)
- [ ] Map SEPOS service endpoints

---

## Reproducing this

Everything here is reproducible with your own legally obtained firmware. See the
[extraction guide](docs/extracting-sep-firmware.md). In short: download the IPSW
from Apple's servers, get the firmware keys for your build from theAppleWiki,
then decrypt → split → load.

## What this repo does *not* contain

- No Apple firmware images, decrypted output, or IPSW files.
- No copied key material — keys are referenced from theAppleWiki, not hosted.

Researching firmware is fine; redistributing copyrighted firmware or key
material is a separate matter, and this repo stays on the right side of it.

---

## References

- Mandt, Solnik & Wang — *Demystifying the Secure Enclave Processor* (Black Hat
  2016). The foundational public reference.
- [theAppleWiki](https://theapplewiki.com/) — firmware keys, board/build
  mappings.
- [sepsplit-rs](https://github.com/justtryingthingsout/sepsplit-rs) — the
  splitter used here.
- [pyimg4](https://github.com/m1stadev/PyImg4) — IMG4 extraction / decryption.

---

## Disclaimer

This is independent security research for educational purposes and coordinated
disclosure. Nothing here is affiliated with or endorsed by Apple. Any
vulnerabilities found will be reported through appropriate channels before any
public detail is published.
