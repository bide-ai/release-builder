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
3. Every action is pinned by full commit SHA with its tag in a comment (CI checks it). The tools that
   make the attested files (goreleaser, syft) are downloaded at a named version and checked against
   a SHA-256 kept in the workflow, never "latest" or a version range (CI checks that too).
4. Inputs are data (a Go version, a flag), validated, never code.
5. The workflows run only on `workflow_call`: they have no trigger of their own, so there is no way
   to run them with a different identity.

## What the claim covers, and what it leaves to the verifier

The Build L3 claim is about these steps: provenance made by them cannot be forged by anything in
bide's repository. It is not a claim about the caller's inputs. The caller chooses the Go version,
whether to attest a dry run, and which commit of this repository to run, and the caller's build
steps (`.goreleaser.yaml`, hooks, the Go sources) are whatever is at its commit. A release is
trustworthy only if the verifier, outside the caller's repository, requires all three of: an
attestation made by `build.yml` here, the commit of this repository it must be (below), and
`--source-ref refs/tags/<tag>`. A build of a tag is always attested and cannot be a snapshot,
whatever the caller passes.

Threats these workflows do not remove: the build steps hold the run's token and can write that
run's caches and artifacts, so they can make the release fail (for example by uploading an artifact
of the name first; `overwrite: false` turns that into a failure, not a substitution) but cannot
change what the attest job attests, which is the artifact of that name from the run, by fixed file
patterns. An administrator of this repository, or GitHub, is outside what a verifier detects.

## Releases of this repository

bide pins a commit of this repository by full SHA. A `v*` tag marks each commit bide may pin: tags
are created by administrators only and are never moved or deleted (repository ruleset), and `main`
accepts only reviewed pull requests. Moving bide to a newer builder is a pull request in bide that
changes the pinned SHA.

A pin alone is not trusted. GitHub runs any commit of this repository's fork network when it is
named by SHA, including one that only exists in a fork, and the ruleset does not apply to it. So
bide's verifier refuses a builder commit that is not an ancestor of this repository's `main`, and
a person verifying a release should use the commit of a `v*` tag listed here, not whatever the
release's `release.yml` names:

| Tag | Commit |
|---|---|
| v1.0.0 | e84bc18f472a1e1791e766f3d9d80f71e11d303b |
| v1.0.1 | f0279be1dd065a9b9cc56d06920baec9983e540f |

## Licence

Apache-2.0, as bide.
