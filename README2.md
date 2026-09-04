# DKR-R — Fedora 44 KDE Plasma Native Build (No WSL / No AppImage)

This guide builds **DKR-R 1.0.4** natively on **Fedora Linux 44 (KDE Plasma)** using modern Fedora packages — not the old Ubuntu 24.04 packages documented upstream. It replaces the WSL-first workflow (`Build-DKR-Runtime.cmd` + WSL Ubuntu) with a pure Linux flow and produces an unpackaged native binary.

**Verified on:** Fedora 44 KDE Plasma, `cmake 4.3.0`, `ninja 1.13.2`, `gcc 16.2.1`, `clang 22.1.8`, Vulkan `1.4.341`, SDL2 `2.32.70` / `sdl2-compat`, `SDL3 3.4.14`, `gtk3 3.24.52`.

**Important:** `funcs` (RecompiledFuncs) **must be recompiled before the game binary can be built**. The final `runtime-recomp` CMake build fails if `DKR_GENERATED_SOURCE_V77` or `DKR_GENERATED_SOURCE_V80` are missing or identical — both revision payloads are linked into one `DKR-R` executable.

Binary output (native, no AppImage):
```text
build/dkr-runtime-linux/bin/Release/DKR-R
build/dkr-runtime-linux/bin/Release/libexec/dkr-r/DKR-R-InputHost + libSDL3.so.0
```

---

## 1. Prerequisites (Fedora 44)

Upstream's `Setup-Linux.sh` prints Ubuntu `apt` names. On Fedora use `dnf`:

```bash
sudo dnf install \
  cmake ninja-build pkg-config git python3 python3-pip \
  gcc gcc-c++ clang lld \
  SDL2-devel SDL3-devel gtk3-devel \
  vulkan-headers vulkan-loader-devel vulkan-tools vulkan-validation-layers-devel \
  alsa-lib-devel pulseaudio-libs-devel libsamplerate-devel \
  libX11-devel libXext-devel libXrandr-devel wayland-devel \
  libxkbcommon-devel libxkbcommon-x11-devel \
  pcre2-devel libpng-devel zlib-devel \
  binutils-mips64-linux-gnu
```

Check:
```bash
cmake --version; ninja --version; c++ --version; pkg-config --modversion sdl2; pkg-config --modversion gtk+-3.0; pkg-config --modversion vulkan
vulkaninfo --summary | head
```

> **Note:** `Setup-Linux.sh` only checks `sdl2`, `gtk+-3.0`, `vulkan` via `pkg-config`. On Fedora `libsamplerate` is queried as `samplerate` (`pkg-config --modversion samplerate`), not `libsamplerate`, and `SDL2` is provided by `sdl2-compat`. The runtime also needs ALSA/Pulse/Samplerate/X11/Wayland at build time (they are linked via RT64/plume).

---

## 2. Prepare ROMs

Both US revisions are required — one `DKR-R` binary contains both payloads and selects the correct one at runtime by SHA-1.

Expected SHA-1 **after** big-endian normalisation (`z64`):
```text
0cb115d8716dbbc2922fda38e533b9fe63bb9670  # US v1.0 / v77
6d96743d46f8c0cd0edb0ec5600b003c89b93755  # US Rev A / v1.1 / v80
```

If you have the two dumps as `baserom.z64` (v77) and `baseromv1-1.z64` (v80) in the repo root (as provided in this fork), copy/normalise them:

```bash
# v77 already at baseroms/baserom.us.v77.z64 and extern/dkr-decomp/baseroms/baserom.us.v77.z64
# v80 needs to be present as baserom.us.v80.z64:
cp baseromv1-1.z64 extern/dkr-decomp/baseroms/baserom.us.v80.z64
cp extern/dkr-decomp/baseroms/baserom.us.v77.z64 baseroms/baserom.us.v77.z64 2>/dev/null || true
cp extern/dkr-decomp/baseroms/baserom.us.v80.z64 baseroms/baserom.us.v80.z64 2>/dev/null || true
sha1sum baserom.z64 baseromv1-1.z64 extern/dkr-decomp/baseroms/baserom.us.v*.z64
```

The decomp's `ver/splat/update_baserom_names.py` auto-renames by SHA-1, so any filename with the correct hash is accepted.

---

## 3. Clone pinned dependencies (no WSL)

From the repo root:

```bash
python3 scripts/bootstrap_dependencies.py
bash scripts/apply-dependency-patches.sh
```

This clones the exact commits from `dependencies.lock.json` into `extern/` (`n64-modern-runtime`, `rt64`, `gekkonet`, `monocypher`, `libdatachannel`, `mbedtls`, `sdl3`) and applies `patches/manifest.json`.

N64Recomp is a submodule of `n64-modern-runtime`; its two patches must be applied manually on Fedora because `apply-dependency-patches.sh` skips the `.git` file check:

