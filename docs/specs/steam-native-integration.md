# Spec: Native Steam HTPC Integration for LibreELEC

- **Status:** Draft
- **Date:** 2026-11-26
- **Primary source:** Claude share `df5221ca-5f19-48f5-9d48-1000d3bb39f8`
  ("Mini PC für 4K-Streaming und Gaming", conversation `60e1eb0d-dd21-4a6b-93f4-44eae633c664`)
- **Secondary source:** Claude share `b046ba4e-4011-4da4-8dd7-37c91382fc0e`
  ("Steam 32-bit Container cross-compilation", build-strategy notes only)
- **Repo:** LibreELEC.tv fork that adds native Steam integration

## 1. Summary

Build a LibreELEC-based mini-PC HTPC that runs Kodi and the native Steam client
on the same machine. The target device is an AMD Ryzen 7 8845HS / Ryzen 9 8945HS
class mini PC with Radeon 780M graphics, 32 GB dual-channel DDR5 SO-DIMM and two
M.2 slots, mounted behind a TV. Steam runs locally (not via Steam Link) in an
isolated runtime that provides the 32-bit libraries; the LibreELEC x86_64 image
stays pure 64-bit.

Realistic expectation: 4K is for video streaming and light games; current AAA
titles run at 1080p low/medium (optionally with FSR/XeSS). Native 4K AAA gaming
is not a goal for this hardware class.

## 2. Source sessions

The primary session is a hardware-selection conversation (German, 20 messages,
2026-06-29). It starts with the requirements for a TV-mounted mini PC for
streaming and gaming and then narrows down CPU/GPU, RAM, storage, HDMI audio,
pricing, and CEC/power behavior.

The secondary session is a short cross-compilation exchange. It is used only for
build-strategy notes; its repository-specific claims are corrected in section 7.

This revision replaces the earlier draft that was based only on the secondary
session.

### 2.1 Post-session updates (checked 2026-11-26)

Two Steam changes landed after the source sessions:

- **Steam Link** has been a native x86_64 Linux application since 2021-03-02
  (Flatpak `com.valvesoftware.SteamLink`, x86_64 only; current 1.4.5.326). This
  is a streaming client, not a local Steam client, so it does not replace the
  native integration.
- **Steam client** on Linux became a 64-bit application with the 2026-11-11
  stable update (beta since 2026-10-15) and now runs inside a Steam Runtime
  container. Valve still requires 32-bit libraries and graphics drivers;
  64-bit-only x86_64 systems are explicitly unsupported. The 32-bit runtime and
  kernel IA32 requirement therefore remain necessary for games and i386
  compatibility.

## 3. Target platform requirements (derived from the session)

### 3.1 Form factor

- Mini PC that can be mounted behind the TV (VESA mount desirable).
- Must fit into a living-room setup with a soundbar between the PC and the TV.
- Must run without a desktop OS; Linux/LibreELEC is the target.

### 3.2 CPU / GPU

- **Preferred:** AMD Ryzen 7 8845HS or Ryzen 9 8945HS with **Radeon 780M**
  (RDNA 3, 12 CU). Mature Linux support, good price/performance, SO-DIMM RAM.
- **Alternative:** Intel Core Ultra 7 155H with Arc iGPU, mainly if reliable
  HDMI TrueHD/Atmos passthrough is required.
- **Not preferred:** Ryzen AI 9 HX 370 with Radeon 890M. Faster (session
  benchmark: ~30-40% across the board) but typically soldered LPDDR5X (no RAM
  upgrade), higher price, and not within the 300-600 EUR budget.
- iGPU only; no discrete GPU or eGPU in the base configuration. OCuLink (UM890
  Pro) is a future option, not a requirement.

### 3.3 RAM

- 32 GB total as **2 x 16 GB DDR5-5600 SO-DIMM**, dual-channel, non-ECC.
- Both slots populated: the 780M iGPU depends on memory bandwidth; single-channel
  can cost up to ~30% GPU performance in games.
- Upgrade path: up to 96 GB (2 x 48 GB; session-confirmed on UM890 Pro); 64/96 GB
  single modules are not supported. 8 GB modules are not economical (~133 EUR
  each vs. ~296 EUR for 2 x 16 GB).
- Soldered RAM is acceptable only if performance clearly justifies it; for this
  project, upgradable SO-DIMM is preferred.

### 3.4 Storage

