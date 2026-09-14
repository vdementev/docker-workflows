# Security policy

## Reporting a vulnerability

Please report privately rather than opening a public issue.

- [GitHub private vulnerability reporting](https://github.com/vdementev/docker-workflows/security/advisories/new) (preferred)
- `security@lotuswebagency.com`

Useful in a report: the tag or digest you found it in, the CVE or a
reproduction, and what an attacker gets out of it. If you have a fix, a pull
request is welcome, but send the report first.

## Response targets

Business days, Asia/Bangkok.

| Stage | Target |
|---|---|
| Acknowledgement | 2 business days |
| Triage and severity call | 5 business days |
| Published fix — fixable CRITICAL or HIGH | 7 days from triage |
| Published fix — everything else | the next weekly rebuild |

Fixes ship as a new `v1.x` release of the reusable workflows; callers pinned to
`@v1` pick them up on their next run.

## What is already automated

Every third-party action used here is pinned to a full commit SHA with the
version in a trailing comment — in March 2026 `aquasecurity/trivy-action` had 75
of its 76 version tags force-pushed with credential-stealing code
(GHSA-69fq-xp46-6x23), and a moving tag in a workflow is a supply-chain hole.
A pull request that introduces an unpinned action will not be merged.

These workflows hold the Docker Hub credentials of every `dementev/*` image, so
treat anything that could exfiltrate a secret from a workflow run — script
injection through an input, an unpinned action, an overbroad `permissions:`
block — as a high-severity report.

Findings that are accepted rather than fixed (no upstream fix available, or not
reachable in this image) are recorded in a `.trivyignore` file in this
repository together with the reason, so the gate stays meaningful instead
of being switched off.

## Scope

In scope: anything shipped from this repository.

Out of scope: vulnerabilities in upstream projects that we only package — report
those upstream, and tell us so we can pin or patch around them; findings that
require an already-compromised host or Docker daemon; and unfixable CVEs already
listed in `.trivyignore` with a reason.
