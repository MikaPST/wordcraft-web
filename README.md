<p align="center">
  <img src="https://raw.githubusercontent.com/storytold/wordcraft/main/assets/app-icon/hicolor/256x256/apps/ai.storyteller.wordcraft.png" width="128" height="128" alt="WordCraft icon">
</p>

<h1 align="center">WordCraft Web — Docker Image</h1>
<p align="center">
  A self-hosted browser build of WordCraft, an open-source word processor written in Rust.
  <br>
  Packaged as a static web application and published to GitHub Container Registry.
</p>
<p align="center">
  <a href="https://github.com/MikaPST/wordcraft-web/actions/workflows/publish-image.yml">
    <img src="https://github.com/MikaPST/wordcraft-web/actions/workflows/publish-image.yml/badge.svg" alt="Build and publish WordCraft Web">
  </a>
  <a href="https://github.com/MikaPST/wordcraft-web/releases/latest">
    <img src="https://img.shields.io/github/v/release/MikaPST/wordcraft-web?label=Release&logo=github&color=blue" alt="Latest Release">
  </a>
  <a href="https://github.com/MikaPST/wordcraft-web/pkgs/container/wordcraft-web">
    <img src="https://ghcr-badge.egpl.dev/mikapst/wordcraft-web/latest_tag?trim=major&label=GHCR" alt="GHCR image tag">
  </a>
  <a href="docker-compose.yml">
    <img src="https://img.shields.io/badge/Docker%20Compose-ready-2496ED?logo=docker&logoColor=white" alt="Docker Compose configuration">
  </a>
</p>

---

## Why this repository?