- Small OS SSD (~256 GB) plus a games SSD (500 GB-1 TB) with good random I/O.
- Two M.2 slots are required to keep OS and games separate.
- Session snapshot (2026-06-29, revalidate before purchase): OS ~45 EUR
  (Crucial P310 256 GB), games ~180 EUR (Samsung 990 Pro 1 TB). A 500 GB
  990 Pro was no longer regularly available; a 500 GB Crucial P310 was the
  cheaper alternative.
- A bundle (32 GB + 1 TB) plus a second OS SSD was cheaper than buying barebone,
  RAM, and SSDs separately (~604-615 EUR vs. ~942 EUR in the session snapshot).

### 3.5 Video decode and display

- Hardware decode of 4K AV1, HEVC/H.265, and H.264 (the session wrote "HEC",
  meaning HEVC).
- HDMI 2.0a/2.1 output with 4K60, HDR10, and Dolby Vision passthrough.
- Kodi is the media frontend; Steam is launched as an external full-screen
  program.

### 3.6 Audio (HDMI and soundbar)

- Audio must go over HDMI because the soundbar is inline:
  `Mini PC -> HDMI IN soundbar -> HDMI OUT (ARC) -> TV`.
- The Radeon 780M supports HDMI audio (PCM, Dolby Digital/AC-3, DTS) on Linux
  with the AMDGPU driver.
- Session finding: TrueHD, DTS-HD MA, and Atmos are unreliable on AMDGPU/Linux;
  Intel Arc is the safer choice if those formats are mandatory.
- The session's soundbar (Teufel Cinebar 11 Mk2 2020) supports Dolby Digital and
  DTS but not Atmos/DTS:X, so the AMD 780M limitation is irrelevant for this
  setup.
- Requirement: stereo/5.1 PCM and Dolby Digital/DTS passthrough must work from
  Kodi and from Steam.

### 3.7 Input, CEC, and power

- x86 mini PCs have no native HDMI-CEC. Use a **Pulse-Eight USB-CEC adapter**
  (~35-40 EUR) with Kodi/libCEC for TV-remote control and power behavior.
- The UM890 Pro supports Auto Power On, Wake-on-LAN, PXE boot, and RTC boot in
  the BIOS. Wake-on-LAN/auto-power is the fallback when CEC wake is not
  available.
- Steam Input requires `/dev/uinput`, `/dev/hidraw*`, and `/dev/input` access
  inside the runtime.

### 3.8 Connectivity

- 2.5 GbE, Wi-Fi 6, Bluetooth 5.2 as a practical baseline.
- USB4 and/or OCuLink are optional; OCuLink enables a future eGPU.
- VESA mount is required for the behind-TV use case; the Beelink SER8 lacks it,
  the Minisforum UM890 Pro and ASUS NUC 14 Pro have it.

### 3.9 Linux support

- AMD 780M: mature Mesa/AMDGPU support; preferred for LibreELEC.
- Intel Arc: good support, better HDMI HD-audio passthrough.
- Radeon 890M (Strix Point): needs a newer kernel/Mesa; soldered RAM and higher
  price make it a poor fit for this project.

### 3.10 Budget

- Target: 300-600 EUR.
- Session snapshot: UM890 Pro bundle (32 GB/1 TB) + OS SSD ~615 EUR; SER8 bundle
  + OS SSD ~604 EUR; separate purchase ~942 EUR. Treat these as indicative and
  revalidate before buying.

### 3.11 Hardware candidates

| Model | CPU | iGPU | RAM | VESA | OCuLink | Notes |
|---|---|---|---|---|---|---|
| Beelink SER8 | Ryzen 7 8845HS / 8745HS | Radeon 780M | 2 x DDR5-5600 SO-DIMM | No | No | Best value; no VESA mount |
| Minisforum UM890 Pro | Ryzen 9 8945HS | Radeon 780M | 2 x DDR5-5600 SO-DIMM, up to 96 GB | Yes | Yes | Recommended for behind-TV |
| Minisforum UM790 Pro | Ryzen 9 7940HS | Radeon 780M | 2 x DDR5 SO-DIMM | Yes | ? | Previous generation; OCuLink only on UM790 XTX |
| ASUS NUC 14 Pro | Core Ultra 7 155H | Intel Arc | 2 x DDR5 SO-DIMM | Yes | No | Better HDMI HD audio; alternative |

`?` = not confirmed in the session; verify before purchase.

## 4. Software requirements

- **OS:** LibreELEC `Generic` / `x86_64` (Kodi frontend).
- **Steam:** official proprietary client, launched locally from Kodi; not Steam
  Link streaming. Since the 2026-11-11 stable update the Linux client is 64-bit
  and runs in a Steam Runtime container, but Valve still requires 32-bit
  libraries and graphics drivers; 64-bit-only x86_64 is unsupported.
