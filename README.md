# aosp-cuttlefish-riscv64

This is an umbrella project that aims to run AOSP on cuttlefish on a RISC-V board.  Specifically in near term, we aims to run AOSP android16-qpr2 on SpaceMIT k3 board.

Since this is first attempt in the world, it needs to tie many different pieces together. This **umbrella repository** serves as a central pointer to all the scattered pieces, including:

- AOSP image build + mainline kernel upgrade
- [Android cuttlefish](https://github.com/monkey-jsun/android-cuttlefish) build for RISCV64
  - [crosvm](https://github.com/monkey-jsun/crosvm) porting for RISCV64 and K3
  - [u-boot](https://github.com/monkey-jsun/u-boot) porting for RISCV64 and K3
- [Cuttlefish host container](https://github.com/monkey-jsun/cuttlefish-host-container) extension for running the image
- [K3 host kernel](https://github.com/monkey-jsun/linux-6.18-k3) support

**Status:** Milestone v1.0 (WebRTC) achieved 2026-05-31 — phone screen visible in a browser via WebRTC streaming on the K3 host.

## For the Impatient
- Download k3 kernel deb package and upgrade k3 kernel
- Download [aosp cf-k3-v1.0 image zip](https://github.com/monkey-jsun/aosp-cuttlefish-riscv64/releases/download/cf-k3-v1.0/aosp_cf_riscv64_phone-img-cf-k3-v1.0.zip).
- Download [cvd-host package cf-k3-v1.0 version](https://github.com/monkey-jsun/android-cuttlefish/releases/download/cf-k3-v1.0/cvd_host_package_riscv64-cf-k3-v1.0.tar.gz).
- Build, init and run with cuttlefish host container:
  ```sh
  git clone -b riscv64 https://github.com/monkey-jsun/cuttlefish-host-container
  cd cuttlefish-host-container

  # one-time
  docker build . -t cf-host

  # whenever artifacts change
  ./cf-init.sh -H <path-to>/cvd_host_package_riscv64-cf-k3-v1.0.tar.gz \
               -P <path-to>/aosp_cf_riscv64_phone-img-cf-k3-v1.0.zip

  ./cf-run.sh

  # Connect (from any host that can reach the riscv host):
  #   WebRTC:  https://<riscv host ip>:8444
  #   adb:     adb connect <riscv host ip>:6520

  ./cf-stop.sh
  ```

## Under the Hood - AOSP image

### Overview

Customized AOSP riscv64 phone image for [Cuttlefish](https://source.android.com/docs/devices/cuttlefish), running as a guest VM on a real RISC-V host (SpaceMiT K3).

Equivalent to Google's published [`aosp_cf_riscv64_phone-userdebug` CI builds](https://ci.android.com/builds/submitted/15150359/aosp_cf_riscv64_phone-userdebug/latest), with two deliberate departures:

1. **Kernel upgraded to 6.10+** for RISC-V AIA (IMSIC/APLIC) support — the AOSP-checked-in prebuilt is stale at 6.8.
2. **Guest-side keymint + gatekeeper** as the riscv64 default (`guest_keymint_insecure`, `guest_gatekeeper_insecure`), matching x86_64/arm64. AOSP currently hardcodes riscv64 to host-side as a placeholder.


### Deviations (4 patch series; sources in [`patches/`](patches/))

Kernel:

1. **AIA enablement** — new `riscv64_aia.fragment` setting `CONFIG_RISCV_INTC/APLIC/IMSIC=y` for the cuttlefish virtual-device build, plus `CONFIG_WERROR` disabled (matches arm).
2. **Build-fix workarounds** for rolling-drift on android-mainline-riscv64 (Rust toolchain pin bump, `kernel_riscv64.outs` override). Expected to disappear when upstream catches up.

AOSP cuttlefish:

3. **secure_hals default for riscv64** — `assemble_cvd/flags.cc` flips to guest-side keymint/gatekeeper, closing the existing `TODO: schuffelen` block.
4. **Debug-userland scaffolding** — SELinux permissive + multi-phase init pause shell. Inert without `androidboot.debug_pause_console=hvc20`; normal boot unchanged.

Upstream submission status: pending.


### Reproduction recipe

Build host: Linux x86_64 (AOSP does not yet support riscv64 as a build host). Tested with the standard AOSP build prerequisites.

#### 1. Clone source trees

```sh
# AOSP guest tree (~150 GB on disk, hours to sync)
mkdir android16-qpr2-release && cd android16-qpr2-release
repo init --partial-clone --no-use-superproject \
    -b android16-qpr2-release \
    -u https://android.googlesource.com/platform/manifest
repo sync -j8 -c --no-tags --optimized-fetch --retry-fetches=3
cd ..

# Kernel tree (~25 GB) — pinned via the snapshot manifest in this repo so
# all ~50 kernel-manifest projects are at the exact SHAs v1.0 was built against
mkdir kernel-android-mainline-riscv64 && cd kernel-android-mainline-riscv64
repo init -u https://android.googlesource.com/kernel/manifest \
    -b common-android-mainline-riscv64
cp ../aosp-cuttlefish-riscv64/manifests/kernel-snapshot.xml .repo/manifests/
repo init -m kernel-snapshot.xml
repo sync -j8 -c --no-tags --optimized-fetch --retry-fetches=3
cd ..
```

#### 2. Apply kernel patches

```sh
cd kernel-android-mainline-riscv64/common
git am ../../aosp-cuttlefish-riscv64/patches/kernel-common/*.patch
cd ../common-modules/virtual-device
git am ../../../aosp-cuttlefish-riscv64/patches/kernel-virtual-device/*.patch
cd ../../..
```

#### 3. Build kernel + vendor modules

`--nopahole_from_sources` works around a kleaf default that expects a `//common:pahole` target our pinned common doesn't define. `-- --destdir=...` is required by current kleaf (replaces old `--dist_dir=`).

```sh
cd kernel-android-mainline-riscv64
tools/bazel run //common:kernel_riscv64_dist --nopahole_from_sources \
    -- --destdir=out/kernel_riscv64_dist
tools/bazel run //common-modules/virtual-device:virtual_device_riscv64_dist --nopahole_from_sources \
    -- --destdir=out/virtual_device_riscv64_dist
cd ..
```

Wall-clock for a clean first build: ~17 min.

#### 4. Drop kernel into AOSP cuttlefish prebuilts

```sh
SRC=kernel-android-mainline-riscv64/out/virtual_device_riscv64_dist
DEST=android16-qpr2-release/device/google/cuttlefish_prebuilts/kernel/mainline-riscv64

cp $SRC/Image      $DEST/kernel-mainline
cp $SRC/Image.gz   $DEST/kernel-mainline-gz
cp $SRC/Image.lz4  $DEST/kernel-mainline-lz4
cp $SRC/System.map $DEST/

# system_dlkm/ partition contents (extracted from staging archive).
rm -rf $DEST/system_dlkm
mkdir -p $DEST/system_dlkm
TMP=$(mktemp -d)
tar xzf $SRC/system_dlkm_staging_archive.tar.gz -C $TMP
cp $TMP/flatten/lib/modules/*.ko $DEST/system_dlkm/

# Top-level: vendor modules + the 7 virtio modules. Vendor modules go ONLY
# at top-level. GKI modules go ONLY in system_dlkm/. The 7 virtio modules
# are the exception — they live in BOTH (vendor_ramdisk/first-stage-init
# needs them at top-level; the system_dlkm partition needs them in subdir).
rm -f $DEST/*.ko
KEEP="virtio_blk.ko virtio_console.ko virtio_pci.ko virtio_pci_legacy_dev.ko virtio_pci_modern_dev.ko virtio_balloon.ko vmw_vsock_virtio_transport.ko"
for ko in $SRC/*.ko; do
    name=$(basename $ko)
    if [ ! -f $TMP/flatten/lib/modules/$name ]; then
        cp $ko $DEST/                  # vendor module — top-level only
    elif echo " $KEEP " | grep -q " $name "; then
        cp $ko $DEST/                  # virtio module — both locations
    fi
done
rm -rf $TMP
```

#### 5. Apply AOSP cuttlefish patches

```sh
cd android16-qpr2-release/device/google/cuttlefish
git am ../../../aosp-cuttlefish-riscv64/patches/aosp-cuttlefish/*.patch
cd ../../..
```

#### 6. Build AOSP guest image

```sh
cd android16-qpr2-release
source build/envsetup.sh
lunch aosp_cf_riscv64_phone-trunk_staging-userdebug
m dist
```

Wall-clock for a clean first build: ~2–4 h on a decent x86_64 host; incremental rebuilds are minutes.

Output: `out/dist/aosp_cf_riscv64_phone-img-jsun.zip` (~830 MB). Deploy by extracting into the cuttlefish product dir on the host (`cf-data/product/`), the same way Google's CI img zip is deployed.

## Under the Hood - crosvm

### Overview

Custom RISC-V build of [crosvm](https://crosvm.dev/), Google's Rust-based VMM.
Used by Cuttlefish as the VM manager (`launch_cvd` invokes crosvm to create
the guest VM via KVM).

Fork: <https://github.com/monkey-jsun/crosvm> — branch `riscv64`, pinned at
tag `cf-k3-v1.0`. Upstream Google crosvm has initial RISC-V scaffolding but
is incomplete for actually booting Android.

Deviations from upstream Google crosvm (10 patches):

1. **FDT generation** queries ISA extensions and MMU type from KVM rather
   than hardcoding; adds serial nodes, stdout-path, FDT-dump support, and
   `android_fstab` entries.
2. **Memory layout** places FDT/initrd above kernel BSS; enlarges the MMIO
   region from 1 MB to 32 MB to fit cuttlefish's device set.
3. **BIOS loading** support (cuttlefish uses U-Boot as BIOS on riscv64).
4. **APLIC enablement** in the KVM in-kernel AIA path; caps `nr_sources` to
   the per-vsfile id count reported by KVM (kernel constraint).
5. **SBI exit logging** for unhandled SBI calls (debug aid).

All patches have been submitted upstream via Gerrit
([CL URLs](https://chromium-review.googlesource.com/q/owner:jsun%40junsun.net+project:crosvm/crosvm)).


### Reproduction recipe

Build host: Linux riscv64. Toolchain: Rust 1.88+, plus `build-essential`,
`pkg-config`, `libcap-dev`, `libwayland-dev`, `protobuf-compiler`.

#### 1. Clone the fork

```sh
git clone https://github.com/monkey-jsun/crosvm.git
cd crosvm
git checkout cf-k3-v1.0
```

#### 2. Build

```sh
cargo build --release \
    --features android-sparse,composite-disk,fs_runtime_ugid_map,audio_aaudio,android_display,android_display_stub,libaaudio_stub
```

Excluded features:

- `media` — v4l2r bindgen fails on riscv64 (clang can't handle the riscv64
  ABI for `videodev2.h`).
- `gdb` — causes vCPU run-loop interference (RCU stall) under riscv64 KVM.

Wall-clock for a clean build: ~30 min.

Output: `target/release/crosvm` (~12 MB).

## Under the Hood - u-boot

### Overview

Custom riscv64 build of U-Boot for Cuttlefish. AOSP builds bootloader
binaries per VM-manager + arch combination, but does not currently produce
one for `riscv64 + crosvm`, so we build it from a fork.

Fork: <https://github.com/monkey-jsun/u-boot> — branch `riscv64`, pinned at
tag `cf-k3-v1.0`. Base: u-boot 2024.04 (via AOSP `u-boot-mainline`).

Deviations from AOSP `u-boot-mainline` (3 patches; all submitted to AOSP
Gerrit):

1. **virtio_console.c compile fix** — GCC on riscv64 rejects
   `static const size_t` at file scope when later used as an array dimension.
   Switched to `#define`.
2. **default_env.txt preboot line** — `preboot=setenv fdtaddr ${fdtcontroladdr}`.
   crosvm places the FDT at a dynamic address; U-Boot saves it as
   `${fdtcontroladdr}` then relocates to high memory, overwriting the original.
   The preboot line carries the relocated FDT address through to `boot_android`.
3. **`crosvm-riscv64.fragment`** — new config fragment that sets the same FDT
   fix via `CONFIG_PREBOOT` (used when `CONFIG_USE_DEFAULT_ENV_FILE` is off).
   Mirrors the existing `crosvm-aarch64.fragment` / `crosvm-x86_64.fragment`.

Full per-patch detail and rationale live in the fork's own build doc:
[`doc/README.cuttlefish-crosvm-riscv64`](https://github.com/monkey-jsun/u-boot/blob/riscv64/doc/README.cuttlefish-crosvm-riscv64).


### Reproduction recipe

Build host: Linux riscv64.

#### 1. Clone the fork and the AOSP sibling repos

U-Boot's build expects two AOSP sibling repos at fixed relative paths
(via symlinks `lib/libxbc/...` → `../../../bootable/libbootloader/libxbc/...`
and `scripts/dtc/...` → `../../external/dtc/...`).

```sh
mkdir uboot && cd uboot
git clone https://github.com/monkey-jsun/u-boot.git
cd u-boot && git checkout cf-k3-v1.0 && cd ..
git clone --depth 1 \
    https://android.googlesource.com/platform/external/dtc \
    external/dtc
git clone --depth 1 \
    https://android.googlesource.com/platform/bootable/libbootloader \
    bootable/libbootloader
```

#### 2. Build

```sh
cd u-boot
scripts/kconfig/merge_config.sh -m -r \
    configs/qemu-riscv64_smode_defconfig \
    qemu-minimal.fragment \
    riscv64.fragment \
    cuttlefish.fragment \
    crosvm.fragment \
    crosvm-riscv64.fragment \
    avb_unlocked.fragment
make olddefconfig
make -j$(nproc)
```

Output: `u-boot.bin` (~750 KB).

## Under the Hood - Android Cuttlefish

### Overview

Custom riscv64 build of Google's [Cuttlefish](https://source.android.com/docs/devices/cuttlefish), the AOSP virtual Android device tooling. Used here to host the AOSP riscv64 guest image on a SpaceMIT K3 board.

Fork: <https://github.com/monkey-jsun/android-cuttlefish> — branch `dev/riscv64`, pinned at tag `cf-k3-v1.0`. Upstream android-cuttlefish ships build flows only for x86_64 and arm64.

At a high level, the fork adds **riscv64 support**, wires **crosvm** as the riscv64 VM manager, and fills in **WebRTC streaming**:

1. **riscv64 build infrastructure** — host-side and container-based build flows under `tools/buildutils/`, plus a new `cvd_host_package_riscv64` bazel target, so the riscv64 host-side artifacts exist at all.
2. **crosvm wiring** — `assemble_cvd` defaults for riscv64 (guest-side keymint + gatekeeper, bootconfig handling) so crosvm boots the AOSP guest cleanly.
3. **WebRTC operator** — missing operator-side JS assets added to the host `.deb` so the WebRTC client streams end-to-end.

Some of the above are already upstream ([merged PRs](https://github.com/google/android-cuttlefish/pulls?q=is%3Apr+author%3Amonkey-jsun+is%3Amerged)); others are pending ([open PRs](https://github.com/google/android-cuttlefish/pulls?q=is%3Apr+author%3Amonkey-jsun+is%3Aopen)) and will be re-worked for upstream. More are coming.


### Reproduction recipe

Build host: Linux riscv64 (native) or Linux x86_64 (via qemu-user binfmt; slower). Toolchain: just `docker` for the default container build, or bazel + clang-19 + rustc on the host for the alternative host build.

#### 1. Clone the fork

```sh
git clone https://github.com/monkey-jsun/android-cuttlefish.git
cd android-cuttlefish
git checkout cf-k3-v1.0
```

#### 2. Build — container (default)

```sh
tools/buildutils/build-cf-riscv64.sh
```

On x86_64 hosts the script checks that `qemu-user-static` + `binfmt-support` are installed and that the `qemu-riscv64` binfmt is registered with the `C` flag.

Outputs at the repo root:

- 5 host .debs: `cuttlefish-base_1.50.0_riscv64.deb` (126 MB) + `cuttlefish-common` / `-defaults` / `-integration` / `-metrics`.
- `cvd_host_package_riscv64.tar.gz` (~225 MB).

Wall-clock for a clean first build: ~3.5 h on x86_64 under qemu (~2 h image build incl. bazel-from-source, ~1.5 h cuttlefish C++ tree); much faster on native riscv64.

#### 2'. Build — host (alternative, riscv64 only)

```sh
tools/buildutils/setup-riscv64-host.sh                          # once
tools/buildutils/build_package.sh base                           # → 5 .debs at repo root
tools/buildutils/cf-bazel-build.sh \
    //cuttlefish/package/cvd_host_riscv64:cvd_host_package_riscv64
                                                                 # → cvd_host_package_riscv64.tar.gz
```

