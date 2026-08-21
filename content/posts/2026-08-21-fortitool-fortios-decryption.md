---
layout: post
title:  "fortitool: cracking FortiOS firmware end to end, and a key nobody had"
comments: true
aliases: [/fortitool-fortios-decryption]
date:   2026-08-21
tags:
- Fortinet
- FortiOS
- Reverse Engineering
- Firmware
- Cryptography
keywords:
- fortios
- fortigate
- firmware decryption
- fortitool
- CVE-2019-6693
- config-secret
- forticrack
description: 'A single static Go binary that decrypts and unpacks FortiOS firmware end to end, plus the FortiOS 7.4 config-secret key nobody had published before this.'
images:
- /img/fortitool-fortios-decryption/titlepic.png
categories:
- Reverse Engineering
---

I own an FWF-60E and I've spent a good chunk of time pulling its firmware apart to see what's actually in there. FortiOS doesn't do casual encryption. The whole `.out` file sits under an outer XOR layer, and the filesystem image buried inside that has its own second scheme, which has changed shape several times across versions. None of this is exactly secret. Researchers have been chipping away at it for years. But every writeup solves one slice and then leaves you to stitch the rest together on your own.

<!--more-->

![fortitool: cracking FortiOS firmware end to end, and a key nobody had](/img/fortitool-fortios-decryption/titlepic.png)

## The tooling problem

So before this tool existed, decrypting a FortiOS image meant picking your weapon based on version and architecture. `forticrack` for the outer layer. `fgx` for the 7.6.x rootfs scheme. `forticrack_v8` for 8.0. A pile of blog posts and one-off scripts if you were on the ARM 7.4.x path. Each one handles its slice fine. They don't talk to each other, and half of them want `openssl`, `binwalk`, or Python installed and behaving themselves on your box.

I got tired of that. So I built `fortitool`: one static Go binary, zero runtime dependencies, that runs the whole pipeline. Outer cipher, rootfs decryption, filesystem extraction, signature verification, config-secret decryption. It picks the right method by itself instead of demanding a version flag.

## Two layers, and how to find keys without a disassembler

First, what's actually inside one of these images. The `.out` file is a gzip wrapper around an encrypted disk image. Peel that and you get an MBR with a single ext3 volume labeled `FORTIOS` starting at offset 512. Inside the volume: `flatkc` (the kernel), `rootfs.gz` (the initramfs), `datafs.tar.gz` (IPS/AV seeds, config templates, web assets), and some boot metadata. Two quirks worth knowing early. `flatkc` is a Linux 3.2.16 kernel, and Fortinet never published the GPL source for it. And those inner `.tar.xz` archives use a Fortinet-modified xz format that standard xz refuses to touch; you extract them with the `ftar` binary shipped in the image's own `sbin/`.

FortiOS stacks two crypto layers on top of that. The outer one wraps the entire disk image in a block XOR cipher: 512-byte blocks, a 32-byte alphanumeric key, CBC-like chaining between blocks with a fixed IV of `0xFF`. BishopFox broke it with a known-plaintext attack. The trick is knowing that bytes 48 through 79 of each decrypted block are null padding in practice, which hands you the keystream directly. And since Fortinet reuses a tiny pool of keys across product lines (BishopFox found roughly 25 distinct keys across ~29k images), recovery is fast. My `internal/l1` package is that attack reimplemented from scratch in Go.

The inner layer wraps the actual root filesystem, and this is where things fork by era. From 7.4.1 through 7.4.11 you get ChaCha20 with a non-RFC7539 counter layout, where the key and IV come from SHA-256 over rotated slices of a 32-byte seed hidden in the kernel next to a 270-byte obfuscated RSA public key, plus AES-CTR for the body with the CTR key carried inside the signature block and a custom counter step. From 7.6.x onward it switches to XOR plus RSA plus either FORT-RC4 or a modified RC4 variant, depending on the silicon. FORT-RC4 deserves its own sentence honestly: hacefresko found it's RC4 with a modified KSA and PRGA, and the 32-byte key rides inside a PKCS#1 v1.5 signature appended to `rootfs.gz`. Even the variants differ per product line; the FGT build zeroes the RC4 state after KSA while the FFW build carries `j` across.

