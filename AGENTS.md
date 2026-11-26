# AGENTS.md

## Purpose

This is the LibreELEC.tv build system: a source-built Linux distribution for Kodi.
This fork adds **native Steam integration for a living-room HTPC**: Steam runs
locally on the LibreELEC host as a LibreELEC-conform addon, not via Steam Link
streaming.

Target device: an AMD Ryzen 7 8845HS / Ryzen 9 8945HS mini PC with Radeon 780M,
32 GB dual-channel DDR5 SO-DIMM, two M.2 slots, 4K AV1/HEVC decode, HDMI audio,
and optional USB-CEC. Valve's Linux client is 64-bit since the 2026-11-11 stable
update and runs in a Steam Runtime container, but 32-bit libraries and graphics
drivers are still required; the runtime and games remain self-contained. The
LibreELEC x86_64 image stays pure 64-bit.

Read `docs/specs/steam-native-integration.md` and this file before changing build
logic, kernel configs, or addons.

## Non-negotiable rules

- Build as a normal user, never as root (`config/options:2`). Paths must not contain spaces.
- Run build and validation commands from the repository root.
- Supported `ARCH` values are only `x86_64`, `aarch64`, and `arm` (`config/arch.*`). There is no 32-bit x86 target.
- Do not add 32-bit libraries or packages to the main x86_64 image. LibreELEC builds a pure 64-bit x86_64 userland (gcc/binutils use `--disable-multilib`; no lib32 in the image).
- Keep Steam scoped to x86_64 `Generic` (Vulkan-capable). Steam on ARM would require FEX/box64 + Wine and is out of scope.
- The Steam runtime (including its 32-bit libraries) is self-contained and lives in the addon or on `/storage`, never in the read-only system image.
- Do not commit generated state: `build.*/`, `sources/`, `target/`, `.libreelec/` are gitignored.
- Never commit proprietary Steam binaries, credentials, or tokens. Download/build at image-build time or runtime and verify with SHA256.
- Pin runtime inputs (container digest, rootfs tarball, Steam Runtime version) and record their SHA256.

## Target hardware

The spec's primary target is the AMD 780M class. The relevant candidates:

| Model | CPU | iGPU | RAM | VESA | OCuLink | Notes |
|---|---|---|---|---|---|---|
| Beelink SER8 | Ryzen 7 8845HS / 8745HS | Radeon 780M | 2 x DDR5-5600 SO-DIMM | No | No | Best value; no VESA mount |
| Minisforum UM890 Pro | Ryzen 9 8945HS | Radeon 780M | 2 x DDR5-5600 SO-DIMM, up to 96 GB | Yes | Yes | Recommended for behind-TV |
| Minisforum UM790 Pro | Ryzen 9 7940HS | Radeon 780M | 2 x DDR5 SO-DIMM | Yes | ? | Previous generation; OCuLink only on UM790 XTX |
| ASUS NUC 14 Pro | Core Ultra 7 155H | Intel Arc | 2 x DDR5 SO-DIMM | Yes | No | Better HDMI HD audio; alternative |

Hard requirements:

- 32 GB total as **2 x 16 GB DDR5-5600 SO-DIMM**, dual-channel. Single-channel
  can cost up to ~30% iGPU performance; 8 GB modules are not economical.
- Two M.2 slots: small OS SSD (~256 GB) plus games SSD (500 GB-1 TB).
- 4K AV1/HEVC/H.264 hardware decode and HDMI 2.0a+ (4K60, HDR10, DV passthrough).
- HDMI audio through the soundbar: PCM, Dolby Digital (AC-3), DTS.
- VESA mount if the PC goes behind the TV.
- Optional: Pulse-Eight USB-CEC adapter for TV-remote control and power behavior;
  Auto Power On / WoL / RTC as fallback.

Not preferred: Ryzen AI 9 HX 370 / Radeon 890M (faster, but typically soldered
LPDDR5X, higher price, outside the 300-600 EUR budget). Native 4K AAA gaming is
not a goal; expect 1080p low/medium (optionally FSR/XeSS) for current titles.