```bash
git -C extern/n64-modern-runtime/N64Recomp apply patches/n64recomp/0001-fix-high-vram-entrypoint-comparison.patch
git -C extern/n64-modern-runtime/N64Recomp apply patches/n64recomp/0002-name-recomp-context-struct.patch
```

---

## 4. Build matching decomp ELFs (both revisions)

Upstream builds the ELF under WSL `mips-linux-gnu`. On Fedora 44 the system `mips64-linux-gnu-ld` (binutils 2.46) fails with:

```
header.s.o: .symtab local symbol at index 8 (>= sh_info)
...
ABI is incompatible with that of the selected emulation
```

Use the decomp's bundled `tools/binutils/mips64-elf-*` (also binutils 2.46 but configured for `elf32btsmip`) via `CROSS=tools/binutils/mips64-elf-`. You must `clean` between versions because `build/` is shared.

```bash
cd extern/dkr-decomp
make distclean
make setup          # creates .venv, builds tools/dkr_assets_tool, fetches ido-5.3

# --- v77 ---
make REGION=us VERSION=v77 CROSS=tools/binutils/mips64-elf- extract
make REGION=us VERSION=v77 CROSS=tools/binutils/mips64-elf- -j$(nproc)
cp build/dkr.us.v77.elf /tmp/dkr.us.v77.elf
cp build/dkr.us.v77.z64 /tmp/dkr.us.v77.z64

# --- v80 ---
make clean
make REGION=us VERSION=v80 CROSS=tools/binutils/mips64-elf- extract
make REGION=us VERSION=v80 CROSS=tools/binutils/mips64-elf- -j$(nproc)
cp build/dkr.us.v80.elf /tmp/dkr.us.v80.elf
cp build/dkr.us.v80.z64 /tmp/dkr.us.v80.z64

# restore both to build/ for the recomp step
cp /tmp/dkr.us.v77.elf build/dkr.us.v77.elf
cp /tmp/dkr.us.v77.z64 build/dkr.us.v77.z64
cp /tmp/dkr.us.v80.elf build/dkr.us.v80.elf
cp /tmp/dkr.us.v80.z64 build/dkr.us.v80.z64
sha1sum build/dkr.us.v77.z64 build/dkr.us.v80.z64
# should be 0cb115... and 6d967...
cd ../..
```

---

## 5. Build recompilation tools (native, no Visual Studio)

```bash
cmake -S extern/n64-modern-runtime/N64Recomp -B build/runtime-tools/n64recomp -G Ninja
cmake --build build/runtime-tools/n64recomp --parallel --target N64RecompCLI RSPRecomp
# outputs: build/runtime-tools/n64recomp/N64Recomp  and  RSPRecomp
```

---

## 6. Recompile funcs — must be done before building the game

Runtime policies already exist at `runtime-recomp/dkr.us.v77.recomp-policy.json` and `dkr.us.v80.recomp-policy.json`.

Resolve entrypoints (`mainproc`) and `use_mdebug`:

```bash
# v77: 0x80065D40, v80: 0x80065F80, both use_mdebug=false
python3 - <<'PY'
import re, subprocess, shutil, pathlib, struct
for ver, elf, rom in [
  ("v77", "extern/dkr-decomp/build/dkr.us.v77.elf", "extern/dkr-decomp/build/dkr.us.v77.z64"),
  ("v80", "extern/dkr-decomp/build/dkr.us.v80.elf", "extern/dkr-decomp/build/dkr.us.v80.z64"),
]:
  b = pathlib.Path(rom).read_bytes()
  hdr = int.from_bytes(b[8:12], 'big')
  nm = shutil.which("mips64-linux-gnu-nm")
  out = subprocess.check_output([nm, "-n", "--defined-only", elf], text=True)
  mp = next((int(re.match(r'^\s*([0-9A-Fa-f]+)',l).group(1),16) for l in out.splitlines() if " mainproc" in l), None)
  print(ver, f"header {hdr:#x} mainproc {mp:#x}" if mp else "no mainproc")
PY
```

Generate TOML + run N64Recomp:

```bash
mkdir -p runtime-recomp/RecompiledFuncs build/generated-v80
rm -rf runtime-recomp/RecompiledFuncs/* build/generated-v80/*

python3 scripts/generate_recomp_config.py \
  --policy runtime-recomp/dkr.us.v77.recomp-policy.json \
  --elf extern/dkr-decomp/build/dkr.us.v77.elf \
  --rom extern/dkr-decomp/build/dkr.us.v77.z64 \
  --output-functions runtime-recomp/RecompiledFuncs \
  --entrypoint 0x80065D40 \
  --output runtime-recomp/dkr.us.v77.generated.toml

python3 scripts/generate_recomp_config.py \
  --policy runtime-recomp/dkr.us.v80.recomp-policy.json \
  --elf extern/dkr-decomp/build/dkr.us.v80.elf \
  --rom extern/dkr-decomp/build/dkr.us.v80.z64 \
  --output-functions build/generated-v80 \
  --entrypoint 0x80065F80 \
  --output build/dkr.us.v80.generated.toml

build/runtime-tools/n64recomp/N64Recomp runtime-recomp/dkr.us.v77.generated.toml
build/runtime-tools/n64recomp/N64Recomp build/dkr.us.v80.generated.toml

ls runtime-recomp/RecompiledFuncs | wc -l   # ~37 funcs_*.c + headers
ls build/generated-v80 | wc -l
```

> **If you skip this step, `cmake` in §7 fails with `DKR_GENERATED_SOURCE_V80 must name ...` or `No N64Recomp output was found`.**

---

## 7. Recompile RSP microcode

```bash
build/runtime-tools/n64recomp/RSPRecomp runtime-recomp/rsp/aspMain.us.v77.toml
ls -lh runtime-recomp/RecompiledRSP/aspMain.cpp
```

Single RSP payload is shared between revisions.

---

## 8. Configure and build native DKR-R (no AppImage)

```bash
VERSION=$(tr -d '\r\n' < VERSION)
cmake -S runtime-recomp -B build/dkr-runtime-linux -G Ninja \
  -DCMAKE_BUILD_TYPE=Release \
  -DDKRPORT_ROOT="$(pwd)" \
  -DDKR_RELEASE_VERSION="$VERSION" \
  -DDKR_RUNTIME_BUILD_GENERATED=ON \
  -DDKR_RUNTIME_BUILD_RT64=ON \
  -DDKR_RUNTIME_BUILD_SDL3_INPUT_HOST=ON \
  -DDKR_GENERATED_SOURCE_V77="$(pwd)/runtime-recomp/RecompiledFuncs" \
  -DDKR_GENERATED_SOURCE_V80="$(pwd)/build/generated-v80"

cmake --build build/dkr-runtime-linux --parallel
ctest --test-dir build/dkr-runtime-linux --output-on-failure -R '^DKR'
```

Tests: 55 passed. Self-tests:

```bash
build/dkr-runtime-linux/bin/Release/DKR-R --self-test-pak "$(mktemp -d)"
build/dkr-runtime-linux/bin/Release/libexec/dkr-r/DKR-R-InputHost --self-test --mappings assets/controllers/gamecontrollerdb.txt
```

No `linuxdeploy`/`patchelf`/`AppImage` required — this is a normal ELF linked against system `libSDL2`, `libvulkan`, `gtk3`, etc.

---

## 9. Run

```bash
./build/dkr-runtime-linux/bin/Release/DKR-R
# or: ./build/dkr-runtime-linux/bin/Release/DKR-R --rom /path/to/dkr.us.v77.z64
```

The launcher prompts for a ROM on first run (accepts `.z64`/`.v64`/`.n64`, auto-detects byte order, validates SHA-1 before renderer/audio startup). In-game overlay: `Esc`/`F1` or controller `View`/`Back`; fullscreen `Alt+Enter`/`F11`.

To make an AppImage later (optional, matches `Build-Linux.sh`):

```bash
DKR_LINUX_V80_GENERATED_SOURCE="$(pwd)/build/generated-v80" ./Build-Linux.sh
# produces dist/DKR-R-1.0.4-Linux-x86_64.AppImage (needs linuxdeploy + patchelf)
```

---

## 10. Troubleshooting (Fedora)

- `Setup-Linux.sh` says missing `libsamplerate` / `alsa`: On Fedora `pkg-config` name is `samplerate` (`samplerate` provides `libsamplerate-devel`). Check `pkg-config --modversion samplerate alsa sdl2 gtk+-3.0 vulkan`.
- `mips64-linux-gnu-ld: failed to merge target specific data … ABI is incompatible`: Always build the decomp with `CROSS=tools/binutils/mips64-elf-` and `clean` between `VERSION=v77`/`v80`.
- `N64Recomp failed` / `No N64Recomp output was found`: Ensure `build/runtime-tools/n64recomp/N64Recomp` exists and both `RecompiledFuncs` directories are populated **before** the `cmake -S runtime-recomp` step.
- `Missing N64ModernRuntime`: Ensure `python3 scripts/bootstrap_dependencies.py` was run and `extern/n64-modern-runtime/CMakeLists.txt` exists.
- Wayland vs X11: The build enables both `wayland` and `x11` SDL video drivers dynamically; on KDE Wayland the binary auto-selects Wayland.

See `docs/BUILDING.md:1`, `runtime-recomp/README.md`, `docs/ROM_SETUP.md`.
