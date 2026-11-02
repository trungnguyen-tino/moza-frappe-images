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