Audio caveat: AMDGPU on Linux reliably does PCM, Dolby Digital, and DTS, but not
TrueHD, DTS-HD MA, or Atmos. The reference soundbar (Teufel Cinebar 11 Mk2 2020)
only needs DD/DTS, so this is acceptable. If HD audio passthrough is mandatory,
Intel Arc (ASUS NUC 14 Pro) is the safer choice.

## Architecture reality: x86_64 vs 32-bit

Steam's Linux client is 64-bit since the 2026-11-11 stable update and runs in a
Steam Runtime container, but Valve still requires 32-bit libraries and graphics
drivers; 64-bit-only x86_64 is unsupported. Many games and Proton components are
still 32-bit. LibreELEC's x86_64 image is 64-bit-only, and the Generic kernel
currently disables 32-bit compatibility:

```
projects/Generic/linux/linux.x86_64.conf:691
# CONFIG_IA32_EMULATION is not set
```

Consequences:

1. A 32-bit game/Proton component cannot execute on the stock kernel, even via Flatpak or a container.
2. Enabling 32-bit execution requires `CONFIG_IA32_EMULATION=y` in the x86_64
   kernel config. Change it with `tools/adjust_kernel_config menuconfig` and commit
   the regenerated `projects/Generic/linux/linux.x86_64.conf`. Do not put
   arch-specific symbols into `distributions/LibreELEC/kernel_options`.
3. The kernel switch alone is not enough: the 32-bit loader and libraries must come
   from a self-contained runtime (container / Steam Runtime / Flatpak), never from
   the main image.
4. 32-bit Mesa/Vulkan userspace must match the host kernel driver. The AMD 780M
   with Mesa (`radeonsi`/RADV) is the primary target; NVIDIA 32-bit userspace is
   the riskiest case.
5. `PKG_ARCH` gates packages by target (`x86_64`, `aarch64`, `arm`, `any`). Use
   `PKG_ARCH="x86_64"` for Steam addons. `target_has_feature 32bit` is only true for
   the ARM 32-bit userland, not for x86.
6. The 64-bit Steam client does not remove the 32-bit requirement: Valve still
   lists 64-bit-only x86_64 as unsupported and requires 32-bit libraries and
   graphics drivers.

## Integration paths

### Primary: 32-bit runtime (custom container or Valve steamrt3c)

Build or fetch a pinned runtime and run Steam inside it:

- Upstream-aligned option: use Valve's Steam Runtime container (`steamrt3c`),
  which the 64-bit client now uses. It still needs host IA32 and 32-bit graphics
  drivers.
- Custom option: Debian/Ubuntu i386, Valve's Steam Runtime, or a pinned container
  image (`--platform linux/386`). Build outside the LibreELEC target toolchain.
- Runtime: Docker or Podman (both addons exist at `packages/addons/service/docker`
  and `packages/addons/service/podman`), or bwrap/systemd-nspawn if available.
- Bind-mount: `/dev/dri`, `/dev/input`, `/dev/uinput`, `/dev/hidraw*`, the
  Wayland/X11 socket, and the PulseAudio socket.
- Provide the 32-bit Mesa/RADV/Vulkan loader and ICD inside the runtime.
- Store the rootfs and Steam/Proton data on `/storage`; keep the system image
  read-only.

This is the path that matches the user's existing 32-bit container work and gives
explicit control over the 32-bit stack.

### Fallback: Flatpak addon

LibreELEC ships the plumbing:

- `packages/addons/tools/flatpak/` - Flatpak + bubblewrap + xdg-dbus-proxy, packaged as `tools.flatpak`.
- `packages/addons/tools/externalhelper/` - runs external programs in a minimal Wayland session (`cage`, `seatd`, `kanshi`).
- `packages/addons/flatpak/flatpak.steamlink/` - exact template for a Steam-family launcher.
- `packages/addons/tools/flatpak/source/resources/lib/flatpakhelper/` - Python helper.

Create `packages/addons/flatpak/flatpak.steam/`:

```make
PKG_NAME="flatpak.steam"
PKG_VERSION="0"
PKG_REV="0"
PKG_ARCH="x86_64"
PKG_LICENSE="LicenseRef-proprietary"
PKG_SITE="https://store.steampowered.com/"
PKG_DEPENDS_TARGET="toolchain"
PKG_SHORTDESC="Steam"
PKG_LONGDESC="Steam desktop client via Flatpak (com.valvesoftware.Steam)."
PKG_TOOLCHAIN="manual"

PKG_IS_ADDON="yes"
PKG_ADDON_NAME="Steam"
PKG_ADDON_TYPE="xbmc.python.script"
PKG_ADDON_PROJECTS="any !Generic-legacy"
PKG_ADDON_PROVIDES="executable"
PKG_ADDON_REQUIRES="tools.flatpak:0.0.0"

addon() {
  :
}
```

`source/default.py`:

```python
from flatpakhelper.addon import FlatpakAddon
import xbmcaddon

FlatpakAddon(addon=xbmcaddon.Addon(), appid='com.valvesoftware.Steam').start()
```

Flatpak still needs kernel IA32 emulation; it is not a substitute for the kernel
change. Steam Link is a separate native x86_64 Flatpak streaming client; it is
not a substitute for native Steam and is not the integration path.

### Not allowed

- Steam Link streaming as the integration path.
- Compiling Steam from source.
- `lib32`/32-bit packages in the main image.
- ARM/aarch64 Steam via FEX/box64 + Wine.

## Addon layout and conventions

- Addon directory: `packages/addons/<category>/<pkg-name>/` with `package.mk`, `source/`, `icon/icon.png`, `changelog.txt`.
- Addon ID is derived as `${PKG_SECTION//\//.}.${PKG_NAME}`; when `PKG_SECTION` is unset the ID is `PKG_NAME` (this is why `flatpak.steamlink` has ID `flatpak.steamlink`).
- `PKG_ADDON_TYPE` must have a matching template in `config/addon/<type>.xml` (`xbmc.python.script`, `xbmc.service`, `xbmc.python.pluginsource`, ...).
- `PKG_ADDON_PROJECTS` is matched against `DEVICE` or `PROJECT`; use `any !Generic-legacy` plus `PKG_ARCH` for the real architecture gate.
- Required `package.mk` fields: `PKG_NAME`, `PKG_VERSION`, `PKG_SHA256` (for source downloads), `PKG_LICENSE`, `PKG_SITE`, `PKG_URL`, `PKG_LONGDESC`. Use the full git hash, never an abbreviation.
- Add the SPDX header `# SPDX-License-Identifier: GPL-2.0-or-later` (or the correct license) and a copyright line.
- Late binding: never reference `PKG_BUILD`, `PKG_DIR`, `PKG_INSTALL`, or toolchain variables at top level. Use `configure_package()` and `pre_*`/`post_*` functions. `tools/pkgcheck` enforces this.
- Reference other packages with `$(get_install_dir <pkg>)` / `$(get_build_dir <pkg>)`, never absolute host paths.
- Shell scripts: bash, 2-space indentation in `package.mk`, `set -e` where appropriate. The build system already sets `LC_ALL=C`.
- Large runtime payloads are not committed. Build them at image-build time or
  download them at runtime; verify SHA256 and pin versions/digests.

## Plugin runtime: launching Steam from Kodi

- The Kodi entry point is `source/default.py` inside the addon.
- To launch a full-screen external program, do not run it directly inside Kodi. Use the existing helper:
  1. `externalhelper.utils.run_external_program(...)` calls `systemd-run -u kodi-run-external`.
  2. `kodi-run-external` stops Kodi, starts a Wayland kiosk session (`seatd` + `cage` + `kanshi`), runs the program, then restarts Kodi.
  3. The helper sets `KODI_EXTERNAL_SESSION=wayland` and passes the Kodi audio device via `KODI_AUDIO_DEVICE` (translated to ALSA/PULSE/PIPEWIRE); the launcher must map that into the container/Steam runtime.
