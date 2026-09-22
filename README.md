# Pixel 3a XL — LineageOS 24.0 (Android 17)

The **Pixel 3a XL** (`bonito`) on Android 17, running an Android 15 kernel and Android 12 vendor
blobs. It boots, and the hardware works. LineageOS stopped supporting this device after a stale
`lineage-23.0` branch; 23.x will freeze once 24 matures, so the target went straight to 24.

> For a supported build, use the [`lineage-22.2`](../../tree/lineage-22.2) branch. This one is
> ours, not upstream's.

## State

Working: boot (~30 s), display, touch, wifi, Bluetooth incl. audio, audio, fingerprint, camera
(both sensors, stills verified), modem, **VoLTE and VoWiFi**, NFC, vibrator with its factory
calibration, GPU memory reporting, power.stats over AIDL, double-tap-to-wake.

Not working, and why:

| | |
|---|---|
| **CHRE** | The SLPI refuses the DSP image: `undefined symbol #25 __sensors_island_start`. Everything on the Android side is correct — fastrpc opens, the user PD is created, the sensor registry is readable — so this is inside the DSP blobs, paired with firmware from the same factory image. Not fixable from here. |
| **Active Edge** | Needs a CHRE nanoapp, so it follows CHRE. `ElmyraService` hides its own settings entry and tile rather than leaving a toggle that does nothing. The strain gauges are separately readable as `com.google.sensor.elmyra.raw`, so a CHRE-free implementation is possible — but that sensor is `non-wakeUp`, which is the part CHRE existed to avoid. |
| **Double twist** | The sensor exists; nothing consumes it. The gesture lived in Google Camera, which this build does not ship. Correctly absent rather than broken. |
| **Some `.bpf` programs** | `memevents`, `bpfRingbufProg` and `kernelWakelockDuration` want BPF features newer than 4.9 (ringbuf is 5.8+). `gpuMem` and `gpuWork` do load. |

Not gaps: wireless charging (the 3a series has no coil).

## The kernel gate

Android 16+ needs eBPF features this device's 4.9 kernel does not have, and no 4.14+ kernel exists
for its SoC — so the backports had to come to `kernel/google/msm-4.9` rather than the kernel being
replaced. That work is done and is `overlay/patches/kernel/google/msm-4.9/`, one patch per thing
Android 17 userspace needs from the kernel, in the order it hits them. Background and sources in
[ANDROID-16.md](ANDROID-16.md).

Two kernel things are config rather than code, and live in the forge as
`KERNEL_EXTRA_CONFIGS` fragments: `bpf-events` (without `CONFIG_BPF_EVENTS` the tracing BPF program
types are not registered at all, so every `.bpf` object fails to load with a bare `EINVAL` and an
*empty* verifier log) and the `linux` option's container fragments.

## What exists

The full device patch series below, the kernel backports, and the forge options. Every option that
carries patches now has a `lineage-24.0` set. GApps needs `GAPPS_URL` set in `device.conf`, because
NikGapps has no Android 17 build — see the comment there.

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
`bp1a`).

GApps arrives in two halves, which is worth knowing before changing either. GMS Core and the Play
Store come from `vendor/gapps` — **MindTheGapps built from source** by the `gapps` option's own
local manifest (revision `cinnamonbun` = Android 17) — and need nothing set here. `GAPPS_URL` feeds
a separate step, `extract-gapps-apps.sh`, which swaps seven stock apps for Google's builds of them
(Calculator, Calendar, Clock, Contacts, Files, Messages, Phone).

`GAPPS_URL` still has to be set, because the forge otherwise derives Android 17 from the branch and
NikGapps has no Android 17 build — it stops rather than guessing, which is right: GApps are
version-specific and a mismatch is silent, the apps install and are simply built for another
platform. MindTheGapps 17 is used for that half too.

**Velvet** — the Google app and Assistant, the single largest component — is dropped outright by
the option's own patch on every branch, so it does not ship either way.

