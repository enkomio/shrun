# Shellcode Runner

A minimal Windows PE builder written in Rust. Takes a raw shellcode file (or a hex string) and produces a standalone `.exe` that wraps the shellcode in a single RWX `.text` section — ready to load in your preferred debugger.

Intended for shellcode analysis and debugging. No external dependencies, no runtime, no imports beyond `KERNEL32.VirtualAlloc` — just a bare PE with your bytes starting at `ImageBase + 0x1000`.

---

## Usage

```
shrun.exe <input> [32|64]
```

| Argument | Description |
|----------|-------------|
| `input`  | Path to a raw shellcode binary, **or** a hex-encoded string |
| `32\|64` | Output PE bitness. Default: `64` |

---

## Examples

### From a binary file

```powershell
# 64-bit PE (default)
.\shrun.exe .\shellcode.bin

# 32-bit PE
.\shrun.exe .\shellcode.bin 32
```

Output:
```
[*] input:     C:\Users\user\Desktop\shellcode.bin
[*] mode:      PE64 (64-bit)
[*] payload:   42 bytes
[*] output:    C:\Users\user\Desktop\shellcode_sh.exe
[*] shellcode: 0x0000000180001000  (= BASE_ADDRESS → rcx)
[+] done  —  entry/stub: 0x000000018000102a
```

---

### From a hex string

If the first argument is not an existing file, it is decoded as a hex string and used directly as the shellcode payload.

```powershell
# x64: xor rax,rax / inc rax / ret
.\shrun.exe 4831c048ffc0c3 64

# x86: xor eax,eax / inc eax / ret
.\shrun.exe 31c040c3 32

# 0x prefix accepted
.\shrun.exe 0x4831c048ffc0c3
```

Output:
```
[*] input:     <hex string>
[*] mode:      PE64 (64-bit)
[*] payload:   7 bytes
[*] output:    C:\Users\user\Desktop\shellcode_sh.exe
[*] shellcode: 0x0000000180001000  (= BASE_ADDRESS → rcx)
[+] done  —  entry/stub: 0x0000000180001007
```

---

## Build

```powershell
# 64-bit host (default on Windows)
cargo build --release --target x86_64-pc-windows-msvc

# 32-bit target
cargo build --release --target i686-pc-windows-msvc
```

---

## Features

- Builds a minimal PE32 or PE32+ executable from scratch (hardcoded header, no linker)
- Accepts a raw binary file **or** a hex string as input
- Supports both **32-bit** (PE32 / x86) and **64-bit** (PE32+ / x86-64) output
- ASLR disabled — `DllCharacteristics = 0x0000` (no `DYNAMIC_BASE`)
- DEP disabled — no `NX_COMPAT`, `.text` section flagged as `RWX` (`0xE0000020`)
- Shellcode placed at `ImageBase + RVA 0x1000` (section start, always page-aligned)
- A small **position-independent stub** is appended after the shellcode and set as the entry point; it passes `ImageBase + 0x1000` (the shellcode address) as the first argument before jumping to the shellcode
- Three imports from `KERNEL32.DLL` (import table in `.rdata`): `VirtualAlloc`, `LoadLibraryA`, `GetProcAddress`

### PE layout

```
File offset 0x000   PE headers (padded to 0x200)
File offset 0x200   .text  RVA 0x1000  — RWX
                      [shellcode bytes]
                      [stub 17–18 bytes]  ← AddressOfEntryPoint
File offset 0x200+  .rdata              — import table (VirtualAlloc, LoadLibraryA, GetProcAddress)
```

### Stub behaviour

| Step | x64 | x86 |
|------|-----|-----|
| Recover shellcode address | `CALL $+5` / `POP RCX` / `SUB RCX, imm32` | `CALL $+5` / `POP EAX` / `SUB EAX, imm32` |
| Pass as first argument | value in **RCX** | **PUSH EAX** |
| Transfer control | `JMP rel32` → shellcode | `JMP rel32` → shellcode |

The subtraction constant is computed at build time as `shellcode_len + 5`, so the result is always `ImageBase + RVA_TEXT` regardless of payload size.