- The container/Steam process must receive `WAYLAND_DISPLAY`, `XDG_RUNTIME_DIR`,
  `/dev/dri`, `/dev/input`, `/dev/uinput`, `/dev/hidraw*`, and the audio socket.
- In shell scripts, load the addon environment with `. /etc/profile` and `oe_setup_addon <addon-id>` (see `packages/addons/script/steamlink/source/bin/steamlink-start.sh`).
- Declare cross-addon Python dependencies with `PKG_ADDON_REQUIRES` (e.g. `tools.flatpak:0.0.0`, `tools.externalhelper:0.0.0`) so Kodi loads their `resources/lib` modules.

## Graphics, audio, input requirements

- Steam/Proton requires Vulkan. The Generic devices currently set `VULKAN="no"` (`projects/Generic/devices/*/options`). For a Steam image set `VULKAN="vulkan-loader"` and ensure `GRAPHIC_DRIVERS` includes the target GPU (`radeonsi` for the 780M; `iris` for Intel Arc; `nvidia-ng` only if applicable). Without Vulkan, Proton/DXVK fails.
- The default Generic device is headless (`DISPLAYSERVER="no"`, `OPENGLES="mesa"`); the externalhelper `cage` session provides the display for Steam. Wayland/X11 variants are not required for this pattern.
- 32-bit graphics/Vulkan drivers are not in the host image. The container/Flatpak runtime must provide the 32-bit Mesa/Vulkan stack; do not add host lib32 drivers.
- Audio: `PULSEAUDIO_SUPPORT="yes"` is the default (`distributions/LibreELEC/options`). Kodi exposes an ALSA/PULSE/PIPEWIRE device; the external launcher receives it as `KODI_AUDIO_DEVICE` and must map it into the runtime. Target PCM, Dolby Digital, and DTS; do not promise TrueHD/Atmos on AMDGPU.
- Input: the kernel config already has `CONFIG_INPUT_UINPUT=y`, `CONFIG_HIDRAW=y`, `CONFIG_UHID=y`, `CONFIG_JOYSTICK_XPAD=m`, `CONFIG_USB_HIDDEV=y`. The container needs device access for Steam Input.
- CEC: x86 has no native HDMI-CEC. Use a Pulse-Eight USB-CEC adapter with Kodi/libCEC; Auto Power On / WoL / RTC is the fallback.
- Flatpak/container namespaces: `USER_NS`, `SECCOMP`, `FUSE_FS`, `OVERLAY_FS` must be available (`FUSE_FS=m`, `OVERLAY_FS=m`, so the modules must be loadable).

## Build and verify

Run from the repository root. Defaults: `PROJECT=Generic`, `DEVICE=Generic`, `ARCH=x86_64`, `DISTRO=LibreELEC`.

```sh
scripts/checkdeps                      # host dependencies
make image                             # build image (scripts/image mkimage)
make                                   # release build (scripts/image release)
make clean                             # remove build dirs + stamps
make distclean                         # also remove build root + ccache

PROJECT=Generic DEVICE=Generic ARCH=x86_64 make image
PROJECT=Generic DEVICE=Generic ARCH=x86_64 scripts/create_addon steam
scripts/build toolchain                # build the target toolchain (no `make toolchain`)
scripts/build <package>                # one package (target by default)
scripts/build --all <package>          # all configured build dirs
scripts/build quilt:host               # required before refresh-patches

tools/pkgcheck <package>               # static package checks (late binding, duplicate funcs, ...)
tools/find_build_warnings.sh           # scan build logs for warnings/errors
tools/refresh-patches <package>        # update patches/ (not for PKG_NO_REFRESH_PATCHES)
tools/update-pkg <package> <version>   # bump a package
tools/update-scan                      # find upstream updates
```

Addon output: `target/addons/<ADDON_VERSION>/<DEVICE|PROJECT>/<ARCH>/<addon-id>/`.

