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
TODO - more details and links later:
- Download k3 kernel deb package and upgrade k3 kernel
- Download aosp cf-k3-v1.0 image zip file
- Download cvd-host package cf-k3-v1.0 version.
- Build, init and run with cuttlefish host container
  - TODO, cf-build.sh, cf-init.sh, cf-run.sh, cf-stop.sh

## Under the Hood - AOSP image

### Overview

Customized AOSP riscv64 phone image for [Cuttlefish](https://source.android.com/docs/devices/cuttlefish), running as a guest VM on a real RISC-V host (SpaceMiT K3).

Equivalent to Google's published [`aosp_cf_riscv64_phone-userdebug` CI builds](https://ci.android.com/builds/submitted/15150359/aosp_cf_riscv64_phone-userdebug/latest), with two deliberate departures:

1. **Kernel upgraded to 6.10+** for RISC-V AIA (IMSIC/APLIC) support — the AOSP-checked-in prebuilt is stale at 6.8.
2. **Guest-side keymint + gatekeeper** as the riscv64 default (`guest_keymint_insecure`, `guest_gatekeeper_insecure`), matching x86_64/arm64. AOSP currently hardcodes riscv64 to host-side as a placeholder.


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

Output: `out/dist/aosp_cf_riscv64_phone-img-jsun.zip` (~830 MB). Deploy by extracting into the cuttlefish product dir on the host (`cf-data/product/`), the same way Google's CI img zip is deployed.

## Under the Hood - crosvm
TODO 

## Under the Hood - u-boot
TODO 

## Under the Hood - Android Cuttlefish
TODO 

