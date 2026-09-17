# bonito (Pixel 3a XL) and Android 16 — where things actually stand

Researched 2026-09-09. See `forge/docs/lineage-branches.md` for the branch picture for all three devices.

**2026-09-17: this branch now targets `lineage-24.0` (Android 17).** Everything below about 16 holds
for 17 — the stale 23.0 tree, the HAL picture and the kernel gate at the end are the same — so the
skeleton was retargeted rather than built on 23.2, which will freeze once 24 matures.

## Do not base an A16 attempt on upstream's lineage-23.0 tree

`LineageOS/android_device_google_bonito` has a `lineage-23.0` branch, so it looks like the natural
base. It is a stale fork point, not a newer tree:

```
merge-base(lineage-22.2, lineage-23.0) == the lineage-23.0 tip (bde25fa6, 2025-08-22)
commits in 23.0 not in 22.2:  0
commits in 22.2 not in 23.0:  9   (through 2025-11-13)
```

Those nine commits are real work we would be discarding: *"Migrate audio HAL to blueprint"*,
*"Update namespace imports"*, *"Drop unused in-tree kernel headers"*, *"Resolve qtidataservices
denial"*, AVB key handling.

`sargo` is in the identical position — same branch structure, same stale 23.0 tip.

**The correct base for any A16 attempt is this repo's 22.2 tree**, which is newer than both upstream
branches and already carries our OEM assets, GApps swap, and perf tuning.

## Which Android 16 branch to target

`lineage-23.2` (`android-16.0.0_r4`), not 23.0. 23.0 and 23.1 both froze on 2025-12-27; 23.2 is
still taking security bulletins. Our earlier "23 does not boot on the 3a XL" finding was tested
against 23.0 **and should not be treated as settled** — that branch had already been superseded
twice by the time we tried it, and its device tree was the stale one described above.

That is not a prediction that 23.2 will work. It means the experiment we ran does not answer the
question, because both halves of it were stale.

## What the move would actually involve

Upstream never moved bonito past that Aug 2025 fork, so we would be doing the 22.2 -> 23.2 carry
forward ourselves. Measured on `redfin` (a device that did make the move), the device-tree side is
small: 8 commits / 5 files for 15 -> 16, then 7 mostly-deletion commits for 16 -> 16 QPR2, almost
entirely compat shims and cleanup.

The cost is not there. It is in vendor-blob and HAL compatibility -- which for a Pixel with
published binaries is a far better position than the msm8992/msm8994 work on the Nextbit Robin ([TheDBP/ether-lineage](https://github.com/TheDBP/ether-lineage)).

Dependencies to sync alongside the device tree: `android_device_google_gs-common`,
`android_kernel_google_msm-4.9`, `android_packages_apps_ElmyraService`.

---

## UPDATE (2026-09-09, same day): this is now the better first A16 target

Checked what each device actually depends on. bonito references only `qcom-caf/common`
(has lineage-23.2) and `qcom-caf/bootctrl` in its makefiles; its real HALs come from
`hardware/google/*`:

| dependency | lineage-23.2? |
|---|---|
| `hardware/google/pixel` | yes |
| `hardware/google/camera` | yes |
| `hardware/google/interfaces` | no -- check whether it still exists upstream |
| `hardware/google/av` | no -- same |
| `qcom-caf/common` | yes |
| `qcom-caf/bootctrl` | no branches matched; verify |

Compare the V20, whose msm8996 CAF audio tree stops at 22.2 and display/media at 23.0 -- three HAL
trees to carry forward. bonito has almost no per-SoC CAF exposure, which is the expensive kind.

Also confirmed: sdm710 IS in QCOM_BOARD_PLATFORMS on 23.2, so the silent-gating trap that cost days
on ether (Bluetooth and the power HAL vanishing) does not apply.

So the obstacle here is the stale device tree documented above -- which is *our* kind of work, the
same carry-forward we did for ether -- rather than HAL trees upstream has abandoned.

Open question before committing: what happened to `hardware/google/interfaces` and
`hardware/google/av` on 23.x. Actively-supported Pixels build on 23.2, so they were most likely
merged or retired rather than dropped; confirm before treating this as settled.

---

## The kernel gate (2026-09-17): Android 16+ needs eBPF that 4.9 does not have

Everything above is about device trees and HALs. The wall is in front of both. LineageOS's
[Changelog 30](https://lineageos.org/Changelog-30/): Android 16 "requires Linux 5.4 and above, and
... the necessary features have only been properly backported as far back as 4.14. Unfortunately,
LineageOS 22.2 still supports many devices running 4.4 and 4.9. As of now, no complete backports of
the required features exist for these kernels." Their own SDM845 common kernel (4.9) stops at
Android 15 for the same reason.

bonito is `kernel/google/msm-4.9` (4.9.337). No 4.14 or 4.19 exists for sdm670/sdm710 from
Qualcomm or the community; the only newer kernel for this SoC is mainline, which is a different
project (no vendor HALs, no modem, no camera).

So 23.2 and 24.0 cost the same thing first: the 4.14-era eBPF backport set adapted onto msm-4.9.
That is kernel networking/bpf subsystem work, bounded but large, and nothing in this repo starts
it. What 16's loader does below 4.14 has not been read here; LineageOS's statement is that the
features are required, not advisory. They ask to be told (devrel@lineageos.org) if anyone
completes the backport.