The LibreELEC toolchain is 64-bit x86_64 and cannot produce i386 binaries. Build
the 32-bit rootfs/container outside the target toolchain (e.g. `debootstrap
--arch=i386`, `docker buildx --platform linux/386`, or Valve's Steam Runtime).

## Patches and kernel changes

- Package patches live in `patches/` and are applied in filename order: `NNNN-short-description.patch`.
- Use `PKG_PATCH_DIRS` for variant-specific patch sets; respect `PKG_NO_REFRESH_PATCHES` when patches are generated by a script.
- Kernel config changes go through `tools/adjust_kernel_config menuconfig` and are committed as the regenerated `projects/<PROJECT>/linux/linux.<arch>.conf`.
- `CONFIG_IA32_EMULATION=y` is mandatory for the 32-bit Steam runtime; commit the regenerated Generic config.
- Do not hand-edit `build.*` outputs or `sources/`.

## Validation checklist

- [ ] A clean x86_64 host builds the image without an AMD GPU.
- [ ] `CONFIG_IA32_EMULATION=y` is present in the built kernel config.
- [ ] The host image contains no 32-bit Steam libraries.
- [ ] `VULKAN="vulkan-loader"` is enabled and `vulkaninfo` works on the 780M.
- [ ] 4K AV1/HEVC video plays in Kodi with hardware decode.
- [ ] HDMI audio (PCM, Dolby Digital, DTS) works through the soundbar.
- [ ] The addon builds and installs from `target/addons/...`.
- [ ] Steam starts from Kodi, renders, accepts input, plays audio, and exits.
- [ ] Kodi restarts after Steam exits.
- [ ] CEC via Pulse-Eight controls Kodi (if the adapter is present).
- [ ] `tools/pkgcheck` reports no failures for the new addon.
- [ ] No proprietary blobs, credentials, or generated artifacts are committed.
- [ ] Runtime/rootfs inputs are pinned and checksum-verified.

## Pull request checklist

- [ ] Addon is x86_64-only (`PKG_ARCH="x86_64"`) and excluded from `Generic-legacy`.
- [ ] No 32-bit libraries added to the main image; the 32-bit runtime is self-contained.
- [ ] Kernel `CONFIG_IA32_EMULATION=y` is committed for x86_64.
- [ ] Vulkan is enabled for the target device (`VULKAN="vulkan-loader"`, `GRAPHIC_DRIVERS` includes `radeonsi`).
- [ ] Steam launches through `externalhelper`/`cage` and returns cleanly to Kodi.
- [ ] Audio (PCM/DD/DTS) and input (uinput/hidraw) work inside the runtime.
- [ ] `tools/pkgcheck` is clean; addon builds; image boots; Steam starts and exits.
- [ ] No proprietary blobs, secrets, or generated artifacts are committed.
- [ ] Commit style matches the repo: `<package>: update to <version>` or `<package>: update addon (<rev>)`; topic branch, one PR per feature.

## References

- `docs/specs/steam-native-integration.md` - full spec (hardware + software).
- `projects/Generic/linux/linux.x86_64.conf` - `CONFIG_IA32_EMULATION`.
- `projects/Generic/devices/*/options` - `VULKAN`, `GRAPHIC_DRIVERS`.
- `packages/addons/script/steamlink/` - launcher pattern.
- `packages/addons/flatpak/flatpak.steamlink/` - Flatpak pattern.
- `packages/addons/tools/flatpak/`, `packages/addons/tools/externalhelper/`.
- `packages/addons/service/docker/`, `packages/addons/service/podman/`.
- `config/arch.*` - supported architectures.
- `packages/readme.md` - package/addon conventions.
- Steam client 64-bit/container announcement (2026-11-11):
  <https://store.steampowered.com/news/app/593110/view/678515036155543747>
- TechSpot 64-bit transition summary (2026-10-19):
  <https://www.techspot.com/news/114260-steam-linux-going-64-bit-valve-leans-harder.html>
- Flathub Steam Link (x86_64 only):
  <https://flathub.org/en/apps/com.valvesoftware.SteamLink>