Here's the fun part though: in every single case, the key material FortiOS hides in the kernel image can be found without ever opening a disassembler. The recovered RSA key always decrypts to the same ASN.1 DER prefix, so you scan the kernel for byte windows satisfying that constraint. Then you take each candidate, try every rootfs cipher against a probe block, and look for gzip magic bytes in the output. When it comes out right, you have the cipher and the key at once. Ghidra never opened.

I can't take credit for the scanning trick itself. That belongs to `hackintoanetwork/fgx`, and RandoriSec independently found the same class of trick for the 7.4.7+ stripped-kernel path. What I did was generalize it into one scanner covering every layout I could find real firmware for, then wire it into a pipeline that doesn't care which era it's looking at.

## The part that took a disassembler: config-secret

FortiOS config backups store secrets as `set <field> ENC <base64>` lines. Admin passwords, PSK values, certificate passphrases, all of them. This has been a known weak point since 2019, when gquere published CVE-2019-6693: a hardcoded AES-128-CBC key baked into the firmware. Literally the string "Mary had a littl". Sixteen bytes, no more, no less. Fortinet's PSIRT advisory FG-IR-19-007 responded to it.

The common understanding of that advisory is just wrong, though. The story everyone repeats is that Fortinet rotated the hardcoded key at FortiOS 6.2 in response. I went looking for evidence of that rotation and couldn't find it in any image I had. So I pulled apart the `init` binary across several versions to check directly. Turns out the 6.2 fix isn't a key rotation at all. It's a different, opt-in feature called `private-data-encryption`, basically a passphrase over the whole backup file that you have to turn on yourself. The per-field `ENC` mechanism, the thing gquere's key actually targets? Never touched. That same 2019 key still decrypts real secrets on builds through at least 7.2.3. Six minor versions and three years past when everyone assumed it was dead.

The blob format makes this easy to verify yourself, and it's worth spelling out because the layout barely changes even after the key finally does. Base64-decode an `ENC` blob and you get: 4 bytes of random IV, zero-padded to a full AES block, followed by ciphertext. Pre-7.4 blobs are exactly 200 base64 characters (148 raw bytes). Certificate and PKI passwords use a fixed 144-byte buffer, nine AES blocks regardless of secret length, no padding scheme at all, just a calloc'd buffer with the secret copied in at offset zero and everything after it left as zeros, so the plaintext runs to the first null byte.

My favorite correctness proof from this whole project: running the legacy key against my 7.2.3 backup recovered all 22 factory certificate passwords as clean 64-char hex strings, plus the default admin account's password, which decrypted to the literal ASCII word `guest`. A plain English word falling out the far end is about as good as blind key validation gets.

It does change eventually. Just much later than people think. At 7.4, specifically build 2731, the cipher moves to AES-256-CBC under a new hardcoded key, and the encoding grows a tell: an unencrypted 8-byte ASCII marker, `Yf267vE@`, gets tacked onto the ciphertext before base64 encoding. That takes the blob from 200 to exactly 208 base64 characters, nothing else moves. I actually found the marker first. It sat there byte-identical across every 7.4-era blob I had, spanning two months of backups, even though each blob carried its own random IV. If that marker were inside the CBC output it would vary with the IV. So it had to be appended after encryption.

But finding the marker isn't finding the key behind it. So I loaded `/bin/init` from a 7.4.11 image into Ghidra and started tracing. Getting there was its own small fight: the binary is ARM Thumb code, so raw byte scans for ARM-mode `BL` encodings return nothing; you find code through literal-pool pointer loads instead. Anchoring on the `"ENC %s"` format string led to the CMDB field-type dispatcher, a 28-entry jump table keyed on the config variable's declared type, with the password cases calling into the crypto.

