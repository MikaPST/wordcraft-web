<p align="center">
  <img src="https://raw.githubusercontent.com/storytold/wordcraft/main/assets/app-icon/hicolor/256x256/apps/ai.storyteller.wordcraft.png" width="128" height="128" alt="WordCraft application icon">
</p>

<h1 align="center">WordCraft Web — Docker Image</h1>
<p align="center">
  A community-maintained Docker image for
  <a href="https://github.com/storytold/wordcraft">WordCraft</a>,
  bringing its browser-based word processor to a self-hosted web interface.
  <br>
  Automated official-release builds and publication to GitHub Container Registry (GHCR).
</p>

<p align="center">
  <a href="https://github.com/MikaPST/wordcraft-web/actions/workflows/publish-image.yml"><img src="https://github.com/MikaPST/wordcraft-web/actions/workflows/publish-image.yml/badge.svg" alt="Build and publish WordCraft Web"></a>
  <a href="https://github.com/MikaPST/wordcraft-web/releases/latest"><img src="https://img.shields.io/github/v/release/MikaPST/wordcraft-web?label=Release&logo=github&color=blue" alt="Latest Release"></a>
  <a href="https://github.com/MikaPST/wordcraft-web/pkgs/container/wordcraft-web"><img src="https://ghcr-badge.egpl.dev/mikapst/wordcraft-web/latest_tag?trim=major&label=GHCR" alt="GHCR image tag"></a>
  <a href="docker-compose.yml"><img src="https://img.shields.io/badge/Docker%20Compose-ready-2496ED?logo=docker&logoColor=white" alt="Docker Compose configuration"></a>
</p>

---

## Why this repository?

The official WordCraft project provides a WebAssembly-based word processor that can be hosted as a static website. This repository packages the browser application into a Docker image for convenient self-hosting.

The goals are to:

- Build the official WordCraft web application from the exact upstream release tag.
- Publish ready-to-run Docker images to GHCR for both AMD64 and ARM64.
- Check for new official releases daily and publish corresponding versioned images.
- Create a matching GitHub Release **only after** the Docker image has been published.
- Offer a simple Docker Compose configuration and support Unraid deployments.
- Maintain Docker, NGINX and workflow configuration independently of upstream.
- Preserve upstream license, attribution and font-license notices in the image.

This is a packaging and distribution project, **not** a fork intended to develop a separate word processor.

## How it works

WordCraft Web runs inside the browser using Rust, WebAssembly and an `eframe`/`wgpu` interface. It prefers WebGPU and automatically falls back to WebGL2. The Docker container serves the compiled static files through NGINX; it is not a remote desktop or a server-side document processor.

- **Browser-based editing:** use the familiar ribbon interface for text, paragraphs, styles, lists, tables and page layout.
- **Document formats:** open and save DOCX, with support for ODT, RTF, HTML, Markdown, LaTeX and TXT, and export to PDF.
- **Review and references:** comments, tracked changes, citations, footnotes and tables of contents are provided by the upstream app, subject to its current alpha limitations.
- **Local file handling:** opening documents uses the browser file picker; saving and exporting normally use browser downloads.
- **No server-side document storage:** no WordCraft account, document database, mounted volume or remote desktop is included.
- **Embedding:** the upstream web application supports iframe hosting, a same-origin parent bridge and a canvas host API; consult the official hosting guide for its security and integration contracts.

WordCraft is still in **alpha**. Complex DOCX fidelity and advanced feature coverage are evolving; keep backups of important documents. Browser preferences are kept in local storage, not in the Docker container.

## Docker image

The image is published to GitHub Container Registry:

```text
ghcr.io/mikapst/wordcraft-web:latest
```

Each official upstream release is also published under its original version tag, for example:

```text
ghcr.io/mikapst/wordcraft-web:v0.15.1
```

### Option 1 — Docker Compose (recommended)

A `docker-compose.yml` file is included at the repository root:

```yaml
services:
  wordcraft-web:
    image: ghcr.io/mikapst/wordcraft-web:latest
    container_name: wordcraft-web
    ports:
      - "8080:8080"
    restart: unless-stopped
```

