# Pixel 3a XL — LineageOS 24.0 (Android 17) — GROUNDWORK ONLY

An early attempt to take the **Pixel 3a XL** (`bonito`) to Android 17. It was a 23.2 (Android 16)
skeleton until 2026-09-17; the gate is the same for 16 and 17 (see *The kernel gate*), and 23.x
will freeze once 24 matures, so the target moved without anything being built in between.

> **Nothing works here yet.** No build has been attempted.
> **For a working Pixel 3a XL ROM, use the [`lineage-22.2`](../../tree/lineage-22.2) branch.**

## The kernel gate

Before any of the below: Android 16+ needs eBPF features this device's 4.9 kernel does not have, and
no 4.14+ kernel exists for its SoC. The first job is adapting the 4.14-era eBPF backports onto
`kernel/google/msm-4.9`; the device-tree work below only matters after that. Details and sources in
[ANDROID-16.md](ANDROID-16.md), *The kernel gate*.

## What exists so far

The branch, a manifest pinned to the branches that actually exist, and the 22.2 device patches
carried forward: partition sizing for baked-in GApps, schedutil/powerhint tuning, the VINTF OTA
relaxation and the build tag (the Velvet drop is the forge's `gapps` option, not a device patch).

`COMMON_OPTIONS` matches the 22.2 branch. No option has `lineage-24.0` patches in the forge yet, so
the ones that carry patches stop the build at the option check until those are derived from the
23.2 sets.

## The shape of the job

LineageOS stopped supporting this device after a stale `lineage-23.0` branch, so of the seven
projects in the manifest only **one** has an Android 17 branch:

| Project | Pinned to | Why |
|---|---|---|
| `ElmyraService` | **24.0** | Genuinely carried forward |
| `gs-common` | 23.2 | Newest available |
| `device_google_{sargo,bonito}` | 22.2 | A `23.0` branch exists but is an *ancestor* of 22.2 — abandoned 2025-08. The higher number is the older code. |
| `kernel_google_msm-4.9` | 22.2 | Newest available |
| `proprietary_vendor_google_*` | 22.2 | Newest available |

So this is an **Android 17 platform running Android 15 blobs and an Android 15 kernel** — a
two-version gap, the same `hardware/lineage/compat` shape ether-20.0 carries. Carrying those six
forward is the device-side work.

The encouraging part: this device depends on `hardware/google/*` rather than per-SoC Qualcomm CAF
trees, which is why it was picked over the LG V20 as the first Android 16+ target.

## Before the first build

`LUNCH_TARGET` is `lineage_bonito-cp2a-userdebug`: `cp2a` is the one release config
`vendor/lineage` defines on `lineage-24.0` (`vars/aosp_target_release`; 23.2 was `bp4a`, 22.2
`bp1a`). No GApps package for Android 17 is wired into the forge; `clean` and `libre` are the only
presets that can be attempted.

## More

| File | What is in it |
|---|---|
| [ANDROID-16.md](ANDROID-16.md) | The Android 16+ analysis: the kernel gate, why this device over the V20, the stale-branch trap |
| [forge/docs/porting-a-branch-bump.md](forge/docs/porting-a-branch-bump.md) | Run these checks *before* the first build |
| [forge/docs/lineage-branches.md](forge/docs/lineage-branches.md) | Which branches are alive, and the stale-branch trap |

## Building

Nothing here is expected to build yet. When it is worth trying:

```sh
PRESET=clean ./forge/bootstrap.sh
```

One command produces one image. `PRESET` names a saved set of options from `device.conf`; `clean`
is the one with no proprietary inputs, so it is the right first attempt.

## Presets

One build command produces one image. A preset is a saved selection of options — it has no
behaviour of its own. Same as the 22.2 branch; `clean` is the right first attempt.

| preset | tag | adds over `clean` |
|---|---|---|
| `clean` | `turbo-clean` | nothing — this is the baseline |
| `libre` | `turbo-libre` | `fdroid`, `fulguris`, `k9`, `termoneplus`, `kdeconnect`, `connectbot`, `linphone` |
| `full` | `turbo` | `fdroid`, `fulguris`, `gapps`, `k9`, `termoneplus`, `kdeconnect`, `root`, `connectbot`, `linphone` |

Every preset also carries the shared set, which is what makes this build look and behave the way
it does regardless of which preset you pick:

`advanced-restart` `dark-default` `google-feed-off` `home-defaults` `linux` `livedisplay-off` `minimal-home` `nav-icons` `nfc-off` `setupwizard-nag-skip` `teal-skin` `teal-wallpaper` `themed-icons`

`oem` is in no preset. `EXTRA_OPTIONS` adds an option to whichever preset you build, and every
option added that way appends its name to the tag:

```sh
EXTRA_OPTIONS=oem PRESET=full ./forge/bootstrap.sh      # tag turbo-oem
```

Set `EXTRA_OPTIONS="oem"` in `device.conf.local` (gitignored) to get it on every build from this
checkout. The pack here is the Nextbit Robin's (`OEM_ASSET_PACK=nextbit-robin`), on purpose: the
Robin boot animation, wallpapers and sounds on a Pixel. It needs the Robin stock zip
(`Ether_Stock_ROM_*.zip`) next to `device.conf`; those images are for your own phone.

## Options

Every option this device uses, and what each one does. They live in `forge/options/`, so they
work on any device rather than being wired into this tree.

| option | what it does |
|---|---|
| `advanced-restart` | Advanced restart in the power menu |
| `dark-default` | Default to dark theme |
| `fdroid` | F-Droid app store + Privileged Extension (silent installs/updates) |
| `firefox` | Firefox (Fennec F-Droid) as the browser, replacing Jelly — still available, but 320 MB staged, so no preset carries it now |
| `fulguris` | Fulguris as the browser, replacing Jelly — a WebView browser, 9 MB where Fennec stages 320 MB |
| `connectbot` | ConnectBot: an SSH client with saved hosts, keys and port forwarding |
| `linphone` | Linphone: a SIP client, for voice over data where the device has no VoLTE |
| `gapps` | Google apps: Play Store and GMS from MindTheGapps, plus Google's versions of the stock apps |
| `google-feed-off` | Google feed (-1 screen) off by default |
| `home-defaults` | Home screen defaults: no icon labels, no auto-add of new apps |
| `kdeconnect` | KDE Connect: phone <-> desktop notifications, clipboard, files, remote input |
| `linux` | On-device Linux environment (chroot + Docker): container kernel config and cgroup fixes |
| `livedisplay-off` | LiveDisplay off by default |
| `minimal-home` | Minimal home screen: hotseat only, no second page |
| `k9` | K-9 Mail (the Thunderbird for Android codebase) as the mail client |
| `nav-icons` | Nextbit Robin style nav-bar icons, drawn as scalable tintable vectors (on every preset) |
| `nfc-off` | NFC off by default |
| `oem` | Reclaimed stock-ROM boot animation, wallpapers and sounds — the Nextbit Robin's here, see *Presets* |
| `root` | Magisk baked into the boot image, so the zip flashes pre-rooted |
| `setupwizard-nag-skip` | Skip recovery/metrics/backup setup pages |
| `teal-skin` | Teal accent — fixed #009D94 Monet preset seed |
| `teal-wallpaper` | Teal-shag default wallpaper (baked into framework-res) |
| `termoneplus` | TermOne Plus terminal emulator |
| `themed-icons` | Themed (monochrome) app icons on by default |

## Device patches

31 patches across 7 upstream projects, applied at build time from
`overlay/patches/`. Nothing here is a fork: each is a single commit against the upstream tree,
replayed on every build, so upstream stays upstream and what we changed stays legible.

One patch per thing it enables.

### `device/google/bonito`

Carried forward from 22.2 unbuilt (the device tree is pinned to its 22.2 branch, so they apply as-is).

- **0001 size the partitions for what the build bakes in** — Lineage reserves 1000 MiB in product
  and 90 MiB each in system/system_ext for post-install GApps. A 24.0 build with the OEM assets and
  Syncthing-Fork stages ~4100 MiB against the 3880 MiB `google_dynamic_partitions` group with the
  full reservation, so no variant keeps it. Under `WITH_GAPPS`: product 32 MiB, system/system_ext
  16 MiB each; `WITH_FIREFOX` 128 MiB; everything else 512 MiB (~250 MiB of the group to spare).
  Predict it from a built `out/`: sum the sparse-header sizes of the system/system_ext/product/
  vendor `.img` files (content + reservation + ~4% ext4 overhead for any not built yet);
  `check_partition_sizes` reads those, not `du`.
- **0002 schedutil and powerhint tuning** — longer `down_rate_limit_us` on both clusters so the
  governor stops dropping frequency between frames, a higher top-app schedtune boost, powerhint
  floors raised to match. Smoothness, not benchmark peaks.
- **0003 do not enforce VINTF kernel requirements on OTA** —
  `PRODUCT_OTA_ENFORCE_VINTF_KERNEL_REQUIREMENTS := false`; the shipped kernel does not declare
  everything the OTA-time check expects, and the check refuses the package rather than warning.
- **0004 tag the build** — build tag in the zip filename and `ro.lineage.version`
  (`…-UNOFFICIAL-<tag>-bonito`), overridable via `TURBO_BUILD_ID` (how the forge gives each preset
  its tag). Set before the `common_full_phone` inherit, or `version.mk` never sees it.

lineage-24.0 port (the tree stays on its 22.2 branch; upstream has no 24.0 for sdm670). Each
patch is one build failure or one removed interface:

- **0005–0006** drop makefile includes 24.0 no longer has; import the Soong namespaces it added.
- **0007–0009** HIDL → AIDL for dumpstate, health.storage and lights (the LineageOS AIDL lights HAL
  replaces `hardware.google.light`; HBM hook dropped from hwc2, see `hardware/qcom/sdm845/display`).
- **0010–0013** drop `disable_configstore`, `check_dynamic_partitions`, string-typed Soong bools, and
  the deleted `vendor/lineage/config/device_framework_matrix.xml` include.
- **0014 FCM target-level 5 → 7** — 24.0 has matrices 7, 8, 2024xx–2027xx only; libvintf's
  deprecation check needs a matrix at the device's level, so 5 fails `check_vintf_compatible` with
  `Cannot find framework matrix at FCM version 5`. Coral/sunfish made the same move upstream.
- **0015 Bluetooth audio HIDL 2.0 → AIDL** and **0016 drop power.stats@1.0** — neither HIDL package
  is in any matrix ≥ 7, so `checkUnusedHals` rejects the instances. The AIDL BT audio impl brings
  its own VINTF fragment; power.stats comes back later as AIDL.
- **0017 drop the PixelLogger sepolicy** — 0005 removed `PixelLogger.mk`, which was what put
  `hardware/google/pixel-sepolicy/logger_app` (the `logger_app` type) on the sepolicy dirs; the
  device's own `logger_app.te` then fails checkpolicy with `unknown type`. Same as coral `9e807a47`.
- **0018 drop the wifi_perf_diag sepolicy** — its `wifi_logging_data_file` type also came from the
  PixelLogger dir, and no `/vendor/bin/wifi_perf_diag` ships in the blobs. Coral 24.0 has neither.
- **0019 thermal HAL from `hardware/google/pixel/thermal`** — gs-common's `thermal_hal/device.mk`
  (dropped in 0005) only packaged that HAL and named gs-common's sepolicy dir; both are done in the
  device tree now, sepolicy copied verbatim from 22.2 gs-common (coral `0d401e37`). Without it
  `pixel-sepolicy/power-libperfmgr` fails on `thermal_link_device`/`vendor_thermal_prop`. The AIDL
  thermal v3 fragment is in matrices 202504+, so `checkUnusedHals` is satisfied at level 7.
- **0020 let the platform label `/sys/class/typec`** — vendor API 202604 labels it `sysfs_typec`
  (system/sepolicy `a4f2ea842`), and `secilc` rejects a second `genfscon` on the same path at
  `precompiled_sepolicy` ("conflicting genfscon rules"). `/class/typec/usbc0` stays `sysfs_usb_c`.
  Preview the fix without a build: run the failing `secilc` line from the log with the edited
  `vendor_sepolicy.cil` substituted (it only reports the first conflict).

- **0021 drop the HIDL context hub HAL** — `android.hardware.contexthub@1.2` is in no compatibility
  matrix the tree ships (AIDL `IContextHub` only, FCM 7 up), so `check_vintf_compatible` rejects the
  device manifest. No AIDL CHRE HAL for this platform exists in the tree (`system/chre` has no
  `hal_generic/aidl`) and the AIDL `example` is a fake hub, so the HAL goes; the `chre` daemon and the
  sensors HAL are unaffected. Its sepolicy goes with it.
- **0022 health HAL to AIDL** — same cause for `android.hardware.health@2.1`. Rebuilt on
  `hardware/interfaces/health/aidl/default` (`libhealth_aidl_impl`): `Health` subclass with
  `UpdateHealthInfo` + eMMC `getStorageInfo`/`getDiskStats`, one binary
  `android.hardware.health-service.bonito` that is also the charger (`--charger`, `charger_vendor`
  domain; `overrides: charger`, so `init.hardware.rc`'s `vendor.charger` points at it).
  `BatteryRechargingControl`/`BatteryInfoUpdate` take the AIDL `HealthInfo`; libpixelhealth already
  has `HealthInfo` overloads. Untested on hardware: charger mode, battery defender, learned-capacity
  backup.
- **0023 report Treble labeling violations instead of failing the build** — 24.0 runs
  `check-selinux-treble-labeling` (`system/sepolicy/tests/treble_labeling_tests.md`) on every build
  at `BOARD_API_LEVEL` 202604, and bonito's vendor sepolicy fails it on both counts: coredomain
  types defined in vendor policy (`obdm_app`, `google_camera_app`, `con_monitor_app`) and vendor
  `seapp_contexts` labeling system_ext/product apps (`qtelephony`, `hardware_info_app`,
  `omadm_app`, `secure_ui_service_app`, `ril_config_service_app`, `grilservice_app`, `obdm_app`).
  `PRODUCT_ENFORCE_SELINUX_TREBLE_LABELING := false` keeps the report in the log. Proper fix, later:
  move those app domains and lines to system_ext sepolicy. Not a preflight candidate — the test
  needs the built APKs and both precompiled policies.

### `kernel/google/msm-4.9`

Backports only; there is no newer kernel for this SoC. Each patch names the upstream commit it
comes from. What Android 17 userspace needs from the kernel, in the order it hits it:

- **0001 mm: backport MADV_WIPEONFORK and MADV_KEEPONFORK** — upstream 4.14 `d2cd9ede6e19`.
  Bionic 17 (`upstream-openbsd/android/include/arc4random.h`) calls `madvise(MADV_WIPEONFORK)` on
  its arc4random state and aborts on failure, so on plain 4.9 *every* process dies in libc init —
  `init` first — and the device loops at the Google logo with nothing on USB. Takes the
  `VM_ARCH_2` bit as upstream did; x86 `VM_MPX` moves to a high arch bit (not built here).
- **0002 Revert "selinux: Android kernel compatibility with M userspace"** — Android-only
  `099f006be6e0` ("NOT intended for new Android devices"). Its shim treats any xperms entry whose
  type byte is not 1/2 as an M-format policy and misparses everything after it. Android 15+ policy
  has `allowxperm ... nlmsg` rules, which libsepol writes as type 3 (`AVTAB_XPERMS_NLMSG`), so
  `init selinux_setup` died with `SELinux: avtab: invalid type or class` → `InitFatalReboot` →
  bootloader. Upstream 4.9 stores the byte and loads the policy.
- **0003 selinux: ignore unknown extended permission types instead of BUG()** — companion to
  0002: `services_compute_xperms_decision()` has `BUG()` for type 3, and it is reachable (ioctl
  xperms and nlmsg xperms share the `(domain, self, netlink_route_socket)` key). No netlink xperm
  hook exists in 4.9, so the rules are inert; skip them, warn once.

Find the next floor item without a full boot, on the phone (slot b, test data disposable):

1. Build a hybrid boot image: the 24.0 `boot.img` header + kernel + dtb with the **22.2 recovery
   ramdisk** (unpack both with `unpack_bootimg --format=mkbootimg`, repack with the 24.0 args and
   the 22.2 `--ramdisk`). It boots fastbootd/recovery on the candidate kernel; the 24.0 ramdisk is
   what fails.
2. `fastboot flash boot_b hyb.img`, `fastboot reboot fastboot`, then from fastbootd
   `fastboot reboot recovery` — the bootloader's own `reboot recovery` is unsupported and
   `fastboot boot <img>` does a *normal* boot, so neither is usable. Recovery-mode boot failures
   bounce to the bootloader without touching the slot retry count; normal-boot failures burn one
   each (6 → unbootable). Do not normal-boot slot b until the harness passes.
   `fastboot --set-active=<slot>` resets that slot's retry count and clears `unbootable`; a `misc`
   BCB left by a failed recovery request redirects the next `reboot fastboot` into recovery.
3. In recovery: `adb root`; push the **whole** 24.0 ramdisk (cpio-extract, `chown -R 0:0`) to
   `/tmp/rd24` on the phone (its own tmpfs, lost on reboot). Then run the real init as PID 1 of a
   throwaway PID namespace — `reboot(2)` from a non-root pidns only kills that namespace, so
   `InitFatalReboot` is harmless and the message stays in the live kernel log:

       adb shell 'dmesg -w' > init24-kmsg.log &
       adb shell 'timeout -s KILL 20 toybox unshare -f -p -m chroot /tmp/rd24 /init'

   It runs first stage → `selinux_setup` → second stage on the 24.0 ramdisk against the live
   kernel (recovery mode: no first-stage mount, so `super` is untouched). Read the `init:` lines;
   the first FATAL is the next kernel gap. This is how 0002/0003 were found. For the earlier,
   pre-init floor (bionic itself) `chroot /tmp/rd24 /system/bin/toybox uname -a` is enough — that
   is how 0001 was found (`arc4random data MADV_WIPEONFORK failed: Invalid argument`). Stop the
   harness before second stage starts services if adb matters: 24.0 `adbd` would reconfigure the
   USB gadget under the running recovery.
4. Only then flash the real `boot.img` (+ `dtbo.img`, `vbmeta.img` from the same build) and
   normal-boot; `adb logcat -s NetBpfLoad:* LibBpfLoader:*` for the eBPF floor.

No log survives a reboot on this device: every `reboot` is a PMIC hard reset (`PMIC@SID0
Power-on reason: Triggered from Hard Reset and 'cold' boot`), DDR is power-cycled and
pstore/ramoops comes up empty. `msm_poweroff.warm_reset=1` does not change it (TZ or the
bootloader forces the hard reset with `androidboot.ramdump=disabled`). Use the harness above.

### `vendor/google/bonito`

The blob repo (TheMuppets, lineage-22.2 branch — there is no 24.0 one). The patch touches only
`bonito-vendor.mk` and `Android.bp`; no blob bytes are in this repo.

- **0001 drop the secure UI platform pieces** — `libsecureuisvc_jni.so` (system_ext) links libgui
  by the pre-17 `SurfaceComposerClient::createSurface` signature and fails `check_elf_file`
  (unresolved symbol; `allow_undefined_symbols` would only move the crash to runtime). It is the
  JNI half of `com.qualcomm.qti.services.secureui`, the stock trusted-UI PIN pad; Lineage never
  starts it. Drops the app, the JNI lib, `libsecureui_svcsock_system` and the system_ext copy of
  `vendor.qti.hardware.tui_comm@1.0`. The vendor-side libs, HIDL service and its manifest entry
  stay. Prebuilt `check_elf_file` runs late (droidcore), so preflight it: for every
  `proprietary/(system|system_ext|product)/lib*/*.so` in the blob `Android.bp`, take
  `llvm-nm -D --undefined-only` non-weak symbols and look them up in the defined symbols of the
  staged `out/target/product/bonito/{system,system_ext,product}` libs plus
  `apex/com.android.runtime/<arch>/bionic` and `apex/*/<arch>` (strip `@VERSION`). 14 platform
  blobs here; this was the only one with a miss.

### `packages/modules/Connectivity`

- **0001 netbpfload: do not hang or reboot when bpf programs fail to load** — diagnosis only, while
  the kernel fails NetBpfLoad's floor (`NetBpfLoad.cpp:1571`, 25Q2+ wants >= 5.4). Stock rc reboots
  (`reboot_on_failure reboot,bpfloader-failed`, `netbpfload.35rc`) before init reaches `boot`, so
  adbd never starts; without it init would hang at `wait_for_prop bpf.progs_loaded 1` instead. The
  patch comments out the reboot and starts `netd1shot`/`netd` from `on property:bpf.progs_loaded=1`,
  so on the failing kernel there is no netd (and no network, no system_server) but adbd and logcat
  run. `adb logcat -s NetBpfLoad:* LibBpfLoader:*` shows the floor message. Drop it once the kernel
  passes (eBPF backport + `ro.bpf.kver_override`).

Preview checkpolicy errors without a build: the failing run leaves
`out/soong/.intermediates/system/sepolicy/vendor_sepolicy.conf/.../vendor_sepolicy.conf`; loop
`out/host/linux-x86/bin/checkpolicy -C -M -c 30 -o /dev/null` on a copy, blanking each reported
statement, to list every unknown type at once instead of one per 40-minute run.

Preview these checks without a build: run a built tree's `out/host/linux-x86/bin/checkvintf
--check-compat` with `--dirmap /vendor:` pointing at a directory holding the device manifest
(+ `<sepolicy><version>` appended) and its fragments, and `/system`,`/system_ext`,`/product`,`/apex`
at any built 24.0 tree. Required-HAL enforcement is gone from libvintf (`c2de8e5`), so a device
matrix naming a removed framework HAL (schedulerservice) no longer fails.
The vendor dir must hold the manifest fragments of every HAL *module* the product installs, not
just the device tree's `manifest.xml`: `contexthub@1.2` and `health@2.1` came from
`hardware/interfaces` fragments and a dry run fed only the device manifest passed while the build
failed. Take `vendor/etc/vintf/` from the last built `out/` of the same device (any branch) and
edit that.

## License

Apache-2.0 — see `LICENSE`. The patches under `overlay/patches/` modify Apache-2.0 (AOSP/LineageOS)
code and carry that license.

## Support

This is unpaid work on phones their makers abandoned. If a build saved one from the drawer, [a donation](https://www.paypal.com/donate/?hosted_button_id=7U8PDZLK7742Q) keeps the next one coming.
