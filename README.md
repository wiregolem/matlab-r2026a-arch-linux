# MATLAB R2026a on Arch Linux (Private Compatibility Libraries)

### TL;DR

This guide installs **MATLAB R2026a on Arch Linux** using the official MathWorks Package Manager (MPM), without modifying or downgrading system-wide libraries.

It addresses three runtime compatibility issues:

1. **Licensing crash (`lc_init` symbol collision):** Extracts compatible versions of `gnutls` (3.8.9-1), `nettle` (3.10.1-1), and `leancrypto` (1.2.0-1) into a private MATLAB directory, preventing MATLAB's licensing library from incorrectly resolving `lc_init` to the system's Leancrypto implementation.

2. **Java classpath cache crash:** Uses `LD_PRELOAD` alongside `LD_LIBRARY_PATH` to explicitly load compatible GnuTLS, Hogweed, Nettle, and Leancrypto libraries, resolving the subsequent startup failure in `mwJavaClasspathCache.so`.

3. **Missing `libcrypt.so.1`:** Privately extracts `libxcrypt-compat` (4.5.2-1), resolving a missing legacy dependency encountered when opening Simulink.

The official MPM installation also avoids the graphical installer crash encountered on the tested system. A [third-party C shim](https://github.com/stefanoconiglio/MatlabR2026a-fix) has been proposed for that issue, but this guide deliberately avoids it because it relies on fixed-address memory mapping and segmentation-fault interception to alter execution flow. See the [ArchWiki MATLAB article](https://wiki.archlinux.org/title/MATLAB) for additional compatibility information.

**All compatibility packages are extracted locally rather than installed or downgraded through pacman.** A user-local launcher applies `LD_LIBRARY_PATH` and `LD_PRELOAD` specifically to MATLAB and its child processes. These variables should not be set globally.

**Verified:** MATLAB R2026a Update 5 launches successfully, and the Simscape Electrical inverting-amplifier example executes and produces output plots under KDE Plasma 6 Wayland.

---

### Tested Configuration

- **MATLAB:** R2026a Update 5
- **Toolboxes:** MATLAB, Simulink, Simscape, Simscape Electrical
- **Operating system:** Arch Linux, x86_64
- **Desktop environment:** KDE Plasma 6, Wayland
- **Simulation test:** `openExample("simscape/OpAmpCircuitInvertingAmplifierExample")`

This is a community workaround for an **unsupported MATLAB platform**. Successful operation of the tested example does not guarantee compatibility with every MATLAB component, hardware configuration, or subsequent release.

# Installation

## 1. System Tools (pacman)

```bash
sudo pacman -S --needed curl libarchive unzip ca-certificates
```

`libarchive` provides `bsdtar`; these are ordinary host tools. This guide does **not** install old `gnutls`, `nettle`, or `leancrypto` globally.

## 2. Install MATLAB with MathWorks Package Manager

A valid MathWorks license is required to run MATLAB. The installation itself does not require root privileges.

```bash
mkdir -p "$HOME/.local/opt/MATLAB/tools"
cd "$HOME/.local/opt/MATLAB/tools"
curl -fL -o mpm https://www.mathworks.com/mpm/glnxa64/mpm
chmod u+x mpm

./mpm install \
  --release=R2026a \
  --destination="$HOME/.local/opt/MATLAB/R2026a" \
  --products=MATLAB Simulink Simscape Simscape_Electrical
```

MPM normally installs the latest available update within that release. Our tested version was **R2026a Update 5**. For an exact update level, consult MathWorks' MPM input-file/update-level support rather than assuming the command above always installs Update 5.

Official references: https://www.mathworks.com/help/install/ug/get-mpm-os-command-line.html and https://www.mathworks.com/help/install/ug/mpminstall.html .

## 3. Retrieve the pinned compatibility packages

These are the **exact Arch package versions** used in the successful setup:

| Package | Version | Archive filename |
| --- | --- | --- |
| gnutls | 3.8.9-1 | `gnutls-3.8.9-1-x86_64.pkg.tar.zst` |
| nettle | 3.10.1-1 | `nettle-3.10.1-1-x86_64.pkg.tar.zst` |
| leancrypto | 1.2.0-1 | `leancrypto-1.2.0-1-x86_64.pkg.tar.zst` |
| libxcrypt-compat | 4.5.2-1 | `libxcrypt-compat-4.5.2-1-x86_64.pkg.tar.zst` |

```bash
mkdir -p "$HOME/.local/opt/MATLAB/compatibility/root"
cd "$HOME/.local/opt/MATLAB/compatibility"

for spec in \
  g/gnutls/gnutls-3.8.9-1-x86_64.pkg.tar.zst \
  n/nettle/nettle-3.10.1-1-x86_64.pkg.tar.zst \
  l/leancrypto/leancrypto-1.2.0-1-x86_64.pkg.tar.zst; do
  curl -fL --retry 3 -O "https://archive.archlinux.org/packages/$spec"
done

curl -fL --retry 3 \
  -o libxcrypt-compat-4.5.2-1-x86_64.pkg.tar.zst \
  https://archive.archlinux.org/packages/l/libxcrypt-compat/libxcrypt-compat-4.5.2-1-x86_64.pkg.tar.zst
```

If the archive is unavailable, consult https://archive.archlinux.org/packages/ and obtain the **same filename** from an official Arch archive or mirror. For public redistribution, check package signatures/checksums; the above commands do not perform cryptographic package verification on their own.

## 4. Extract **only** the shared libraries into MATLAB's private directory

```bash
cd "$HOME/.local/opt/MATLAB/compatibility"
ROOT="$HOME/.local/opt/MATLAB/compatibility/root"

for pkg in \
  gnutls-3.8.9-1-x86_64.pkg.tar.zst \
  nettle-3.10.1-1-x86_64.pkg.tar.zst \
  leancrypto-1.2.0-1-x86_64.pkg.tar.zst \
  libxcrypt-compat-4.5.2-1-x86_64.pkg.tar.zst; do
  # Keep package contents user-local. Extract only usr/lib/*, retaining symlinks.
  bsdtar -tf "$pkg" | grep '^usr/lib/' | \
    bsdtar -xvf "$pkg" -C "$ROOT" -T -
done
```

This creates `~/.local/opt/MATLAB/compatibility/root/usr/lib/`. **Do not** replace `/usr/lib` files, install these old packages with `pacman -U`, or symlink `libcrypt.so.1` to `libcrypt.so.2`.

Check the required filenames:

```bash
COMPAT="$HOME/.local/opt/MATLAB/compatibility/root/usr/lib"
ls -l "$COMPAT"/{libgnutls.so.30,libhogweed.so.6,libnettle.so.8,libleancrypto.so.1,libcrypt.so.1}
```

The `libcrypt.so.1` symlink must point to the extracted `libcrypt.so.1.1.0` file.

## 5. Install the working MATLAB launcher

```bash
mkdir -p "$HOME/.local/bin"
cat > "$HOME/.local/bin/matlab" <<'LAUNCHER'
#!/usr/bin/env bash
MATLAB_ROOT="$HOME/.local/opt/MATLAB/R2026a"
COMPAT="$HOME/.local/opt/MATLAB/compatibility/root/usr/lib"

export LD_LIBRARY_PATH="$COMPAT${LD_LIBRARY_PATH:+:$LD_LIBRARY_PATH}"
export LD_PRELOAD="$COMPAT/libgnutls.so.30:$COMPAT/libhogweed.so.6:$COMPAT/libnettle.so.8:$COMPAT/libleancrypto.so.1"
exec "$MATLAB_ROOT/bin/matlab" "$@"
LAUNCHER
chmod u+x "$HOME/.local/bin/matlab"
bash -n "$HOME/.local/bin/matlab"
```

Run with `"$HOME/.local/bin/matlab"`. To invoke it as `matlab`, add `~/.local/bin` to your shell's `PATH` (in an appropriate shell startup file):

```bash
case ":$PATH:" in
  *":$HOME/.local/bin:"*) ;;
  *) export PATH="$HOME/.local/bin:$PATH" ;;
esac
```

**Both** `LD_LIBRARY_PATH` and all four preloads were needed in the tested setup. `libcrypt.so.1` is found through `LD_LIBRARY_PATH` and need not be preloaded.

## 6. Launch MATLAB

Launch MATLAB using the compatibility wrapper:

```bash
~/.local/bin/matlab
```

The wrapper applies the required LD_LIBRARY_PATH and LD_PRELOAD settings only to MATLAB and its child processes, without modifying system-wide libraries.

On first launch, sign in to your MathWorks account and activate or select an appropriate license if prompted.

If MATLAB opens successfully, the installation is ready for use.

The three license tests returned `1` in our setup. In the opened example, click **Run** and verify that the simulation finishes and plots appear. Successful completion of this example confirms one representative Simscape Electrical simulation, not exhaustive compatibility.

## Notes and limitations

- Arch Linux is not an officially supported MATLAB distribution; other releases, graphics configurations and toolboxes may behave differently.
- Pinned older crypto-related libraries may have security/compatibility implications. They are confined to processes launched via this wrapper rather than installed into the system package database, but they are **not** a security sandbox.
- Avoid setting `LD_PRELOAD` or `LD_LIBRARY_PATH` system-wide. Reassess library versions after upgrades.
- The legacy `libcrypt.so.1` is provided by `libxcrypt-compat` (not an ABI-equivalent symlink to `libcrypt.so.2`).
- MPM and MATLAB remain MathWorks products subject to their license terms; do not redistribute their binaries.

### References

- MathWorks, Get MATLAB Package Manager: https://www.mathworks.com/help/install/ug/get-mpm-os-command-line.html
- MathWorks, `mpm install`: https://www.mathworks.com/help/install/ug/mpminstall.html
- Arch Linux package archive: https://archive.archlinux.org/packages/
- Arch Linux `libxcrypt-compat` package: https://archlinux.org/packages/core/x86_64/libxcrypt-compat/