This repository provides a Docker image for the official [WordCraft](https://github.com/storytold/wordcraft) web application.

The goal is to make WordCraft easy to deploy on a self-hosted server such as Unraid, without having to build the application manually.

- Publish a matching GitHub Release only after the Docker build and push succeed.
- Build the web application directly from the official upstream repository.
- Automatically publish the resulting Docker image to GitHub Container Registry (GHCR).
- Rebuild periodically to follow upstream changes.
- Provide a simple Docker Compose configuration.
- Make the image suitable for Unraid and other Docker environments.
- Keep the Docker packaging separate from the upstream application.

This repository is a packaging project only. It is not a fork of WordCraft and does not modify the upstream source code.

## How it works

WordCraft is an open-source word processor written in Rust. Its web version runs in a modern browser through WebAssembly and uses an `eframe`/`wgpu` interface, with WebGPU and a WebGL2 fallback.

WordCraft provides a familiar word-processing workflow with a ribbon interface, styles, tables, page layout, references, review tools and document formatting. The upstream application reads and writes DOCX, and also supports formats such as ODT, RTF, HTML, Markdown, LaTeX and TXT. PDF export is also available in the application. WordCraft is currently in alpha, and document fidelity and feature depth are still evolving.

The Docker image:

- Serves the prebuilt WordCraft web application as static files using NGINX.
- Does not run a desktop session or remote desktop.
- Does not add a server-side document-processing engine.
- Does not provide server-side user accounts or centralized document storage.

Documents are opened through the browser and saved/exported through browser downloads. Preferences are stored in the browser's local storage; they are not persisted in a Docker volume. The container itself does not store users' documents.

## Docker image

The image is published on GitHub Container Registry:

```text
ghcr.io/mikapst/wordcraft-web:latest
```

Each official upstream release is also published under its exact version tag, for example:

```text
ghcr.io/mikapst/wordcraft-web:vX.Y.Z
```

Use a versioned tag if you want to keep a deployment on a specific release. The `latest` tag updates when an image is published.

### Option 1 — Docker Compose (recommended)

Create a `docker-compose.yml` file:

```yaml
services:
  wordcraft-web:
    image: ghcr.io/mikapst/wordcraft-web:latest
    container_name: wordcraft-web
    ports:
      - "8080:8080"
    restart: unless-stopped
```

Start the container:

```bash
docker compose up -d
```

Open WordCraft in your browser:

```text
http://YOUR_SERVER_IP:8080
```

#### Useful Compose commands

View container logs:

```bash
docker compose logs -f
```

Update the image:

```bash
docker compose pull
docker compose up -d
```

Stop and remove the container:

```bash
docker compose down
```

The container listens on port `8080`. No volume or environment variable is required by this static web image.

### Option 2 — Run directly with Docker

Pull the image:

```bash
docker pull ghcr.io/mikapst/wordcraft-web:latest
```

Run the container:

```bash
docker run -d \
  --name wordcraft-web \
  --restart unless-stopped \
  -p 8080:8080 \
  ghcr.io/mikapst/wordcraft-web:latest
```

Then open:

```text
http://YOUR_SERVER_IP:8080
```

### Network access and reverse proxies

The container exposes an HTTP service on port `8080`.

You can place it behind a reverse proxy such as Traefik or NGINX Proxy Manager to access it through a domain name and HTTPS.

For public access, use HTTPS through a reverse proxy rather than exposing the unencrypted HTTP endpoint directly to the Internet.

WordCraft uses browser graphics APIs. WebGPU and clipboard access require a secure context (HTTPS, or localhost); WebGL2 is used as a fallback when WebGPU is unavailable. The upstream application does not require `Cross-Origin-Opener-Policy` or `Cross-Origin-Embedder-Policy`.

The NGINX configuration serves WebAssembly files with the correct MIME type, enables gzip compression, disables caching for `index.html` and applies long-lived immutable caching to hashed JavaScript and WebAssembly assets.

## Automated builds

GitHub Actions checks the official [WordCraft releases](https://github.com/storytold/wordcraft/releases), builds the WebAssembly application from the selected upstream tag and publishes the image to GHCR.

| Trigger | Behaviour |
|---|---|
| Weekly schedule | Checks the latest official WordCraft release and builds it if it has not already been published here |
| Manual workflow dispatch | Builds the latest official release, or a specified `upstream_tag` |
| Force rebuild | Rebuilds an already published version with `force_rebuild: true` |
| Changes to packaging files on `main` | Rebuilds the current official upstream release with the updated workflow |

### Published tags

- `latest` — the latest successfully published image (mutable tag).
- `<upstream-version>` — the exact official upstream tag, for example `vX.Y.Z`.
- `upstream-<commit>` — the upstream source commit used for the image (first 12 hexadecimal characters).

The image targets both `linux/amd64` and `linux/arm64`. The workflow checks out **the official upstream release tag**, rather than compiling `main` as the versioned source.

### GitHub Releases

After a successful Docker build and GHCR push, the workflow creates or updates a [GitHub Release](https://github.com/MikaPST/wordcraft-web/releases) matching the upstream tag. It includes the upstream release notes, source link and commit, Docker tags, supported architectures and (when available) the published image digest.

The Release badge at the top of this README updates automatically. Scheduled runs skip versions already released here. You can force a rebuild from **Actions → Build and publish WordCraft Web → Run workflow**.

> [!IMPORTANT]
> A GitHub Release is created only after the Docker image has been successfully published. The `latest` tag may change after a rebuild; pin a versioned image tag for a stable deployment. Auxiliary source repositories (for example `craft-fonts`) may be checked out separately and are not necessarily pinned to the upstream release commit.

## Unraid Community Applications

This image can be used with Unraid through Docker Compose or a Community Applications template.

The Unraid template is maintained separately in the [MikaPST/unraid-community-apps](https://github.com/MikaPST/unraid-community-apps) repository.

This repository contains the Docker image build and publication workflow; the Community Applications repository contains the Unraid-specific template.
- **Template:** [wordcraft-web.xml](https://github.com/MikaPST/unraid-community-apps/blob/main/templates/wordcraft-web.xml)

## Upstream project

**WordCraft** is an open-source, clean-room reimplementation of a modern word processor, written in pure Rust.

Its features include:

- DOCX document reading and writing.
- Text and paragraph formatting, styles, themes and lists.
- Tables, pictures, shapes, text boxes, headers and footers.
- Page layout, columns, page numbers and watermarks.
- References, table of contents, footnotes, citations and bibliography.
- Spelling and grammar tools, comments and track changes.
- Import and export for ODT, RTF, HTML, Markdown, LaTeX and TXT.
- PDF export and page image rendering.

The upstream project also provides native desktop applications, a command-line interface and MCP integrations. These are not included in this Docker image, which packages only the browser-based WebAssembly application.

- **Source code:** [github.com/storytold/wordcraft](https://github.com/storytold/wordcraft)
- **Official website:** [getartcraft.com/apps/wordcraft](https://getartcraft.com/apps/wordcraft)
- **Releases:** [GitHub Releases](https://github.com/storytold/wordcraft/releases)
- **Web hosting guide:** [Hosting WordCraft for the web](https://github.com/storytold/wordcraft/blob/main/packaging/web/README.md)

This repository packages the upstream web application for Docker. It does not modify the upstream application.

The MikaPST GitHub Release tracks the upstream version but represents independent Docker packaging, not an official WordCraft distribution.

## License and attribution

WordCraft is dual-licensed under **MIT** or **Apache-2.0**, at your option. Copyright and required notices are maintained by the upstream project in its `NOTICE` file.

Bundled assets, fonts and language data may have their own licenses. Their authors, sources and licenses are documented by the upstream project in `ATTRIBUTION.md` and the relevant dependency repositories.

The WordCraft and ArtCraft names, logos and related brand assets are trademarks of the upstream project and are not covered by the software license.

The Docker packaging files in this repository are provided separately from the upstream application. Refer to the upstream repository for the complete license texts, notices and asset attributions.

## Disclaimer

This is an independent community packaging project and is not affiliated with, endorsed by, or supported by the ArtCraft team.

All product names, trademarks and logos are the property of their respective owners.
