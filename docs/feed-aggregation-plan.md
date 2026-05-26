# Unified Feed Aggregation Plan

## Goal

Keep each project feed isolated, then publish one shared signed feed for users.

Current problem:

- `openwrt-feed/25.12/x86_64/packages.adb` can only represent one generated index.
- If `clashoo` and `daed` both upload to the same flat path, the later workflow overwrites the earlier index.
- Users then see missing packages or `UNTRUSTED signature`.

Target result:

- Project feeds do not overwrite each other.
- The legacy user feed path still works:
  - `openwrt-feed/25.12/x86_64/packages.adb`
  - `openwrt-feed/24.10/x86_64/Packages.gz`
- APK and OPKG indexes are regenerated from the merged packages and signed once with the dllkids feed keys.

## Directory Layout

Project-specific paths:

```text
openwrt-feed/clashoo/<sdk>/<arch>/
openwrt-feed/daed/<sdk>/<arch>/
openwrt-feed/jell/<sdk>/<arch>/
```

Unified user-facing paths:

```text
openwrt-feed/<sdk>/<arch>/
```

Example:

```text
openwrt-feed/clashoo/25.12/x86_64/clashoo-2026.05.23.5639e93-r3-x86_64.apk
openwrt-feed/daed/25.12/x86_64/luci-app-daede-1.1-r2-x86_64.apk
openwrt-feed/jell/25.12/x86_64/luci-app-example-1.0-r1.apk
openwrt-feed/25.12/x86_64/packages.adb
```

## Workflow Design

Each project workflow should upload its own project feed:

```text
clashoo -> openwrt-feed/clashoo/
daed    -> openwrt-feed/daed/
jell    -> openwrt-feed/jell/
```

Only `openwrt-clashoo/.github/workflows/aggregate-feed.yml` writes the unified root feed:

- Download both project feeds from B2.
- Copy only package files into `merged/<sdk>/<arch>/`.
- Regenerate package indexes from all merged packages.
- Sign the regenerated indexes.
- Upload only the unified root path `openwrt-feed/<sdk>/<arch>/`.

This avoids two project workflows writing the root `packages.adb` at the same time.

Triggers:

- `workflow_dispatch`: manual run.
- `schedule`: hourly fallback.
- `openwrt-clashoo` release workflow: triggers after uploading `clashoo/`.
- `luci-app-daed` release workflow: optional trigger with `AGGREGATE_FEED_TOKEN`.
- `compile-jell` workflow: optional trigger with `AGGREGATE_FEED_TOKEN`.

## Signing Commands

APK v3:

```sh
apk mkndx --allow-untrusted \
  --sign-key private-key.pem \
  --output packages.adb \
  ./*.apk
```

OPKG:

```sh
ipkg-make-index.sh . > Packages
gzip -9nc Packages > Packages.gz
usign -S -m Packages -s key-build
```

Signing keys:

- `APK_PRIVATE_KEY` signs `packages.adb`.
- `OPKG_KEY_BUILD` signs `Packages`.
- The public key used by `openwrt-feed-setup.sh` must match these private keys.

## Clashoo Repo Changes In Progress

Files touched:

- `.github/workflows/release.yml`
- `scripts/install.sh`
- `clashoo/files/usr/share/clashoo/update/component_update.sh`
- `luci-app-clashoo/root/usr/share/rpcd/ucode/luci.clashoo`

Current direction:

- `scripts/install.sh` and component update should prefer the project manifest:

```text
openwrt-feed/clashoo/<sdk>/<arch>/manifest-clashoo.txt
```

- Persistent system feed should continue to use the unified root path:

```text
openwrt-feed/<sdk>/<arch>/packages.adb
```

## Daed Repo Changes In Progress

Files touched:

- `.github/workflows/release.yml`

Current direction:

- Upload daed packages to:

```text
openwrt-feed/daed/<sdk>/<arch>/
```

- Optionally trigger `openwrt-clashoo`'s `aggregate-feed.yml` after upload.

## compile-jell Repo Changes

Files touched:

- `.github/workflows/Auto-Package.yml`

Current direction:

- Upload jell packages to:

```text
openwrt-feed/jell/<sdk>/<arch>/
```

- Optional trigger for `openwrt-clashoo`'s `aggregate-feed.yml` after upload.

## Important Notes

- Do not only rely on `rclone sync --include` in the flat root path. That avoids deleting unrelated files but does not solve the single-index problem.
- `packages.adb` must be regenerated from all package files together.
- `Packages.gz` must also be regenerated for 24.10/IPK.
- Use `rclone purge b2:<bucket>/openwrt-feed/<sdk>/<arch>` only inside `aggregate-feed.yml`.
- Do not purge `openwrt-feed/clashoo`, `openwrt-feed/daed`, or `openwrt-feed/jell`.
- Keep code comments short.
- Do not push to GitHub before user confirmation.

## Test Plan

Local/static:

```sh
git diff --check
```

GitHub Actions dry review:

- Check workflow YAML syntax.
- Confirm `aggregate` step does not require unavailable local tools.

B2 verification after workflow:

```sh
curl -fsSL https://down.dllkids.xyz/openwrt-feed/25.12/x86_64/packages.adb -o /tmp/packages.adb
apk adbdump /tmp/packages.adb | grep -E 'name: (clashoo|luci-app-clashoo|luci-app-daede|luci-app-|.*jell.*)'
```

252 verification:

```sh
wget -qO- https://down.dllkids.xyz/openwrt-feed/openwrt-feed-setup.sh | sh
apk update
apk add clashoo luci-app-clashoo luci-i18n-clashoo-zh-cn
```

Expected:

- No `UNTRUSTED signature`.
- `apk add` can find both clashoo and daed packages from the unified feed.
