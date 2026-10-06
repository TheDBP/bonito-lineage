# bonito-lineage

LineageOS for the **Google Pixel 3a XL** (`bonito`, sdm670 / Snapdragon 670, 2019). `main` carries
no build config; each Android version is its own branch, named after the upstream LineageOS branch.

| branch | Android | status |
|---|---|---|
| [`lineage-24.0`](../../tree/lineage-24.0) | 17 | boots in ~30 s and the hardware works — camera (both sensors), VoLTE and VoWiFi, fingerprint, NFC, GPU memory, display |

Upstream LineageOS stopped after a stale `lineage-23.0` branch, so this is not customisation on top
of a maintained build: it is Android 17 running on an Android 15 kernel and Android 12 vendor blobs.
Built with [rom-forge](https://github.com/TheDBP/rom-forge), vendored as `forge/` on the branch.

This build is ours, not upstream's. If you want a supported one, take LineageOS's own — this repo is
not where that lives, and does not try to be.

`lineage-22.2` has been retired. It was a thin layer over supported upstream kept as a fallback,
which is a job upstream already does better without us; keeping it here only implied we maintained
something we did not. Its history is preserved as the tag `archive/lineage-22.2`.

Installing, building, what is changed and what is not: the README on the branch.

## License

Apache-2.0 — see `LICENSE`.