There's a dispatch function there, `FUN_01638e34`, that branches on a mode flag between two static key loaders. One deobfuscates the legacy 2019 key by backward XOR-chaining a blob (`b[i] ^= b[i-1]`, high to low), which decodes to "Mary had a little lamb..." spelled out in full. The other just memcpy's a hardcoded 32-byte constant straight out of the binary at `0x024126ec` and feeds it to `EVP_aes_256_cbc` with padding disabled. That constant is the new key.

I checked it against every real `ENC` field in every 7.4-era config backup I have. All 26 new-format blobs come back clean.

The key, in hex:

```
91bc4d1e0e5e35dea0e84803bb1c4cc49699362830f9d6a6c75880b181f6c1db
```

As far as I can tell, nobody has published this before. The 2019 disclosure only covers the pre-6.2 key. Every other public FortiOS crypto writeup I've found (BishopFox, RandoriSec, `fgx`, `forticrack_v8`) is about the rootfs layer, which is a different subsystem entirely. Maybe someone has this key sitting in a private pentest report somewhere. I have no way of knowing. But nothing about it is on the open internet as far as I've searched.

The blob layout otherwise stayed put across the rotation. Same IV construction, same two field shapes. Ordinary admin and user passwords use a shorter, genuinely PKCS#7-padded encoding instead of the fixed 144-byte buffer, and both shapes exist in both eras. `fortitool config decrypt` auto-detects the era via the trailer marker and the layout via length and padding shape, so you hand it the base64 blob and it figures out the rest.

One thing I noticed while validating all this on real hardware: my FWF-60E images across 7.4.7, 7.4.10, and 7.4.11 all reuse a single L1 key, which tracks with BishopFox's finding that the whole product line shares roughly two dozen. Also worth knowing if you're doing this yourself: the ARM appliance line (FSoC3 silicon, four Cortex-A9 cores) kept the old ChaCha20 rootfs scheme through 7.4.11 even after the x86 line moved at 7.4.7. Fortinet hardened the VM images first and left the appliances on the legacy path for another year.

## fortitool

Code's on [GitHub](https://github.com/mosajjal/fortitool). All of the above, plus tar/gzip/xz unpacking, a pure-Go ext2/ext3 reader for the appliance partition layout, and PKCS#7 signature verification for the signed engine and DB files. One binary:

```sh
go install github.com/mosajjal/fortitool/cmd/fortitool@latest
```

```sh
fortitool decrypt -o outdir image.out          # full pipeline, one command
fortitool l1 -o out.img image.out              # outer XOR layer only
fortitool rootfs -o out.gz flatkc rootfs.gz    # rootfs crypto layer only
fortitool config decrypt <base64-blob>         # config-backup ENC secret
```

Build with `CGO_ENABLED=0` and you get a fully static binary. No glibc, no `openssl`, no `binwalk`, no Python. There's also a Claude Code plugin packaging, so an agent working through a firmware image can drive the CLI directly instead of you spelling out flags by hand.

## Credit where it's due

None of this happens without the people who did the hard part first. BishopFox for the original L1 known-plaintext attack and the ChaCha20 rootfs scheme. `hacefresko/forticrack_v8` for the FORT-RC4 cipher. `hackintoanetwork/fgx` for the modified-RC4 cipher and the disassembly-free scanning approach. `noways-io/fortigate-crypto` for the x86_64 ChaCha20 key-derivation splits. RandoriSec for the stripped-kernel writeup. And gquere for the original CVE-2019-6693 disclosure this whole config-secret thread starts from. I reimplemented every algorithm from scratch off their public writeups, not their source, but the discovery credit is theirs regardless.

## Use it responsibly

Only point this at firmware for hardware you actually own. Fortinet's EULA prohibits reverse engineering contractually, but the DMCA §1201 security research exemption covers good-faith work on lawfully owned devices in the US, and that's the line I'm operating inside. Don't redistribute firmware images, and don't run this against anything that isn't yours.

*I built fortitool and wrote this post with Claude's help, from the code to the copyedit.*
