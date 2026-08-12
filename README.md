# docker-workflows

Reusable GitHub Actions workflows for the [`dementev/*` Docker Hub images](https://hub.docker.com/u/dementev) — the single source of truth for how every image is built, tested, scanned, published, and signed.

Used by: [angie](https://github.com/vdementev/angie-docker) · [nginx](https://github.com/vdementev/nginx-docker) · [adminer](https://github.com/vdementev/docker-adminer-standalone) · [php-fpm-with-ext](https://github.com/vdementev/docker-php-fpm-with-ext) · [mysql-percona](https://github.com/vdementev/mysql-percona-docker)

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

Key inputs (see the workflow file for the full list): `dockerfile`, `context`, `platforms`, `build-args`, `labels` (metadata-action spec), `cache-scope` (set per matrix entry), `test-command` (runs with `$IMAGE` pointing at the locally built amd64 image), `trivy-severity` / `trivy-ignore-unfixed`, `cosign`, `timeout-minutes`.

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