- **Isolation:** the Steam runtime (including its 32-bit libraries) lives in an
  addon/container/runtime; the host image contains no 32-bit libraries.
- **Kernel:** `CONFIG_IA32_EMULATION=y` on x86_64; a container shares the host
  kernel, so this is mandatory.
- **Graphics:** `VULKAN="vulkan-loader"` and the target GPU in
  `GRAPHIC_DRIVERS` (for the 780M: `radeonsi`).
- **Audio:** PulseAudio/ALSA with HDMI passthrough; bridge Kodi's audio device
  into the Steam runtime.
- **Display:** external Wayland session via `externalhelper` (`seatd` + `cage` +
  `kanshi`); Steam runs full-screen and returns to Kodi.
- **CEC:** Kodi/libCEC with a Pulse-Eight USB-CEC adapter (hardware option).
- **Storage:** runtime and games on the writable `/storage` partition, not in the
  read-only system image.

## 5. Architecture

### 5.1 Runtime model

Steam runs in a self-contained runtime (container or chroot-like environment).
Valve's own client now uses a Steam Runtime container (`steamrt3c`), which is the
upstream reference; the project can reuse it or ship a custom 32-bit runtime.
Either way the host kernel is shared, so the kernel must support 32-bit x86
execution, and the runtime must provide:

- 32-bit dynamic loader and glibc,
- 32-bit X11/Wayland client libraries,
- 32-bit Mesa/Vulkan loader and vendor ICDs,
- the 64-bit Steam client and its dependencies (current stable),
- 32-bit game/Proton dependencies for the Steam catalogue.

### 5.2 Display and session

1. The Kodi addon calls `externalhelper.utils.run_external_program(...)`.
2. `kodi-run-external` stops Kodi and starts the Wayland kiosk session
   (`seatd` + `cage` + `kanshi`).
3. The Steam runtime is launched inside that session.
4. When Steam exits, Kodi is restarted.

The runtime receives the Wayland socket (and/or XWayland display), GPU devices
(`/dev/dri`), input devices, and audio access.

### 5.3 Storage and packaging

- Addon directory: `packages/addons/script/steam/` (launcher) or
  `packages/addons/flatpak/flatpak.steam/` (Flatpak wrapper).
- Runtime payload: under the addon directory or `/storage` (for example
  `/storage/steam/...`), never in the read-only system image.
- Large payloads are built or downloaded, not committed.

### 5.4 Integration options