Start the application:

```bash
docker compose up -d
```

Open `http://YOUR_SERVER_IP:8080` (or `http://localhost:8080` when Docker runs on your own computer).

#### Useful Compose commands

```bash
# Follow application logs
docker compose logs -f

# Update to the newest published image
docker compose pull
docker compose up -d

# Stop and remove the container
docker compose down
```

No `.env` file or persistent volume is required. To change the host port, modify the left side of the mapping, for example `8081:8080`.

### Option 2 — Run directly with Docker

```bash
docker pull ghcr.io/mikapst/wordcraft-web:latest

docker run -d \
  --name wordcraft-web \
  --restart unless-stopped \
  -p 8080:8080 \
  ghcr.io/mikapst/wordcraft-web:latest
```

Open `http://YOUR_SERVER_IP:8080`. Optional CPU and memory limits may be configured with Docker if required.

### Network access and reverse proxies

The container listens on HTTP port `8080`. Use a reverse proxy such as Traefik for HTTPS and avoid exposing unencrypted HTTP directly to the public Internet.

**WebGPU and clipboard access require a secure context**, normally HTTPS or localhost. On non-secure origins WordCraft can fall back to WebGL2. The app does not require cross-origin isolation headers such as COOP/COEP. Its iframe host bridge has additional same-origin and sandbox requirements; refer to upstream documentation before embedding it in another site.

## Repository structure

The repository versions its own Docker packaging and retrieves the official application source only during CI builds.

```text
wordcraft-web/
├── .github/
│   └── workflows/
│       └── publish-image.yml
├── .dockerignore
├── Dockerfile
├── nginx.conf
├── docker-compose.yml
└── README.md
```

| File | Purpose |
|---|---|
| `Dockerfile` | Defines the unprivileged NGINX runtime image, OCI metadata and health check. |
| `nginx.conf` | Serves WebAssembly, compressed assets, hashed-file caching, security headers and `/healthz`. |
| `.dockerignore` | Restricts the image build context to the generated site, legal notices and configuration. |
| `docker-compose.yml` | Defines the ready-to-use Compose deployment. |
| `.github/workflows/publish-image.yml` | Detects upstream releases, builds WebAssembly, publishes multi-platform GHCR images and creates GitHub Releases. |
| `README.md` | Documents deployment, architecture, automation and attribution. |

### Docker build architecture

WordCraft's official web build uses **Trunk**, not `cargo xtask web`. Its `apps/wordcraft-web/Trunk.toml` writes the built static site to `dist/web` at the upstream repository root, with relative URLs and content-hashed filenames.

The build process follows these steps:

1. GitHub Actions resolves the latest published official WordCraft release or a manually selected tag.
2. It checks out that exact source tag in `upstream/` and checks out the revision of `craft-i18n` specified by `upstream/craft-i18n.rev`.
3. It checks out the shared `craft-fonts` assets at the reference used by the upstream release workflow at the time this packaging was prepared.
4. Rust, the `wasm32-unknown-unknown` target and Trunk are installed.
5. `trunk build --release` compiles WordCraft in `upstream/apps/wordcraft-web`, with the shared fonts and language data available through `CRAFT_FONTS_DIR` and `CRAFT_I18N_DIR`.
6. The workflow verifies `upstream/dist/web/index.html`, JavaScript and WebAssembly outputs, and copies them to a temporary `image-context/site/` folder.
7. Version-controlled `Dockerfile`, `nginx.conf` and `.dockerignore` are copied from this repository, alongside upstream licenses and attribution files.
8. Docker Buildx publishes a static NGINX image to GHCR for `linux/amd64` and `linux/arm64`.
9. A matching GitHub Release is created or updated **only after** the image push succeeds.

The WebAssembly application is compiled once and packaged into both container architectures. No server-side database, GPU passthrough, document storage volume or remote desktop is needed.

The runtime exposes `/healthz`, uses the unprivileged `nginx` user and serves `.wasm` with the required MIME type. Hashed JS/WASM assets receive immutable caching; `index.html` is revalidated. NGINX supports precompressed `.gz` assets if the upstream build provides them.

