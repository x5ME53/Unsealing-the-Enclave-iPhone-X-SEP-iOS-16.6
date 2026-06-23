# Extracting and Loading SEP Firmware into Ghidra

A reproducible, beginner-followable guide to getting Apple's Secure Enclave
Processor (SEP) firmware out of an IPSW, decrypted, split into its individual
Mach-O modules, and loaded into Ghidra for analysis.

This was written against a **decryptable A11 target (iPhone X, iOS 16.6, build
20G75)** on macOS (Apple Silicon). The same pipeline applies to other firmware,
but decryption only works where firmware keys are publicly available — see
[Why A11](#why-a11-and-not-something-newer).

> **Scope / legality note.** This documents *analysis* of firmware you can
> obtain from Apple's own servers. It does **not** redistribute Apple firmware,
> decryption keys, or any decrypted output. You download the IPSW yourself and
> reference published keys from theAppleWiki. Researching firmware is fine;
> redistributing copyrighted firmware images is a separate matter — don't commit
> them to your repo.

---

## Background: what the SEP is

The Secure Enclave Processor is a physically separate coprocessor inside the
SoC. It has its own CPU, ROM, RAM, and operating system (sepOS), and the main
application processor (the one running iOS) cannot read its memory. The two
sides communicate only through a constrained message-passing **mailbox**
interface.

The SEP exists so that even a fully compromised iOS cannot reach certain
secrets:

- Data Protection keys (the AES keys that encrypt the filesystem)
- Biometric templates (Face ID / Touch ID)
- Passcode verification and anti-bruteforce enforcement (the "wipe after N
  attempts" logic)
- Secure Boot root-of-trust measurements

The firmware image we extract here is the actual code the SEP runs.

---

## Why A11 and not something newer

SEP firmware ships **encrypted** inside the IPSW. The payload key is wrapped
with the SoC's hardware **GID key**, which never leaves the chip and is only
usable through the on-die AES engine — and that engine is only reachable by
running code at a privileged boot stage, i.e. via a bootrom exploit.

- **A11 and earlier** are vulnerable to the **checkm8** bootrom exploit. People
  with physical devices used it to unwrap SEP payload keys and publish them on
  theAppleWiki. That's why these firmwares are decryptable.
- **A12 and later** have no public bootrom exploit, so there are no published
  SEP keys. Their firmware stays encrypted and cannot be decrypted off-device.

So **iPhone X (A11) is the most recent iPhone with publicly decryptable SEP
firmware.** It's also a good teaching target because its layout still matches
the foundational public reference (see [Further reading](#further-reading)).

---

## Prerequisites

- macOS (this guide uses Apple Silicon; Intel works with minor path
  differences)
- [Homebrew](https://brew.sh/)
- Python 3 with `pip3`
- An IPSW restore file for your target device, downloaded from Apple's servers
  (e.g. via [ipsw.io](https://www.ipsw.io/) or
  [ipsw.me](https://ipsw.me/), which redirect to Apple's CDN)
- [Ghidra](https://ghidra-sre.org/) installed

---

## Step 1 — Verify your IPSW download

An IPSW is a multi-GB file; truncated or corrupted downloads are common. Verify
it against the SHA-256 the download site lists.

A hash is a fixed-length fingerprint of a file: the same input always produces
the same digest, and changing even one bit changes the digest completely. If
your file's digest matches the published one, your copy is bit-for-bit
identical to the genuine file.

```bash
cd ~/Downloads
shasum -a 256 "iPhone10,3,iPhone10,6_16.6_20G75_Restore.ipsw"
```

Compare the output to the published value. To let the shell do the comparison
for you (prints `OK` or `FAILED`):

```bash
echo "<EXPECTED_SHA256>  iPhone10,3,iPhone10,6_16.6_20G75_Restore.ipsw" | shasum -a 256 -c
```

(Note the **two spaces** between hash and filename — that's the format
`shasum -c` expects.)

---

## Step 2 — Locate and extract the SEP image

An IPSW is just a ZIP archive. List its contents and find the SEP firmware:

```bash
unzip -l "iPhone10,3,iPhone10,6_16.6_20G75_Restore.ipsw" | grep -i sep
```

You'll see entries like:

```
Firmware/all_flash/sep-firmware.d22.RELEASE.im4p
Firmware/all_flash/sep-firmware.d221.RELEASE.im4p
```

The board identifier (here `d22`) must match the key page you'll use. A device
may have multiple board revisions (`d22` / `d221`); pick the one your keys are
for. Extract it:

```bash
unzip "iPhone10,3,iPhone10,6_16.6_20G75_Restore.ipsw" \
  "Firmware/all_flash/sep-firmware.d22.RELEASE.im4p"
```

> **Tip:** filenames with commas are valid; quote them to avoid shell mangling,
> and use **Tab completion** so you never typo the name.

---

## Step 3 — Decrypt and unwrap the IM4P

The `.im4p` is an IMG4 payload container (a DER/ASN.1 wrapper around the actual
payload). On these devices the payload is **AES-encrypted**. We use
[`pyimg4`](https://github.com/m1stadev/PyImg4) to strip the wrapper, decrypt,
and decompress in one step.

```bash
pip3 install pyimg4
```

Get the **IV** and **Key** for your exact build from the corresponding
theAppleWiki key page (search for the build + device, e.g. the key page for
build `20G75` / `iPhone10,3`). **Do not commit keys to your repo — reference the
wiki page instead.**

```bash
pyimg4 im4p extract \
  --iv <IV_FROM_WIKI> \
  --key <KEY_FROM_WIKI> \
  -i Firmware/all_flash/sep-firmware.d22.RELEASE.im4p \
  -o sep.bin
```

A successful run prints something like
`[NOTE] Image4 payload data is encrypted, decrypting...`. Sanity-check the
output is no longer ciphertext:

```bash
file sep.bin
xxd sep.bin | head -4
```

If the key is correct you'll see **structure** — recognizable bytes and ASCII
markers in the header region (the SEP bundle carries its own header). If the key
is wrong (or the firmware is one of the undecryptable newer ones), you'll see
high-entropy noise instead.

> **Reference point:** an *encrypted* extract prints `is encrypted` (no
> "decrypting") and the bytes are random noise. A *decrypted* one shows
> structure. That difference is the whole reason A11 works and newer firmware
> doesn't.

---

## Step 4 — Split the bundle into Mach-O modules

The decrypted blob is **not a single Mach-O** — it's a packed bundle containing
a SEP bootloader, the sepOS kernel, and one or more SEP apps, concatenated with
a header/app table at the front. Feeding the whole thing to a disassembler gives
you one valid module and garbage after it. Use
[`sepsplit-rs`](https://github.com/justtryingthingsout/sepsplit-rs) to carve out
the individual modules.

Install Rust if you don't have it:

```bash
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
cargo --version
```

Build the tool:

```bash
cd ~/Downloads
git clone https://github.com/justtryingthingsout/sepsplit-rs
cd sepsplit-rs
cargo build --release
```

If the build complains about LLVM (it has a small C dependency for LZVN
decoding):

```bash
brew install llvm
cargo build --release
```

Run it. Usage is `sepsplit-rs <firmware> [output_folder]` — **there is no
verbose flag**, and the output folder must not already exist as a conflicting
path:

```bash
./target/release/sepsplit-rs ../sep.bin ../sep_out/
ls -la ../sep_out/
```

It prints each module's name, size, and UUID, and writes one file per module,
e.g.:

```
sepdump00_boot     — SEP bootloader
sepdump01_kernel   — sepOS microkernel
sepdump02_SEPOS    — main SEPOS app
```

> **Note on layout variation:** on some builds the individual services (SKS,
> sse, scrd, ARTM, etc.) are compiled into the SEPOS module rather than shipped
> as separate Mach-Os, so you may see only three files. That's normal.

---

## Step 5 — Load a module into Ghidra

We'll start with `sepdump01_kernel` — it's small, which makes it the easiest
place to learn the SEP's conventions.

1. Launch Ghidra → **File → New Project** → *Non-Shared Project* → name it
   (e.g. `SEP`).
2. **File → Import File** → navigate to `sep_out/` → select `sepdump01_kernel`.
   - If the dump files don't appear, set the file-type filter to **All Files**
     (they have no extension).
3. In the import dialog:
   - **Format:** likely `Raw Binary` (the splitter carves raw code without a
     Mach-O header).
   - **Language:** click `...` → filter `AARCH64` → choose
     **`AARCH64 / v8A / 64 / little / default`** (`AARCH64:LE:64:v8A:default`).
     Make sure it's **little**-endian, not big.
4. Click **OK**, then **Yes** to analyze, and accept the **default** analyzers.

### A note on base address

Because the module loads as Raw Binary at base `0x0`, absolute data-pointer
cross-references won't resolve cleanly (you'll see stray `DAT_xxxx` labels and an
"overlap" warning in the decompiler). This is expected and harmless for a first
pass:

- **Code flow is intact** — the kernel's internal branches are PC-relative and
  resolve correctly regardless of base.
- **Only absolute data xrefs are affected.** You can rebase later via
  **Window → Memory Map** once you've identified the true load address from an
  `ADRP`/`ADD` pair that should point at a known string.

This split of `sepsplit-rs` doesn't print per-module load addresses, so base
`0x0` is the pragmatic starting point.

---

## Step 6 — Orient yourself

Ghidra auto-names the entry function **`Reset`**. Reading it, you'll see the
classic arm64 kernel boot sequence:

- `msr DAIFSet, #0xf` — mask all interrupts
- `mrs x21, mpidr_el1` — read the core ID
- `mrs ..., cntpct_el0` — read the timer counter
- `msr cpacr_el1, x0` — enable FPU/SIMD access
- `mov sp, x0` — set up the stack
- a copy/zero loop (BSS/data setup), then calls into the real init routines

From here, the fastest way to make sense of a stripped binary is **strings, not
blind call-tracing**:

**Window → Defined Strings**

In a SEP kernel these are gold — especially **assertion strings**, which often
leak original source filenames and function names (e.g. a `.c` filename or an
internal function name). Each string has cross-references back to the code that
uses it, so: pick a string → see which function references it → you now have a
real name for an otherwise anonymous `FUN_xxxx`.

The highest-value target to work toward is the **mailbox handler** — the routine
that receives and dispatches messages from the application processor. It's the
SEP's attack surface (every message type is a potential vector), and it's the
spine that connects everything else. You reach it by following the init chain
out of `Reset` to where interrupt handlers and message queues get set up.

---

## Reproduce-from-scratch checklist

```bash
# 1. Verify
shasum -a 256 "<IPSW>"

# 2. Extract SEP
unzip -l "<IPSW>" | grep -i sep
unzip "<IPSW>" "Firmware/all_flash/sep-firmware.<board>.RELEASE.im4p"

# 3. Decrypt (keys from theAppleWiki for your build)
pip3 install pyimg4
pyimg4 im4p extract --iv <IV> --key <KEY> \
  -i Firmware/all_flash/sep-firmware.<board>.RELEASE.im4p -o sep.bin
file sep.bin && xxd sep.bin | head -4   # expect structure, not noise

# 4. Split
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
source "$HOME/.cargo/env"
git clone https://github.com/justtryingthingsout/sepsplit-rs
cd sepsplit-rs && cargo build --release   # brew install llvm if needed
./target/release/sepsplit-rs ../sep.bin ../sep_out/

# 5. Load sep_out/sepdump01_kernel into Ghidra as
#    Raw Binary, AARCH64:LE:64:v8A:default, analyze with defaults
```

---

## Further reading

- **Mandt, Solnik & Wang — "Demystifying the Secure Enclave Processor"**
  (Black Hat 2016). The foundational public reference; its description of the
  boot flow, mailbox protocol, memory layout, and app model maps well onto
  A11-era images.
- **theAppleWiki** — firmware key pages, device/board identifiers, build
  mappings.
- **checkm8 / ipwndfu** — background on how A11-and-earlier firmware keys were
  recovered in the first place.

---

## What's intentionally not here

- No firmware images, decrypted output, or IPSW files.
- No copied key material — keys are referenced from theAppleWiki, not hosted.

The goal is a guide anyone can *reproduce* with their own legally obtained
firmware and publicly available keys.
