# Reproduction for renovatebot/renovate#47270

Flux `HelmRelease` PRs are rebuilt and force-pushed on every run, although nothing changed, when the `HelmRepository` is defined in another file.

## Setup

- [`flux/repositories/podinfo.yaml`](./flux/repositories/podinfo.yaml): an OCI `HelmRepository` (`oci://ghcr.io/stefanprodan/charts`)
- [`flux/apps/podinfo.yaml`](./flux/apps/podinfo.yaml): a `HelmRelease` for `podinfo` 6.5.0, which references that repository by name
- [`renovate.json`](./renovate.json): `rebaseWhen: conflicted`, so Renovate should reuse the branch once it exists

## Steps

1. Run Renovate once with `LOG_LEVEL=debug`. It creates a PR updating `podinfo`.
2. Optional, to see the force-push: push an unrelated commit to `main` (e.g. edit this README).
3. Run Renovate again with `LOG_LEVEL=debug` and without a matching repository cache (`RENOVATE_REPOSITORY_CACHE=disabled`, or delete the cache).

With a repository cache whose branch fingerprint matches, Renovate skips `getUpdatedPackageFiles()` for the branch and the bug doesn't show.

## Expected

The second run reuses the existing branch (`Using reuseExistingBranch: true`, `Branch dep podinfo in flux/apps/podinfo.yaml is already updated`) and doesn't push.

## Actual

The second run rejects the existing branch and rebuilds it from `main`:

```
DEBUG: Using reuseExistingBranch: true
DEBUG: manager.getUpdatedPackageFiles() reuseExistingBranch=true
DEBUG: Rebasing branch after deps list has changed (packageFile=flux/apps/podinfo.yaml)
       "depName": "podinfo"
DEBUG: manager.getUpdatedPackageFiles() reuseExistingBranch=false
```

When `main` didn't move, the rebuilt content is identical and Renovate skips the commit (`No file changes detected`).
When `main` moved (step 2), the rebuilt branch is force-pushed, although there is no conflict and `rebaseWhen` is `conflicted`.

## Cause

The repository scan (`extractAllPackageFiles()`) resolves the `HelmRepository` from the other file, so the dependency gets `packageName` `ghcr.io/stefanprodan/charts/podinfo`.
The branch reuse check (`checkBranchDepsMatchBaseDeps()`) re-extracts only `flux/apps/podinfo.yaml` with `extractPackageFile()`, which doesn't see the other file, so the dependency has `skipReason: unknown-registry` and a different `packageName`, and the dependency signatures never match.

## Workaround

Map the `sourceRef` name in `flux.registryAliases` (`"podinfo": "oci://ghcr.io/stefanprodan/charts"`).
