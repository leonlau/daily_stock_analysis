# daily-stock-analysis (mirror)

This repository is an automated **mirror** that publishes a Docker image of
[ZhuLinsen/daily_stock_analysis](https://github.com/ZhuLinsen/daily_stock_analysis).

It contains **no application code**. A GitHub Actions workflow runs daily and,
whenever a new upstream **GitHub Release** is detected, builds and publishes a
fresh image to GitHub Container Registry.

## Produced image

```
ghcr.io/leonlau/daily-stock-analysis:latest
ghcr.io/leonlau/daily-stock-analysis:release-<upstream-tag>
```

- `:release-<tag>` is immutable and bound to a specific upstream Release tag
  (e.g. `release-v1.2.3`). Use this for production deployments that need a
  pinned version.
- `:latest` always points to the most recently built image.

## How it works

See [`docs/superpowers/specs/2026-06-15-gh-action-daily-image-build-design.md`](docs/superpowers/specs/2026-06-15-gh-action-daily-image-build-design.md).

Triggered daily at **02:00 Beijing time** (18:00 UTC) via `schedule`, plus
manual `workflow_dispatch` for ad-hoc rebuilds.

If the upstream has not published a new Release since the last build, the
workflow exits early without producing a new image.

## Usage

Pull and run:

```bash
docker pull ghcr.io/leonlau/daily-stock-analysis:latest

docker run --rm \
  -v $(pwd)/data:/app/data \
  -v $(pwd)/logs:/app/logs \
  -v $(pwd)/reports:/app/reports \
  --env-file .env \
  ghcr.io/leonlau/daily-stock-analysis:latest \
  python main.py
```

For long-running operation, follow the upstream
[`docker/docker-compose.yml`](https://github.com/ZhuLinsen/daily_stock_analysis/blob/main/docker/docker-compose.yml)
with the image override:

```yaml
services:
  analyzer:
    image: ghcr.io/leonlau/daily-stock-analysis:latest
    # ... rest same as upstream
```

## One-time setup for repository owner

After pushing this repo to GitHub as `leonlau/daily_stock_analysis`:

1. **Settings → Actions → General → Workflow permissions**
   - Select **"Read and write permissions"**
   - Save
2. (Optional) Trigger the workflow manually:
   - Actions tab → "Build & Publish Docker Image" → "Run workflow"
3. Verify on first run:
   - A new package `daily-stock-analysis` appears under the
     [Packages page](https://github.com/leonlau?tab=packages)
   - Tags `release-<latest upstream release>` and `latest` exist

## Architecture

- Single-arch (`linux/amd64`) only.
- No custom secrets required — uses the built-in `GITHUB_TOKEN`.
- Buildx layer cache via `type=gha` to keep rebuilds under 2 minutes when
  upstream dependencies are unchanged.

## Limitations

- Image freshness is gated on upstream cutting a GitHub Release. If upstream
  merges to `main` but does not release, this image will not update.
- ARM hosts (Apple Silicon, Synology NAS, etc.) must rebuild locally.