Room is the other constraint: `WITH_GAPPS=true` drops the `/product` reservation from 512 MiB to
32, which is what makes a 522 MiB GApps package fit inside the 3880 MiB dynamic-partition budget.

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
| `stock` | `stock` | nothing, and **not the shared set either** — plain LineageOS plus only the patches that make this hardware run. Reserved by the forge, so it needs no row in `device.conf`. Use it to tell our bugs from upstream's. |

Every preset also carries the shared set, which is what makes this build look and behave the way
it does regardless of which preset you pick:

`advanced-restart` `dark-default` `google-feed-off` `home-defaults` `linux` `livedisplay-off` `minimal-home` `nav-icons` `setupwizard-nag-skip` `teal-skin` `teal-wallpaper` `themed-icons`

(`nfc-off` was dropped: NFC off out of the box reads as broken hardware rather than as a default.)

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

61 patches across 16 upstream projects, applied at build time from
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
  governor stops dropping frequency between frames, powerhint floors raised to match. Smoothness,
  not benchmark peaks. Don't tune `top-app/schedtune.boost` in the rc: libperfmgr resets that
  node to its powerhint default on start (`ResetOnInit`), so an rc value never survives.
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
  its own VINTF fragment. 0016 also deletes `powerstats/`, the HIDL service source. It is not the
  only user of `hardware/google/pixel/powerstats`: `vendor/google/bonito`'s `libnos_citadeld_proxy`
  needs `pixelpowerstats_provider_aidl_interface-cpp` from it, so that project is carried back by
  `hardware/google/pixel` 0002 regardless. 0028 serves the same data over AIDL.
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
- **0024 install ueventd.rc at `/vendor/etc/ueventd.rc`** — Android 17 ueventd parses
  `/system/etc/ueventd.rc` only (its `import /vendor/etc/ueventd.rc`); the legacy `/vendor/ueventd.rc`
  path pre-T devices relied on is gone (system/core `1b926a344`). Without it every vendor `/dev` node
  stays `0600 root`: surfaceflinger (`/dev/kgsl-3d0`, `/dev/ion`), keymaster/gatekeeper
  (`/dev/qseecom`) and citadeld (`/dev/citadel0`) abort in a loop, no `avc:` anywhere, the phone
  sits on the Google logo with no USB. Found in `pmsg-ramoops-0` (the last boot's logcat), not in
  the console ring. Check any pre-T device tree for this before its first 24.0 boot.
- **0025 include the Lineage libion sepolicy** — system/sepolicy `9b2d37dc2` (Android 17) dropped
  `/dev/ion`'s file_contexts entry and every coredomain `ion_device` rule; the node is labeled
  `device` and surfaceflinger/keymaster/gatekeeper get `avc: denied { read }` on open (gralloc,
  Adreno and QSEECom all allocate through ION on 4.9). `device/lineage/sepolicy/libion/sepolicy.mk`
  carries the removed rules; `BoardConfigLineage.mk` includes it. Rebuild `systemextimage
  vendorimage` (file_contexts is system_ext; the vendor precompiled policy hashes it).
- **0026 keep the gralloc 2/3 probe in libui** — Android 17 libui gates `Gralloc2Mapper`/
  `Gralloc3Mapper` behind `require_gralloc4_or_newer` (ENABLED, READ_ONLY in cp2a) once
  `ro.build.version.sdk` ≥ 36; the display stack is HIDL mapper@2.1/allocator@2.0, so
  `GraphicBufferMapper()` aborts `gralloc-mapper is missing` in composer@2.2, surfaceflinger and the
  camera provider, and surfaceflinger's restart limit reboots into recovery. Lineage kept the probe
  behind soong config `libui.legacy_gralloc` (frameworks/native `c7d417fbe1`); `device-lineage.mk`
  sets it. One module builds both the system and vendor libui. Rebuild `systemimage vendorimage`.
- **0027 drop the CHRE daemon** — `chre_daemon_msm` cannot load its DSP image: the SLPI rejects
  `libchre_slpi_skel.so` with `undefined symbol #25 __sensors_island_start`. It then exits badly
  enough, often enough, to trip init's updatable-crash path and reboot the device. Dropping it
  costs sensor offload to the DSP; the ordinary sensor HAL is unaffected.
- **0028 port the power.stats HAL to AIDL** — restores what 0016 had to drop. `power.stats@1.0` is
  in no matrix ≥ 7, but the AIDL interface is in matrix 7, so the same data passes
  `checkUnusedHals`. `hardware/google/pixel/powerstats` (back for citadeld anyway) carries the AIDL
  `PowerStats`, the generic and wlan providers, the rc and the VINTF fragment, so the device only
  supplies a `main()` naming its rails. The sysfs formats are unchanged from the HIDL era, so the
  parser configs port verbatim: RPMh `master_stats` for APSS/MPSS/ADSP/CDSP, `system_sleep/stats`
  for the AOSD and CXSD domains, `wlan0/power_stats` when debuggable. Citadel is not carried over —
  it was fed over the old vndbinder `power.stats-vendor` interface, which the citadeld blob speaks
  and the AIDL provider does not. No sepolicy change is needed.

- **0029 disable display multirect** — the pipe allocator in `libsdmextension.so` (prebuilt)
  sometimes stages the second rect of a DMA pipe without the first. `sde_crtc_atomic_check()`
  rejects an SSPP left holding only its virtual plane, so `drmModeAtomicCommit` fails `EINVAL`, SDM
  reports "Composition strategies exhausted" and SurfaceFlinger stops presenting — a black panel
  the framework still reports as on, recoverable only by rebooting. Waking, power cycling and
  unblanking the backlight by hand all fail, and a screencap of the wedged device is uniformly
  black, which is what proves it is the compositor and not the panel. The failing planes are the
  virtual twins of the three DMA pipes (`src_blk` 0x25000/0x27000/0x29000). The allocator is a
  blob, but it reads `vendor.display.disable_multirect`. Costs the five secondary rects, leaving
  one composition layer per SSPP.

  **This fixes one path to the wedge, not the wedge.** The `r1 only virt plane` errors are gone
  (zero since), but a second, unrelated fault killed the display on a theme change until
  `hardware/qcom/sdm845/display` 0002. Tell the two apart by the framebuffer: the multirect one
  leaves it blank (the compositor stopped drawing), the other left it ~97% drawn (the compositor
  was fine and the panel never came back). Same end state, opposite causes.
- **0030 sepolicy for two HALs Android 17 started denying** — Android 17 labels `/sys/class/typec`
  `sysfs_typec`, a type new at board API 202604, and no vendor rule granted it, so the health HAL
  was denied on every poll. The gadget HAL needed `get_prop` on `vendor.usb.config`, which only
  ever had `set_prop`. Not fixed deliberately: `sensors.qti` is denied `/dev/diag`, and handing a
  sensor daemon the Qualcomm diag interface is a bigger concession than one denial per boot is
  worth.
- **0031 stop starting netd from post-fs-data** — a boot-time optimisation from when netd had no
  dependency on BPF. On Android 17 netd is `disabled` and started only by
  `on property:bpf.progs_loaded=1`, because `BpfHandler::init()` aborts if its maps are not pinned.
  Starting it from `post-fs-data` ran it about six seconds early, every boot: SIGABRT, a tombstone,
  and init starting it again from the proper trigger. The tell was that the second start ran `netd`
  then `netd1shot` while the trigger does the reverse.
- **0032 ship the CHRE daemon again, disabled** — brought back so the DSP failure can be retested
  without a rebuild, and made inert: `system/chre` 0001 marks the service `disabled` and `oneshot`,
  so neither init nor a manual start can reach the updatable-crash counter that forced 0027. The
  retest has answered — see *State*.


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
- **0004 cgroup: backport the cpuset_v2_mode v1 mount option** — upstream 4.14 `e1cba4b85daa` +
  `b8d1b8ee93df`. Android 16+ `libprocessgroup` mounts the v1 cpuset hierarchy with
  `noprefix,cpuset_v2_mode` and has no fallback; cgroup v1 answers an unknown option with `ENOENT`,
  so `SetupCgroups` fails, `/sys/fs/cgroup` is never mounted, every `createProcessGroup()` fails,
  `ueventd` and `apexd-bootstrap` never start, and `apexd-bootstrap`'s `reboot_on_failure` sends the
  phone to the bootloader ~10 s into second stage (the 10 s is init retrying to stat `misc`, which
  ueventd never created). USB never enumerates, no panic, nothing in klog. Found with the ramoops
  trick below, not the harness: the harness has no `/system`, so it never reaches `SetupCgroups`.

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

**Getting the console log of a failed normal boot.** Every reboot here is a PMIC hard reset and
the bootloader rewrites the primary ramoops region (`ramoops_mem`, 0xa1810000) on every boot — its
own UEFI log lands there — so pstore is always empty after a normal-boot failure. The `klog`
partition and the alt region get an AES-GCM copy of the console **only on a kernel panic**
(`androidboot.init_fatal_panic=true` turns init fatals into panics; `ramoops-pull.sh` decrypts).
A clean `reboot` (init's `reboot_on_failure`, `InitFatalReboot` without the panic flag) leaves
nothing — except that the alt region (`alt_ramoops_mem`, 0xa1a10000) is untouched on a clean
`reboot` (not on `reboot,recovery`: the bootloader clears it on that reason too, verified with
`adb reboot recovery` from a recovery whose ring had content; `msm_poweroff.warm_reset=1` on the
cmdline changes nothing). So init's `reboot_into_recovery` paths (`enablefilecrypto_failed`,
`init_user0_failed`, `fs_mgr_mount_all`) leave no log at all; read the source instead. So swap the two phandles of the `ramoops` node in the dtbo entry the bootloader picks
(`androidboot.dtbo_idx=8`, id 0x31e; `mkdtboimg dump`, `dtc -I dtb -O dts`, swap
`memory-region`/`alt-memory-region`, byte-patch the two u32s back into `dtbo.img`), `fastboot flash
dtbo_b`, normal-boot, then boot recovery and read `/sys/fs/pstore/console-ramoops-0`: the whole
failed boot including init's last lines. Debug image only — with the regions swapped a real panic
would overwrite the live ring. Restore the built `dtbo.img` afterwards.

**Poking the slot-b system from recovery** (no `dmctl`/`lptools` in recovery): pull the first
4 MiB of `system_b` (`super`), `lpdump` it on the host for the extent offsets, then
`losetup -f -o <off> -S <size> /dev/block/by-name/system_b` and `mount -t ext4` — rw works, the
images have no `shared_blocks`. That is how `apexd --bootstrap` was run chrooted into slot b
(`setprop apexd.config.use_fiemap false` first: recovery's `ro.init.mnt_ns.count=1` would flip it
into the mount-before-data path) and passed on this kernel, ruling apexd itself out.

The harness in step 3 stops being useful once init gets past `selinux_setup`: recovery-mode init
has no `/system`, so `SetupCgroups`, `apexd-bootstrap` and everything in `early-init` never run
there. From that point on, use the ramoops trick.

- **0005 gpu: backport the gpu_mem_total tracepoint and emit it from kgsl** — Android reports GPU
  memory by attaching `gpuMem.bpf` to `gpu_mem/gpu_mem_total` and pinning
  `map_gpuMem_gpu_mem_total_map`, which libmeminfo reads for the per-process figures and, at pid 0,
  the global total. The tracepoint arrived in 5.4. The scaffolding is the upstream 5.4 code;
  `drivers/gpu/Kconfig` does not exist on 4.9, so the new Kconfig is sourced from
  `drivers/video/Kconfig` beside the other `drivers/gpu` subdirectories. kgsl already keeps the
  numbers — only the reporting was missing — so per process `stats[]` is summed on each change, and
  the global figure is one `atomic64` moved by the same deltas, making it the sum of the
  per-process totals by construction rather than a second number that could drift. The summing loop
  sits behind `trace_gpu_mem_total_enabled()`.

  Necessary but **not sufficient**: without `CONFIG_BPF_EVENTS` the tracing program types are not
  registered at all, so `find_prog_type()` rejects every `.bpf` object before the verifier runs. See
  *The kernel gate*.


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

### `frameworks/hardware/interfaces`

- **0001 restore `android.frameworks.stats@1.0`** — partial revert of the HIDL stats removal:
  the interface library only, no client or VTS. The fpc fingerprint blob
  (`android.hardware.biometrics.fingerprint@2.1-service.fpc`, `vendor/google/bonito/Android.bp`)
  links it at load time; nothing serves IStats, `getService()` returns null and the blob carries
  on. Needed for as long as that blob is.

### `hardware/qcom/sdm845/display`

- **0001 drop the HDR HBM hook in hwc2** — `hardware/google/interfaces/light/1.0` is gone on 24.0,
  so hwcomposer.qcom no longer linked. The hook toggled panel high-brightness mode through the
  Google light HAL's `setHbm()` when an HDR layer covered more than half the screen; the AIDL
  lights HAL has no such call. Backlight brightness is unaffected (sysfs write).

- **0002 defer fb_id removal until a commit later** — `~FrameBufferObject` called `drmModeRmFB`
  as soon as its refcount hit zero, without regard for whether the hardware was still scanning that
  buffer out. Removing a live framebuffer makes the kernel force-disable the plane and then the
  CRTC (`drm_framebuffer_remove` -> `drm_mode_set_config_internal`, `NOMODE`), so the panel switched
  off while SurfaceFlinger still believed the display was on and never powered it back up — only a
  reboot recovered it. Reproducible on any wallpaper or colour-scheme change, which destroys and
  recreates nearly every surface at once.

  The hazard came from two QCOM commits, `20ec28d0 "sdm: Clear fb_id map if it exceeds the size
  limit"` and `0f70014c "sdm: Reduce the fb_id cache limit for UI layers"`; there is no upstream
  fix. Every path that drops the last reference can trigger it — cache eviction, layer teardown,
  display reconfigure — so the fix is the lifetime rule, not the callers: destruction queues the
  id and the queue drains one full commit later. Verified by counters rather than by the symptom;
  forced plane disables, CRTC `NOMODE` and the composer's own `drm_atomic.c:868` "FB set but no
  CRTC" warning all sit at zero across a dozen theme changes, where the warning previously fired
  continuously.

### `external/tinyxml2`

- **0001 revert "Upgrade tinyxml2 to 11.0.0"** — 11.0.0 changed `DynArray`/`MemPoolT` to hold
  their sizes as `size_t` rather than `int`, which grows every embedded `DynArray` by 8 bytes and
  with it `sizeof(XMLDocument)`. `camera.sdm710.so` stack-allocates an `XMLDocument`:
  `ImageSensorUtils::ReadSensorCalibration()` parses `camera_imu_average_calibration.xml` into one
  on its stack, sized for the tinyxml2 the blob was built against. The 11.0.0 constructor writes
  past that reservation and over the register save area — caught with a watchpoint on the saved
  slot, which trapped the write inside `/vendor/lib64/libtinyxml2.so`. The register is `x20`,
  where `ImageSensorModuleData::GetStaticCaps()` keeps its `TuningDataManager`; the epilogue
  restores it as NULL and `GetChromatix()` dereferences it, so the camera provider SIGSEGVs before
  registering and the device enumerates zero cameras. Nothing in the logs shows it — every CamX
  message matches a working build, because no CamX state is wrong. Not bonito-specific: any
  prebuilt CamX HAL that stack-allocates an `XMLDocument` is affected. The proper fix is a vendor
  variant pinned to the 10.0.0 ABI so the platform can keep 11.

### `system/core`, `system/vold`

One change, three reverts: the session-keyring path for fscrypt v1 keys on a kernel without
`FS_IOC_ADD_ENCRYPTION_KEY` (5.4+; msm-4.9 has only the `fscrypt:`/`ext4:` keyring lookup in
`fs/crypto/keyinfo.c`). Android 17 vold issues that ioctl unconditionally, it fails with ENOTTY,
`vdc cryptfs enablefilecrypto` fails, and init reboots into recovery with
`Reason: enablefilecrypto_failed` (no logcat line survives: the reboot is before adbd starts and
the bootloader clears the alt ramoops region on a `reboot,recovery`).

- **system/core 0001 Revert "Remove libkeyutils"** — the wrapper library both reverts below link.
- **system/core 0002 Revert "init: remove session keyring workaround for old kernels"** — init
  creates the session keyring and the `fscrypt` keyring in it before `installkey`.
- **system/vold 0001 Revert "vold: remove session keyring workaround for old kernels"** — restores
  `isFsKeyringSupported()` (probes the ioctl once) and the `add_key("logon", ...)` fallback for
  v1 keys, plus the drop_caches eviction that goes with it. Rebased onto the wrapped-key changes
  that landed after it.

### `packages/modules/Connectivity`

- **0001 netbpfload: do not hang or reboot when bpf programs fail to load** — diagnosis only, while
  the kernel fails NetBpfLoad's floor (`NetBpfLoad.cpp:1571`, 25Q2+ wants >= 5.4). Stock rc reboots
  (`reboot_on_failure reboot,bpfloader-failed`, `netbpfload.35rc`) before init reaches `boot`, so
  adbd never starts; without it init would hang at `wait_for_prop bpf.progs_loaded 1` instead. The
  patch comments out the reboot and starts `netd1shot`/`netd` from `on property:bpf.progs_loaded=1`,
  so on the failing kernel there is no netd (and no network, no system_server) but adbd and logcat
  run. `adb logcat -s NetBpfLoad:* LibBpfLoader:*` shows the floor message. Drop it once 0002-0004
  are verified on hardware.
- **0002 netbpfload: tolerate a 4.9 kernel** — the U/V/25Q2/25Q4 kernel floors in `NetBpfLoad.cpp`
  are hard returns on 24.0 (22.2 warned); back to warnings. Gate the `bpf_jit_kallsyms` write on
  4.11 (22.2's gate) and the root-only `bpfGetNext{Prog,Map}Id` sanity check on 4.13 (the commands
  do not exist before that; the 4.9 branch of that `if` is documented as "nothing"). Have
  `isMapTypeSupported` skip `LRU_HASH` below 4.10: `netd.o` creates two (`local_net_note_op_cache_map`,
  `loopback_access_cache_map`) unconditionally, and their only users are 5.10+ programs. Do not use
  `ro.bpf.kver_override`: it selects program variants the kernel cannot load.
- **0003 netd.c: let the 4.9 stats programs load on any API level** — the only ingress/egress
  `stats` variant for kernel 4.9 is `stats$4_9_t`, loader range `[T, V)`; on 25Q2+ nothing is pinned
  at `netd_shared/prog_netd_{ingress,egress}_stats` and netd's `BpfHandler::init` aborts on ENOENT.
  Every other variant starts at 4.19, so `[T, MAXAPI)` overlaps nothing.
- **0004 BpfHandler: warn instead of failing on a kernel below 5.4** — the V/25Q2 floors in
  `BpfHandler.cpp` become an `abort()` in `NetdUpdatable`; the rest of `init` is kernel-gated.
  Verifier acceptance of the `$4_9` programs on this kernel is not proven statically
  (`.scratch` preflight loaded the skfilter/schedact/schedcls ones); rebuilding Connectivity needs
  `system.img` (the tethering apex lives in `/system/apex`).

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

### `packages/apps/ElmyraService`

- **0001 stop instead of crashing when there is no context hub** — `onCreate()` indexed
  `ContextHubManager.getContextHubs()[0]` unchecked. Device patches 0021 and 0027 drop the context
  hub HAL and the CHRE daemon, so the list is empty and that throws
  `IndexOutOfBoundsException`; the app is `android:persistent`, so ActivityManager logs
  `crashed too many times, killing!` and immediately re-adds it, forever. Over 2000 process starts
  in one session, forking zygote and burning CPU throughout. The patch checks the list and stops
  the service. `onDestroy()` is guarded too: it unregistered a preference listener the early return
  never registers, and a receiver only registered when `screenRegistered` is set.
  Active Edge stays dead either way — the gesture comes from a CHRE nanoapp. The strain gauges are
  separately visible as `com.google.sensor.elmyra.raw` (MAX11261, continuous 1-100 Hz) via the
  normal sensors HAL, so a CHRE-free implementation is possible, but it would be `non-wakeUp`:
  detection only while the AP is awake, which is the part CHRE existed to avoid.


### `hardware/google/pixel`

- **0001 revert "pixel: Restore drv2624 vibrator HAL APEX"** — brings the drv2624 vibrator HAL
  back. Restoring `vibrator/drv2624` alone does not build: `VibratorHalDrv2624BinaryDefaults` needs
  `PixelVibratorBinaryDefaults` from `vibrator/Android.bp`, so the revert takes the whole set.
- **0002 revert "pixel: Drop powerstats HAL"** — not optional, and an earlier audit was wrong to
  remove it: `vendor/google/bonito` builds `libnos_citadeld_proxy` against
  `pixelpowerstats_provider_aidl_interface-cpp` from `powerstats/`, so dropping the project breaks
  the Titan M proxy at link time. Having it back is also what makes device 0028 cheap.

### `frameworks/base`

- **0001 cap the SystemServiceRegistry wtf loop** — a missing service makes
  `SystemServiceRegistry` log a `wtf` per lookup, which on this device is a flood. Gated on
  `sReportedMissingServices`, so each missing service is reported once.

### `hardware/interfaces`

- **0001 libhealthloop: gate the uevent BPF filter on a 5.3 kernel** — the health HAL's
  `skfilter/power_supply` program needs a verifier this kernel does not have. With
  `DEFINE_BPF_PROG_KVER(..., KVER(5, 3, 0))` the loader skips it by design instead of failing; the
  log line `skipping program ... min_kver:5030000 (kver:4090151)` is this working.

### `system/bpf`

- **0001/0002 loader fixes for 4.9** — the ringbuf test object is not critical, and `prog_name` is
  only written when the kernel is at least 4.15. `bpf_attr` ends at `kern_version` before that, and
  bpf(2) rejects the whole call if any byte past the last field it knows is set — before the
  verifier runs, so the failure arrives as a bare `EINVAL` with nothing to read.

### `system/memory/libmeminfo`

- **0001 check the GPU map exists before opening it** — `BpfMapRO`'s constructor calls
  `abortOnMismatch()`, which aborts the process when the map is not pinned, so the `isValid()` check
  after it can never run. system_server died here on every boot while `gpuMem.bpf` was failing to
  load. `access()` first and take the graceful path the function already has. Still correct now that
  the map does load: it is the difference between a missing feature and a dead system_server.

### `system/chre`

- **0001 do not start the msm daemon automatically** — on a device where the daemon cannot reach
  the SLPI it exits non-zero about 33 times a boot; init counts the 5th bad exit before
  `sys.boot_completed`, sets `sys.init.updatable_crashing`, and apexd answers that by reverting and
  rebooting. `disabled` stops init starting it, `oneshot` stops init restarting it when it is
  started by hand, so the daemon can be tested on a booted device instead of costing a reboot to
  find out.


## License

Apache-2.0 — see `LICENSE`. The patches under `overlay/patches/` modify Apache-2.0 (AOSP/LineageOS)
code and carry that license.

## Support

This is unpaid work on phones their makers abandoned. If a build saved one from the drawer, [a donation](https://www.paypal.com/donate/?hosted_button_id=7U8PDZLK7742Q) keeps the next one coming.
