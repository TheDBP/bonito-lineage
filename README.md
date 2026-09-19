# Pixel 3a XL — LineageOS 22.2 (Android 15)

A custom LineageOS 22.2 ROM for the **Google Pixel 3a XL** (`bonito`, Snapdragon 670).

This one is straightforward — LineageOS still supports the device, so this repo is a thin layer of
customisation on top of upstream rather than a rescue mission.

**Status: working.** Built, flashed, and running well on one Pixel 3a XL (`full` preset).
Hardware support and limitations are official LineageOS 22.2's for this device; what this build
changes is listed below, and nothing else is claimed.

## Build it

```sh
git clone https://github.com/TheDBP/bonito-lineage.git
cd bonito-lineage
PRESET=clean ./forge/bootstrap.sh
```

Needs Docker and enough free disk for a full AOSP checkout plus build output.

**Output:** `build_output/src/out/target/product/bonito/lineage-22.2-*.zip`

## What you can build

One command produces one image. `PRESET` names a saved set of options:

```sh
PRESET=clean ./forge/bootstrap.sh      # or libre, or full
```

To pick options directly instead of using a preset:

```sh
OPTIONS="gapps root" ./forge/bootstrap.sh
```

Options come from the forge (`forge/options/`) and behave the same on every device; what lives in
this repo's `overlay/patches/` is only what is true of this phone.

## Presets

One build command produces one image. A preset is a saved selection of options — it has no
behaviour of its own.

| preset | tag | adds over `clean` |
|---|---|---|
| `clean` | `turbo-clean` | nothing — this is the baseline |
| `libre` | `turbo-libre` | `fdroid`, `fulguris`, `k9`, `termoneplus`, `kdeconnect`, `connectbot`, `linphone` |
| `full` | `turbo` | `fdroid`, `fulguris`, `gapps`, `k9`, `termoneplus`, `kdeconnect`, `root`, `connectbot`, `linphone` |
| `cloud` | `turbo-cloud` | `full` plus `nextcloud-core` (Files, Talk, NextPush, DAVx5) |

Every preset also carries the shared set, which is what makes this build look and behave the way
it does regardless of which preset you pick:

`advanced-restart` `dark-default` `google-feed-off` `home-defaults` `linux` `livedisplay-off` `minimal-home` `nav-icons` `nfc-off` `setupwizard-nag-skip` `teal-skin` `teal-wallpaper` `themed-icons`

`oem` and `nextcloud` are in no preset. `EXTRA_OPTIONS` adds an option to whichever preset you
build, and every option added that way appends its name to the tag:

```sh
EXTRA_OPTIONS=oem PRESET=full ./forge/bootstrap.sh          # tag turbo-oem
EXTRA_OPTIONS=nextcloud PRESET=libre ./forge/bootstrap.sh   # tag turbo-libre-nextcloud
```

The whole Nextcloud bundle fits on `libre` or `clean` only: `full` with GApps is already within
125 MB of the 4 GB super partition, and the build refuses the pair at lunch. `cloud` is the way to
have Nextcloud beside GApps: Firefox (320 MB staged) is the only thing big enough to make room for
Files, Talk, NextPush and DAVx5 (270 MB), so it goes and Fulguris (9 MB) is the browser.

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
| `connectbot` | ConnectBot: an SSH client with saved hosts, keys and port forwarding |
| `linphone` | Linphone: a SIP client, for voice over data where the device has no VoLTE |
| `fulguris` | Fulguris as the browser, replacing Jelly — a WebView browser, 9 MB where Fennec stages 320 MB |
| `gapps` | Google apps: Play Store and GMS from MindTheGapps, plus Google's versions of the stock apps |
| `google-feed-off` | Google feed (-1 screen) off by default |
| `home-defaults` | Home screen defaults: no icon labels, no auto-add of new apps |
| `kdeconnect` | KDE Connect: phone <-> desktop notifications, clipboard, files, remote input |
| `linux` | On-device Linux environment (chroot + Docker): container kernel config and cgroup fixes |
| `livedisplay-off` | LiveDisplay off by default |
| `minimal-home` | Minimal home screen: hotseat only, no second page |
| `k9` | K-9 Mail (the Thunderbird for Android codebase) as the mail client |
| `nav-icons` | Nextbit Robin style nav-bar icons, drawn as scalable tintable vectors (on every preset) |
| `nextcloud` | Nextcloud bundle: Files, Talk, NextPush, Deck, NC Passwords, Notes, DAVx5, Tasks — the current F-Droid build of each, fetched at build time. `EXTRA_OPTIONS=nextcloud` on `libre` or `clean`, see *Presets* |
| `nextcloud-core` | Files, Talk, NextPush and DAVx5 only — the `cloud` preset |
| `nfc-off` | NFC off by default |
| `oem` | Reclaimed stock-ROM boot animation, wallpapers and sounds — the Nextbit Robin's here, see *Presets* |
| `root` | Magisk baked into the boot image, so the zip flashes pre-rooted |
| `setupwizard-nag-skip` | Skip recovery/metrics/backup setup pages |
| `teal-skin` | Teal accent — fixed #009D94 Monet preset seed |
| `teal-wallpaper` | Teal-shag default wallpaper (baked into framework-res) |
| `termoneplus` | TermOne Plus terminal emulator |
| `themed-icons` | Themed (monochrome) app icons on by default |

