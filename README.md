# Shellcode Runner

A vide-coded minimal Windows PE builder written in Rust. Takes a raw shellcode file (or a hex string) and produces a standalone `.exe` that places the shellcode in a single RWX `.text` section with a fixed entry point — ready to load in your preferred debugger.

Intended for shellcode analysis and debugging. No external dependencies, no runtime, no imports — just a bare PE with your bytes at `0x00401000`.

---

## Features

- Builds a minimal PE32 or PE32+ executable from scratch (hardcoded header, no linker)
- Accepts a raw binary file **or** a hex string as input
- Supports both **32-bit** (PE32 / x86) and **64-bit** (PE32+ / x86-64) output
- ASLR disabled — `DllCharacteristics = 0x0000` (no `DYNAMIC_BASE`)
- DEP disabled — no `NX_COMPAT`, `.text` section flagged as `RWX`
- Entry point always at `ImageBase + 0x1000` = `0x00401000`
- No imports, no relocations, no data directories — clean for static analysis

---

## Build

```powershell
# 64-bit target (default on Windows)
cargo build --release --target x86_64-pc-windows-msvc

# 32-bit target
cargo build --release --target i686-pc-windows-msvc
```

Or with `rustc` directly:

```powershell
rustc main.rs -o shrun.exe
```

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
[*] input:    shellcode.bin
[*] mode:     PE64 (64-bit)
[*] payload:  42 bytes
[*] output:   shellcode_sh.exe
[+] done — entry point: 0x00401000
```

---

### From a hex string

If the first argument is not an existing file, the tool attempts to decode it as a hex string and uses it as the shellcode payload directly.

```powershell
# Simple x64 shellcode: xor rax,rax / inc rax / ret
.\shrun.exe 4831c048ffc0c3 64

# x86 shellcode: xor eax,eax / inc eax / ret
.\shrun.exe 31c040c3 32

# With 0x prefix (both forms accepted)
.\shrun.exe 0x4831c048ffc0c3
```

Output:
```
[*] input:    <hex string>
[*] mode:     PE64 (64-bit)
[*] payload:  7 bytes
[*] output:   shellcode_sh.exe
[+] done — entry point: 0x00401000
```
