# Burp Suite Pro Docker Release Pipeline

Automated multi-arch (`linux/amd64` and `linux/arm64`) Docker build and release
pipeline for Burp Suite Professional.

## Published Images

Images are published to GHCR: `ghcr.io/theunknownzoo/burp-pro`

| Variant | Tag | Description |
|---|---|---|
| **GUI** | `:2026.11.2` (or `:latest`) | Burp Suite Pro on JDK 25 with the `loader.jar` agent, X11/GTK runtime libraries, Xvfb, `xdotool`, and `xclip`. Licensed during the build by `keygen.py` + `activate_burp.py`. |

All tags are multi-arch manifests supporting both `linux/amd64` and
`linux/arm64` natively. The per-architecture tags (`:2026.11.2-linux-amd64`,
`:2026.11.2-linux-arm64`) remain in the registry as build artifacts.

## Triggering a Release

Run the workflow manually via GitHub Actions:

1. Go to **Actions** → **Docker**.
2. Click **Run workflow**.
3. Configure the inputs:
   - **`tag`**: Burp Suite version to download and publish (e.g. `2026.11.2`).
   - **`source_ref`**: Branch, tag, or commit SHA in the private source repo
     `nguyenthdat/burp-pro` (default: `main`).
   - **`push_latest`**: Also tag the manifest as `latest` (default: `false`).
   - **`auto_activate`**: Run the license activation during the build (default:
     `true`). Disable it to publish an unlicensed image and activate at runtime
     instead.

The installer is fetched from PortSwigger by the source repo's
`scripts/burp_download.py`, which verifies the published SHA-256 before the
build starts. A bad version/platform pair fails the `prepare`/installer step,
not the image build.

## Quick Usage

### GUI (via X11)

```bash
docker run --rm -it \
  -e DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  -v burp-home:/home/ninja \
  ghcr.io/theunknownzoo/burp-pro:2026.11.2
```

### Version check

```bash
docker run --rm ghcr.io/theunknownzoo/burp-pro:2026.11.2 --version
```

### Activate an unlicensed image

```bash
# Build/publish with auto_activate: false, then:
docker volume create burp-home
docker run --rm --entrypoint burp-activate \
  -v burp-home:/home/ninja \
  ghcr.io/theunknownzoo/burp-pro:2026.11.2
```

The activation writes the keygen license and the activation response into
`/home/ninja/.java/.userPrefs/burp/prefs.xml`, so keep `/home/ninja` on a volume
if you want the license to survive a container restart.

## Workflow Architecture

- **Native runners**: Builds run on `ubuntu-24.04` (amd64) and `ubuntu-24.04-arm`
  (arm64), so neither the ~420 MB installer extraction nor the Xvfb license
  activation runs under QEMU emulation.
- **Checksum-verified installer**: Each runner downloads only its architecture's
  installer with `scripts/burp_download.py`, which verifies PortSwigger's
  published SHA-256 and fails the job on a mismatch.
- **Build-time activation**: With `auto_activate: true` the Dockerfile boots
  Burp through the license wizard under Xvfb, so the pushed image starts
  licensed; the same path is reachable at runtime via `burp-activate`.
- **Single-arch, then merge**: Each arch pushes `<tag>-linux-<arch>`; the
  `merge` job joins them into one multi-arch manifest with
  `docker buildx imagetools create`.
- **Privacy**: The source repository stays private. Docker build logs are piped
  to a file and only surfaced (error lines) if a build fails, and no build cache
  or source artifacts are stored with the published image.

## One-Time Setup

- **Environment**: create `release` under **Settings → Environments** and
  restrict it to the `main` branch. The `build` job runs in it.
- **Secret**: add `SOURCE_TOKEN` to that environment: a fine-grained PAT with
  **Contents: Read-only** on `nguyenthdat/burp-pro`. The public workflow only
  reads the private source; it never needs write access.
- **Package visibility**: the image embeds the keygen and activation scripts from
  the private source, so after the first push open the `burp-pro` package under
  **Package settings → Danger Zone** and set it to **Private**. Private packages
  require `docker login ghcr.io` before pulling.