## Device patches

4 patches across 1 upstream project, applied at build time from
`overlay/patches/`. Nothing here is a fork: each is a single commit against the upstream tree,
replayed on every build, so upstream stays upstream and what we changed stays legible.

One patch per thing it enables.

### `device/google/bonito`

- **0001 size the partitions for what the build bakes in** — Lineage reserves 1000 MiB in product
  and 90 MiB each in system/system_ext for post-install GApps. With GApps, Firefox and the OEM
  assets baked in, that pushes super over `BOARD_SUPER_PARTITION_SIZE`. Under `WITH_GAPPS`: product
  32 MiB, system/system_ext 16 MiB each. `libre` (Firefox, no GApps) is 306 MB over with the full
  reservation, so it keeps 512 MiB — enough for a small GApps flash, not MindTheGapps; with the
  Nextcloud bundle on top (`WITH_NEXTCLOUD`) it keeps 128 MiB. GApps plus the bundle does not fit
  at all, and the patch says so at lunch. `clean` keeps the full reservation.
- **0002 schedutil and powerhint tuning** — longer `down_rate_limit_us` on both clusters so the
  governor stops dropping frequency between frames, a higher top-app schedtune boost, powerhint
  floors raised to match. Smoothness, not benchmark peaks.
- **0003 do not enforce VINTF kernel requirements on OTA** —
  `PRODUCT_OTA_ENFORCE_VINTF_KERNEL_REQUIREMENTS := false`; the shipped kernel does not declare
  everything the OTA-time check expects, and the check refuses the package rather than warning.
- **0004 tag the build** — build tag in the zip filename and `ro.lineage.version`
  (`…-UNOFFICIAL-<tag>-bonito`), overridable via `TURBO_BUILD_ID` (how the forge gives each preset
  its tag). Set before the `common_full_phone` inherit, or `version.mk` never sees it.

## Flash it

Prebuilt zips are on the [Releases](https://github.com/TheDBP/bonito-lineage/releases) page —
always the `libre` preset: LineageOS plus F-Droid, Fulguris, K-9 Mail, TermOne Plus, KDE Connect,
ConnectBot and Linphone,
no Google apps, not rooted. The zip carries its own boot image, recovery included.

Standard Pixel procedure — unlock the bootloader, boot to fastboot, sideload the zip from recovery.

```sh
adb reboot recovery                    # then Apply update -> Apply from ADB on the phone
adb sideload lineage-22.2-*-bonito.zip # a first install: Factory reset -> Format data first
adb reboot bootloader                  # full preset only: root lives outside the zip on A/B
fastboot flash boot boot-magisk.img
fastboot reboot
```

Root is a separate step here: the zip is payload-based (A/B), so the `root` option cannot swap a
patched boot image into it. `boot-magisk.img` sits next to the zip. Dirty flash = the same steps
without the format.

The sideload sits on *Step 1/2* for a long time. The recovery pulls the 1.5 GB zip over USB 2.0 in
64 KB FUSE reads, twice (signature check, then the payload), and writes every partition of the
other slot; *Step 2/2* is dex2oat of the new slot. It is bandwidth, not a hang.

> **Careful:** this device uses *retrofit dynamic partitions*. Use `fastboot flashall` or sideload
> the zip. Do **not** `fastboot flash system` a single partition — on this layout that writes over
> the physical partition and destroys the super metadata, taking the phone down with it.

## What is different from stock LineageOS

- Themed (monochrome) icons on by default, dark theme by default, teal accent
- Minimal home screen; Google feed (−1 screen) off
- NFC off by default; LiveDisplay off; advanced restart in the power menu
- Setup wizard skips the recovery/metrics/backup nags
- `libre` and `full`: Fulguris, F-Droid, K-9 Mail,
  TermOne Plus, KDE Connect; `full` adds GApps (minus Velvet, the Google app — the `gapps` option
  drops it) and Magisk
- Responsiveness tuning: higher CPU floors and a stronger foreground boost

## More

| File | What is in it |
|---|---|
| [ANDROID-16.md](ANDROID-16.md) | Whether this device can go to Android 16, and what it would take |
| `device.conf` | Every knob this build has |

## License

Apache-2.0 — see `LICENSE`. The patches under `overlay/patches/` modify Apache-2.0 (AOSP/LineageOS)
code and carry that license.

## Support

This is unpaid work on phones their makers abandoned. If a build saved one from the drawer, [a donation](https://www.paypal.com/donate/?hosted_button_id=7U8PDZLK7742Q) keeps the next one coming.