> [!NOTE]
> Upstream Rust, Trunk, fonts, translation data or build-path changes may require adjustments. The archive has been statically checked; a successful GitHub Actions run is required to validate the complete upstream compilation and container publication.

## Automated builds

GitHub Actions checks official WordCraft releases every day.

| Trigger | Behaviour |
|---|---|
| Daily schedule | Checks the latest official release and builds it if it has not already been published here |
| Manual workflow dispatch | Builds the latest release or the specified official upstream tag |
| Force rebuild | Rebuilds an existing version and updates its image and GitHub Release |
| Packaging changes on `main` | Rebuilds the latest official release when version-controlled packaging or documentation changes |

### Published tags

Each successful build publishes:

- `latest` — most recently packaged official WordCraft release.
- `<upstream-version>` — the exact official release tag, for example `v0.15.1`.
- `upstream-<commit>` — the first 12 characters of the upstream source commit.

Images are published for `linux/amd64` and `linux/arm64`.

### GitHub Releases

After the Docker image is successfully pushed to GHCR, the workflow creates or updates a [GitHub Release](https://github.com/MikaPST/wordcraft-web/releases) matching the upstream version.

Each release includes the official upstream release link and notes, source revision, Docker tags, supported architectures, image digest and the shared font/translation revisions used during the build.

The **Release** badge at the top of this README displays the latest GitHub Release. Scheduled runs skip already published versions. To rebuild, open **Actions → Build and publish WordCraft Web → Run workflow** and enable `force_rebuild`.

> [!IMPORTANT]
> A GitHub Release is created **only after** a successful GHCR image push. `latest` is mutable; forced rebuilds can also update versioned tags. Pin an image digest when exact reproducibility is required.

## Unraid Community Applications

WordCraft Web is suitable for Unraid Docker deployments. The Community Applications template is maintained separately from this repository:

- **Unraid Community Applications repository:** [MikaPST/unraid-community-apps](https://github.com/MikaPST/unraid-community-apps)
- **WordCraft Web template:** [wordcraft-web.xml](https://github.com/MikaPST/unraid-community-apps/blob/main/templates/wordcraft-web.xml)
- **Docker image:** `ghcr.io/mikapst/wordcraft-web:latest`
- **Container port:** `8080`

The Docker image does not store documents in Unraid `appdata`; document files are handled in the browser.

## Upstream project

WordCraft is an open-source, clean-room word processor developed by the ArtCraft team and its contributors.

- **Source code:** [storytold/wordcraft](https://github.com/storytold/wordcraft)
- **Official website:** [getartcraft.com/apps/wordcraft](https://getartcraft.com/apps/wordcraft)
- **Official web hosting guide:** [Hosting WordCraft for the web](https://github.com/storytold/wordcraft/blob/main/packaging/web/README.md)
- **Official releases:** [WordCraft releases](https://github.com/storytold/wordcraft/releases)

This repository publishes independent Docker packaging built from the official upstream release, not an official WordCraft distribution.

## License and attribution

WordCraft is dual-licensed under the **MIT License or Apache License 2.0**, at the user's option. Upstream copyright and attribution notices must be preserved.

- [Upstream MIT License](https://github.com/storytold/wordcraft/blob/main/LICENSE-MIT)
- [Upstream Apache License 2.0](https://github.com/storytold/wordcraft/blob/main/LICENSE-APACHE)
- [Upstream NOTICE](https://github.com/storytold/wordcraft/blob/main/NOTICE)
- [Upstream asset attributions](https://github.com/storytold/wordcraft/blob/main/ATTRIBUTION.md)

The workflow copies upstream license and attribution files, together with bundled font licenses, into the published runtime image. The upstream software licenses do not automatically determine the license of independently maintained packaging files; consult this repository's own license if present.

WordCraft, ArtCraft, their logos and trademarks remain the property of their respective owners. This project is community-maintained and is **not affiliated with or endorsed by** the official maintainers.

## Disclaimer

WordCraft is under active development. Features, browser compatibility, import/export fidelity and behavior may change between releases. Keep backups of important documents and test new versions before relying on them for critical work.