**Option A - 32-bit container (primary, matches the user's existing work)**

- Package a 32-bit rootfs (Debian/Ubuntu i386, Steam Runtime, or a pinned
  container image).
- Start it with Docker/Podman (both addons exist), systemd-nspawn (if available),
  or bwrap from the addon.
- Bind-mount GPU, input, audio, and Wayland sockets.
- Pros: explicit isolation, full control over 32-bit libraries.
- Cons: maintain the 32-bit rootfs; 32-bit GPU drivers must match the host
  kernel driver; larger footprint.

**Option B - Flatpak addon (repository-recommended fallback)**

- Reuse `packages/addons/tools/flatpak/` and `externalhelper`.
- Add `packages/addons/flatpak/flatpak.steam/` with
  `appid='com.valvesoftware.Steam'`.
- Pros: least custom runtime maintenance; existing LibreELEC plumbing.
- Cons: depends on Flatpak/Flathub; still needs kernel IA32 emulation.

**Option C - Valve Steam Runtime / steamrt3c (upstream-aligned)**

- Use Valve's Steam Runtime container (the client now runs in `steamrt3c`).
- Launch it through `externalhelper`; it still needs host IA32 and 32-bit
  graphics drivers.
- Pros: closest to upstream; Valve maintains the client environment.
- Cons: less control; the container expects a desktop-ish host; still a
  self-contained runtime.

**Decision:** keep Option A as the implementation baseline (matches the user's
existing work), but evaluate Option C (`steamrt3c`) as the upstream-aligned
alternative before finalizing. Keep Option B as the fallback. Do not add `lib32`
to the main image and do not use `PKG_ARCH="i386"`.

## 6. Repository-specific constraints

- Supported `ARCH` values are only `x86_64`, `aarch64`, `arm`
  (`config/arch.*`). There is no 32-bit x86 target.
- LibreELEC x86_64 is pure 64-bit: gcc/binutils use `--disable-multilib`; no
  `lib32` in the image.
- Generic kernel currently has
  `# CONFIG_IA32_EMULATION is not set`
  (`projects/Generic/linux/linux.x86_64.conf:691`).
- `PKG_ARCH` gates packages by target; use `PKG_ARCH="x86_64"` for Steam addons.
- Addon ID is `${PKG_SECTION//\//.}.${PKG_NAME}`; without `PKG_SECTION` it is
  `PKG_NAME`.
- `PKG_ADDON_PROJECTS` is matched against `DEVICE` or `PROJECT`; use
  `any !Generic-legacy` plus the `PKG_ARCH` gate.
- Never commit `build.*/`, `sources/`, `target/`, `.libreelec/`, proprietary
  Steam binaries, credentials, or tokens.

## 7. Build strategy

### 7.1 Host

- 64-bit Intel or AMD CPU; no AMD GPU required to compile.
- Run `scripts/checkdeps` from the repository root.
- Enough disk space for the toolchain, build tree, and runtime payload.

### 7.2 LibreELEC image

```sh
PROJECT=Generic DEVICE=Generic ARCH=x86_64 make image
```

The build system cross-compiles for `x86_64`; the host architecture does not
need to match the GPU vendor.

### 7.3 Toolchain

```sh
scripts/build toolchain
```

There is no `make toolchain` target. The LibreELEC toolchain is a 64-bit x86_64
toolchain and cannot produce 32-bit i386 binaries.

### 7.4 32-bit runtime/rootfs

Build the 32-bit userspace outside the LibreELEC target toolchain, for example:

- a pinned `i386` container image built with `docker buildx`/`podman build`
  (`--platform linux/386`),
- `debootstrap --arch=i386` into a rootfs tarball, or
- Valve's Steam Runtime as a prebuilt runtime.

Pin the base image digest and package versions; publish a SHA256 for the
resulting rootfs. Build it at image-build time or download it at runtime; do not
commit it.

### 7.5 Addon

```sh
PROJECT=Generic DEVICE=Generic ARCH=x86_64 scripts/create_addon steam
```

The addon must declare `PKG_ARCH="x86_64"` and
`PKG_ADDON_PROJECTS="any !Generic-legacy"` (or a stricter `Generic`-only list).

### 7.6 Corrections to the secondary session

| Claim in the cross-compilation session | Verdict | Repository fact |
|---|---|---|
| LibreELEC always cross-compiles; no AMD PC needed to build | Correct in spirit | The build system uses a target toolchain and builds from source; runtime testing still needs a Vulkan GPU. |
| `PROJECT=Generic ARCH=x86_64 make image` | Correct | `make image` runs `scripts/image mkimage`. |
| `BUILD_PERIODIC=no` is a build switch | Incorrect | `BUILD_PERIODIC` only controls version naming in `scripts/image`. |
| `make toolchain` | Incorrect | Use `scripts/build toolchain`; there is no such Make target. |
| `PKG_ARCH="x86_64 i386"` / `i386` | Incorrect | Only `x86_64`, `aarch64`, `arm` exist. |
| Multilib/i686 toolchain is available | Incorrect | gcc/binutils are `--disable-multilib`; no lib32. |
| Build system downloads the toolchain | Misleading | It builds the toolchain from source; it downloads source archives. |

## 8. Validation and acceptance criteria

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

## 9. Risks and open questions

1. **Kernel 32-bit support:** `CONFIG_IA32_EMULATION` is currently disabled; this
   is a hard blocker for 32-bit games/Proton components. The 64-bit Steam client
   (since 2026-11-11) does not remove this requirement: Valve still requires
   32-bit libraries and graphics drivers.
2. **32-bit GPU drivers:** Mesa 32-bit and NVIDIA 32-bit userspace must match the
   kernel driver. NVIDIA is the riskiest case; the primary target is AMD 780M
   with Mesa.
3. **Rootfs source and licensing:** Debian/Ubuntu i386 vs. Steam Runtime vs.
   container image; each has different licensing and update behavior.
4. **Container runtime choice:** Docker, Podman, systemd-nspawn, or bwrap; Docker
   and Podman addons already exist in the tree.
5. **Display path:** native Wayland vs. XWayland vs. gamescope; Steam's behavior
   differs per game and Proton version.
6. **Audio:** PulseAudio socket bridging vs. PipeWire; Kodi's device selection
   must be preserved; TrueHD/Atmos is not reliable on AMDGPU.
7. **Input:** Steam Input needs `/dev/uinput` and hidraw access; permissions and
   udev rules must be handled inside the runtime.
8. **Footprint:** a full 32-bit rootfs plus Steam/Proton can be large; storage
   location and cleanup must be defined.
9. **Updates:** how Steam, Proton, and the runtime are updated independently of
   the LibreELEC image.
10. **Hardware availability/pricing:** the session prices are a 2026-06-29
    snapshot and must be revalidated.
11. **Existing implementation:** the user's 32-bit container code has not yet
    been reviewed against this spec.

## 10. Out of scope

- ARM/aarch64 Steam via FEX/box64 + Wine.
- 32-bit libraries in the main image.
- Steam Link streaming.
- Native 4K AAA gaming on the 780M.
- Non-Generic projects/devices in the first iteration.

## 11. Review notes

Verified against this checkout (2026-11-26):

- `projects/Generic/linux/linux.x86_64.conf:691` disables `CONFIG_IA32_EMULATION`.
- All `projects/Generic/devices/*/options` set `VULKAN="no"`; `Generic` is
  `DISPLAYSERVER="no"`, `OPENGLES="mesa"`, and already lists `radeonsi` in
  `GRAPHIC_DRIVERS`.
- `packages/addons/tools/flatpak/`, `packages/addons/tools/externalhelper/`,
  `packages/addons/flatpak/flatpak.steamlink/`, and
  `packages/addons/script/steamlink/` exist.
- `packages/addons/service/docker/` and `packages/addons/service/podman/` exist.
- No native Steam addon or 32-bit container code exists in this checkout; only
  Steam Link.
- There is no `make toolchain` target; use `scripts/build toolchain`.
- `BUILD_PERIODIC` only affects `DISTRO_VERSION` naming in `scripts/image`.
- Kernel options for input, FUSE, overlayfs, user namespaces, and seccomp are
  present as stated in `AGENTS.md`.

Verified against the primary session:

- Hardware candidates and the 780M/890M/Arc positioning.
- 32 GB as 2 x 16 GB DDR5-5600 SO-DIMM, 96 GB maximum (2 x 48 GB), dual-channel
  impact.
- M.2 OS + games SSD split.
- AMDGPU audio: PCM/DD/DTS reliable; TrueHD/DTS-HD MA/Atmos unreliable.
- Cinebar 11 Mk2 (2020): DD/DTS, no Atmos/DTS:X; HDMI 2.0a, 4K60, HDR10, DV, CEC.
- x86 has no native CEC; Pulse-Eight USB-CEC is the workaround.
- UM890 Pro: Auto Power On, WoL, PXE, RTC.

Post-session verification (2026-11-26):

- Steam Link for Linux has been x86_64 since 2021-03-02 (Flatpak, x86_64 only).
- Steam client stable 2026-11-11 is 64-bit and runs in a Steam Runtime container.
- Valve's requirements still mandate 32-bit libraries and graphics drivers;
  64-bit-only x86_64 is unsupported.
- Therefore the kernel IA32 and 32-bit runtime requirements remain.

Open verification:

- UM790 Pro OCuLink (only UM790 XTX confirmed in the session).
- Current prices and bundle availability (session snapshot 2026-06-29).
- BIOS auto-power/WoL behavior on the exact model.
- 32-bit Mesa/RADV and (if used) NVIDIA userspace.
- The user's existing 32-bit container implementation.

## 12. References

- Primary session: <https://claude.ai/share/df5221ca-5f19-48f5-9d48-1000d3bb39f8>
- Secondary session: <https://claude.ai/share/b046ba4e-4011-4da4-8dd7-37c91382fc0e>
- Steam client 64-bit/container announcement (2026-11-11):
  <https://store.steampowered.com/news/app/593110/view/678515036155543747>
- TechSpot summary of the 64-bit transition (2026-10-19):
  <https://www.techspot.com/news/114260-steam-linux-going-64-bit-valve-leans-harder.html>
- Flathub Steam Link (x86_64 only):
  <https://flathub.org/en/apps/com.valvesoftware.SteamLink>
- Steam Link Linux x86_64 announcement (2021-03-02):
  <https://steamcommunity.com/app/353380/discussions/10/3106892760562833187>
- `AGENTS.md` in this repository.
- `packages/addons/script/steamlink/` (launcher pattern).
- `packages/addons/flatpak/flatpak.steamlink/` (Flatpak pattern).
- `packages/addons/tools/flatpak/` and `packages/addons/tools/externalhelper/`.
- `projects/Generic/linux/linux.x86_64.conf` (`CONFIG_IA32_EMULATION`).
- `config/arch.*` (supported architectures).
- `packages/readme.md` (package/addon conventions).
