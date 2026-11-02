# moza-frappe-images

Container images of Frappe apps for the Moza app store, built from the unmodified upstream sources with
[frappe_docker](https://github.com/frappe/frappe_docker)'s layered `Containerfile`. One image per app: a Frappe 16 bench
holding that app and the apps it requires.

Why: since frappe_docker read the apps list from a BuildKit secret instead of a build-arg (commit `ae275df161`,
2026-04-05), the official images of these apps published after that date contain Frappe alone (`sites/apps.txt` lists
only `frappe`). The workflow here passes the list as the `apps_json` secret and refuses an image whose bench lacks an app.

| Image | Apps (upstream tag or branch) | License |
|---|---|---|
| `ghcr.io/trungnguyen-tino/frappe-crm` | [frappe/crm](https://github.com/frappe/crm) `v1.91.1` | AGPL-3.0 |
| `ghcr.io/trungnguyen-tino/frappe-insights` | [frappe/insights](https://github.com/frappe/insights) `v3.15.1` | AGPL-3.0 |
| `ghcr.io/trungnguyen-tino/frappe-lms` | [frappe/payments](https://github.com/frappe/payments) `version-16`, [frappe/lms](https://github.com/frappe/lms) `v2.68.0` | MIT, AGPL-3.0 |
| `ghcr.io/trungnguyen-tino/frappe-raven` | [The-Commit-Company/raven](https://github.com/The-Commit-Company/raven) `v3.2.0` | AGPL-3.0 |
| `ghcr.io/trungnguyen-tino/frappe-builder` | [frappe/builder](https://github.com/frappe/builder) `v1.36.5` | MIT |
| `ghcr.io/trungnguyen-tino/frappe-lending` | [frappe/erpnext](https://github.com/frappe/erpnext) `v16.53.0`, [frappe/lending](https://github.com/frappe/lending) `v16.7.1` | GPL-3.0 (both) |

Every image also holds [Frappe Framework](https://github.com/frappe/frappe) (MIT) from the `version-16` branch at build
time. The exact sources of an image are in the image itself (`sites/apps.json`: version and commit of every app) and in
the summary of the workflow run that built it (digest, versions, commits, frappe_docker commit). Nothing is patched.

Tags: `<app tag>` (moves with each rebuild) and `<app tag>-<YYYYMMDD>`. The Moza app store pins images by digest
(`repo@sha256:…`), never by tag.

## Current images (2026-11-02)

Built by runs [41458243322](https://github.com/trungnguyen-tino/moza-frappe-images/actions/runs/41458243322) (crm) and
[41459339591](https://github.com/trungnguyen-tino/moza-frappe-images/actions/runs/41459339591) (the others), frappe_docker
`d6cc2e0d054e69d6c9288134080fa933be564d13`, Frappe 16.54.0. These digests are the ones the Moza catalog pins and tested.

| Image | Apps in `sites/apps.txt` | Digest |
|---|---|---|
| `frappe-crm` | frappe 16.54.0, crm 1.91.1 | `sha256:fdc66453f9f390943fd646a05a4a16efd05f5a7dcb76efca59be4ea23edbf08d` |
| `frappe-insights` | frappe 16.54.0, insights 3.15.1 | `sha256:39568465cf7ca7788306d6426bb4e8a3c88a734710018a36086d865dded31d83` |
| `frappe-lms` | frappe 16.54.0, payments 0.0.1, lms 2.68.0 | `sha256:4297c21ba12679c8a2e6f10ef310445c799375ac1fc229efe3f924c827716cd3` |
| `frappe-raven` | frappe 16.54.0, raven 3.2.0 | `sha256:b5159985980ba1cba1e7e5477f2b1f26b09b83a7e2a585daf31f948e169ec300` |
| `frappe-builder` | frappe 16.54.0, builder 1.36.5 | `sha256:753ff25d91d4821516579fda8df7db8b53e6b8eea39495eddc27d8dfe33c67a8` |
| `frappe-lending` | frappe 16.54.0, erpnext 16.53.0, lending 16.7.1 | `sha256:22322406dec8d1a38064d5f680f3df71664b38f5b2c8326ab79c58f8d60454d4` |

## Build

`apps/<app>.json` is the apps list of each image (frappe_docker `apps.json` format; the app's own release tag last).
The workflow `Build Frappe app images` runs only by hand:

```sh
gh workflow run build.yml -R trungnguyen-tino/moza-frappe-images -f apps=crm          # or crm,raven or all
gh run watch -R trungnguyen-tino/moza-frappe-images
```

Inputs: `frappe_branch` (default `version-16`) and `frappe_docker_ref` (the frappe_docker commit, pinned). Each job
builds `linux/amd64`, pushes to GHCR, then pulls the image back and fails unless every app of its `apps/<app>.json` is
in `sites/apps.txt`.

## Monthly rebuild

1. Bump the tags in `apps/*.json` to the latest upstream releases (check each app's Frappe range in its
   `pyproject.toml`, `[tool.bench.frappe-dependencies]`); bump `frappe_docker_ref` if frappe_docker changed.
2. Run the workflow with `apps=all` (or only the apps that changed). Rebuild immediately on a Frappe security release.
3. From each run summary, take the `repo@sha256:` pin and the versions into the app's pack in the Moza catalog
   (`mozaai-api/resources/catalog/agent-platforms/<slug>`). A new digest makes the catalog app untested: run its
   signed test on the test machine, then approve.
4. Installed apps take the new image with "Cập nhật"; their setup service runs `bench migrate` before the app serves.
