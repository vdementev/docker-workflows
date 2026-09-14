# docker-workflows

Reusable GitHub Actions workflows for the [`dementev/*` Docker Hub images](https://hub.docker.com/u/dementev) — the single source of truth for how every image is built, tested, scanned, published, and signed.

Used by: [angie](https://github.com/vdementev/angie-docker) · [nginx](https://github.com/vdementev/nginx-docker) · [adminer](https://github.com/vdementev/adminer-docker) · [mysql-percona](https://github.com/vdementev/mysql-percona-docker). [php-fpm-with-ext](https://github.com/vdementev/docker-php-fpm-with-ext) is still on its own legacy workflow; its migration is open as a pull request.

## Workflows

### `build-image.yml`

Build → optional test → Trivy vulnerability gate → multi-arch publish → Cosign keyless signing.

```yaml
jobs:
  build:
    uses: vdementev/docker-workflows/.github/workflows/build-image.yml@v1
    permissions:
      contents: read
      id-token: write
    with:
      image: angie            # Docker Hub repo, namespace comes from DOCKERHUB_USERNAME
      push: true              # false on pull_request = build/test/scan only
      tags: |
        type=raw,value=latest
    secrets: inherit
```

Key inputs (see the workflow file for the full list): `dockerfile`, `context`, `platforms`, `build-args`, `labels` (metadata-action spec), `cache-scope` (set per matrix entry; the GHA cache scope rotates daily so package layers cannot go stale), `test-command` (runs with `$IMAGE` pointing at the locally built amd64 image), `version-command` / `version-tag-suffix`, `trivy-severity` / `trivy-ignore-unfixed`, `cosign`, `timeout-minutes`.

### Version tags

`version-command` runs against the image that has just been built, tested and
scanned, and prints the upstream version on stdout. The workflow turns that into
two extra tags — `<version>` and `<major.minor>` — plus
`org.opencontainers.image.version`. Deriving the tag from the artifact rather
than from the Dockerfile means a published tag can never claim a version the
image does not contain, which matters most for the repos whose package is
deliberately unpinned.

```yaml
    with:
      version-command: docker run --rm "$IMAGE" nginx -v 2>&1 | sed -n 's|.*nginx/\([0-9][0-9.]*\).*|\1|p'
```

`version-tag-suffix` appends to both derived tags, for repos that publish one
image per flavor (`-nginx` → `6.0.2-nginx`, `6.0-nginx`). The step fails the
build if the command prints something that is not a version, so a broken
extraction cannot publish a garbage tag.

The Trivy gate fails the build on fixable CRITICAL/HIGH findings. Accepted risks go in a `.trivyignore` file in the caller repo root — it is picked up automatically.

### `sync-description.yml`

Pushes the caller's `DOCKERHUB.md` to the image's Docker Hub description. Run it `needs:`-after the build job on publish.

```yaml
  sync-description:
    needs: build
    uses: vdementev/docker-workflows/.github/workflows/sync-description.yml@v1
    with:
      image: angie
      short-description: "Alpine Angie reverse-proxy…"
    secrets: inherit
```

## Caller pattern

Each image repo carries one thin `ci.yml` triggered by `pull_request`, `push` to `main`, and `workflow_dispatch`, with a computed publish flag:

```yaml
    with:
      push: ${{ github.event_name != 'pull_request' }}
```

Pull requests build, test, and scan without publishing; pushes to `main` (and manual dispatches) publish, sign, and sync the description.

## Versioning

Semver git tags (`v1.0.0`) plus a floating major tag (`v1`) that always points at the latest release in that major. Callers reference `@v1`; breaking interface changes bump the major. Never reference `@main`.

## Action pinning policy

Every third-party action in these workflows is pinned to a full commit SHA with the version in a trailing comment. This is a hard rule: in March 2026 `aquasecurity/trivy-action` had 75 of 76 version tags force-pushed with credential-stealing code (GHSA-69fq-xp46-6x23) — mutable tags are an attack surface. Bump a pin by changing the SHA and the comment together.

## Verifying published images

Images are signed with Cosign (keyless, GitHub OIDC). Verify:

```sh
cosign verify \
  --certificate-oidc-issuer https://token.actions.githubusercontent.com \
  --certificate-identity-regexp 'github.com/vdementev/' \
  dementev/<image>@<digest>
```

SBOM and SLSA provenance (`mode=max`) attestations are pushed alongside every image; inspect with `docker buildx imagetools inspect dementev/<image> --format '{{ json .SBOM }}'`.

## The images

These workflows build and publish:

| Image | What it does |
|---|---|
| [`dementev/angie`](https://hub.docker.com/r/dementev/angie) — [source](https://github.com/vdementev/angie-docker) | Public-facing reverse proxy and TLS terminator — Angie, the nginx fork, with brotli, zstd and cache-purge |
| [`dementev/nginx`](https://hub.docker.com/r/dementev/nginx) — [source](https://github.com/vdementev/nginx-docker) | Static sites and SPAs behind that proxy — brotli/zstd siblings, Prometheus stub_status |
| [`dementev/php-fpm-with-ext`](https://hub.docker.com/r/dementev/php-fpm-with-ext) — [source](https://github.com/vdementev/docker-php-fpm-with-ext) | PHP-FPM and CLI, PHP 7.0 → 8.5, with the extensions most projects reach for |
| [`dementev/mysql-percona`](https://hub.docker.com/r/dementev/mysql-percona) — [source](https://github.com/vdementev/mysql-percona-docker) | Percona Server for MySQL 8.4 LTS, XtraBackup built in, no root inside |
| [`dementev/adminer`](https://hub.docker.com/r/dementev/adminer) — [source](https://github.com/vdementev/adminer-docker) | Adminer 6 with every driver it supports, for reaching any of the above |

## Maintainer

Built and maintained by [Vasilii Dementev](https://vasiliidementev.com) at
[Lotus Web Agency](https://lotuswebagency.com). These images are not a side
project — they are the base layer under the client and product systems we run,
which is why they are gated, tested and signed rather than pushed by hand.

Issues and pull requests:
[github.com/vdementev/docker-workflows](https://github.com/vdementev/docker-workflows).
Need this kind of infrastructure built or maintained for your own stack?
[lotuswebagency.com](https://lotuswebagency.com).

MIT licensed — see [LICENSE](LICENSE). Security policy and reporting channel:
[SECURITY.md](SECURITY.md).
