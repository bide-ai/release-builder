# release-builder

The isolated reusable release workflows for [bide](https://github.com/bide-ai/bide). bide's
`release.yml` calls them to build a release and to attest it with SLSA v1 build provenance, so that
the provenance is made by steps that a change to bide's own repository cannot alter. That separation
is what supports a SLSA Build Level 3 claim for bide releases from v0.21.0 on (see
[Verifying releases](https://github.com/bide-ai/bide/blob/main/docs/trust/verify-releases.md) in bide
for what it proves and what it does not).

## Workflows

| File | What it does |
|---|---|
| `.github/workflows/build.yml` | `build` job: goreleaser builds the archives, per-archive SBOMs and `checksums.txt`, and the approve page `SHA256SUMS` is built, with a read-only token and no `id-token`. `attest` job: attests those files; it holds `id-token` and `attestations: write` and runs none of the caller's code. |
| `.github/workflows/module-sbom.yml` | The same two jobs for the SBOM of bide's published Go module set. |

Neither signs with cosign nor publishes anything: that stays in bide's `release.yml`, which verifies
the attestations first (`gh attestation verify`, pinned to this repository, workflow and commit).

## Rules for changing these files

These keep the isolation true. A change that breaks one of them breaks the Build L3 claim, so review
it as a change to bide's release security.

1. The job that runs the caller's code (`build`, `generate`) has no `id-token` and no `attestations`
   permission, and its token is `contents: read`.
2. The `attest` job runs no code from the caller's repository: no checkout, no script, no input
   that becomes a command. It attests fixed file patterns from the artifact the build job uploaded.
3. Every action is pinned by full commit SHA with its tag in a comment (CI checks it). Tools the
   workflows download are pinned by version and checked by checksum.
4. Inputs are data (a Go version, a flag), validated, never code.
5. The workflows run only on `workflow_call`: they have no trigger of their own, so there is no way
   to run them with a different identity.

## Releases of this repository

bide pins a commit of this repository by full SHA. A `v*` tag marks each commit bide may pin: tags
are created by administrators only and are never moved or deleted (repository ruleset), and `main`
accepts only reviewed pull requests. Moving bide to a newer builder is a pull request in bide that
changes the pinned SHA, and the signer digest bide verifies is read from that same line.

## Licence

Apache-2.0, as bide.
