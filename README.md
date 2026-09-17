# bonito-lineage

LineageOS for the **Google Pixel 3a XL** (`bonito`, sdm670 / Snapdragon 670, 2019). `main` carries
no build config; each Android version is its own branch, named after the upstream LineageOS branch.

| branch | Android | status |
|---|---|---|
| [`lineage-22.2`](../../tree/lineage-22.2) | 15 | builds, flashes and runs |
| [`lineage-24.0`](../../tree/lineage-24.0) | 17 | groundwork only — nothing built; gated on an eBPF backport to the 4.9 kernel |

Upstream LineageOS still maintains this device, so the branches are customisation on top: a short
device patch series plus the shared options, built with
[rom-forge](https://github.com/TheDBP/rom-forge), vendored as `forge/` on each branch.

Installing, building, what is changed and what is not: the README on the branch.

## License

Apache-2.0 — see `LICENSE`.
