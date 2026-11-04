# Terraria Zarf package

Package versions combine the Terraria version with a package build tag. The
current package version is `1.4.5.8-build.1`. The publishable package contains
the Terraria chart plus the TShock and vanilla runtime images, but no world
data. At deployment it can optionally turn a local directory into a minimal
OCI image using Zarf's embedded tools, push that image directly to the cluster's
Zarf registry, and import it into a new or existing PVC.

## Prerequisites

### Deployment

- The latest Zarf release
- A cluster initialized with Zarf
- Kubernetes 1.31 or newer with the `ImageVolume` feature enabled; image
  volumes are generally available in Kubernetes 1.35
- For optional import, an absolute local path containing at least one `.wld`
  file

### Local builds (optional)

- The latest Zarf release
- Maru

Maru is only required to build the package locally. Deploying the published
package does not require a local build.

## Automated image updates

Renovate opens grouped image-update PRs for vanilla and TShock, updating the
Zarf image list, default and example values, and matching image-tag schema
constraints together. Custom version ordering includes the `build.N` suffix,
so image rebuilds are detected even when the game version has not changed.
Local deploy values and world data are not scanned; automerge is disabled.

Review compatibility between TShock and vanilla before merging. Renovate uses
the vanilla image's game version to update the package prefix in `zarf.yaml`
and README references, preserving the package's independent `build.N` suffix.
TShock's version is not a Terraria game version, so it does not control that
prefix. When only an image build or
TShock changes, manually increment the package's `build.N` and README package
references before merging so CI can publish a fresh package.
CI builds PRs, and publication skips package versions already in GHCR. Update
any image overrides in your own local deploy values after reviewing the PR.

BusyBox is intentionally held at `1.36`: the published chart hardcodes that
image in init containers. The chart needs updated before changing the bundled BusyBox
image, otherwise the package may lack images required at deployment.

## Deploy the published package

Review and copy `deploy-values.example.yaml` before adding a password or other
environment-specific configuration. Deploy without a world import by omitting
`WORLD_PATH`:

```sh
cp deploy-values.example.yaml deploy-values.local.yaml
zarf package deploy \
  oci://ghcr.io/jbmay/zarf-packages/terraria:1.4.5.8-build.1 \
  --values deploy-values.local.yaml \
  --confirm
```

To import local world data, pass an absolute directory path. The deploy action
uses Zarf's embedded archiver and registry client to create and push a minimal
scratch OCI image; Docker, Podman, and Buildah are not used:

```sh
zarf package deploy \
  oci://ghcr.io/jbmay/zarf-packages/terraria:1.4.5.8-build.1 \
  --values deploy-values.local.yaml \
  --set-variables WORLD_PATH=/absolute/path/to/Worlds \
  --confirm
```

The generated tag contains the world layer's SHA-256 prefix. Temporary layer
and OCI-layout files are removed when the action finishes. Zarf's embedded
registry client handles the authenticated, mTLS-protected tunnel, and the world
image remains only in the target cluster's Zarf registry. The package attaches
Zarf's `private-registry` pull secret so the kubelet can retrieve that image in
registry-proxy mode.

Individual non-secret settings can also be overridden without a file:

```sh
zarf package deploy \
  oci://ghcr.io/jbmay/zarf-packages/terraria:1.4.5.8-build.1 \
  --set-values terraria.serverSettings.maxplayers=12 \
  --confirm
```

Choose the runtime using the provided vanilla override. Zarf includes both
images, and the values schema prevents mismatched server types and images:

```sh
zarf package deploy \
  oci://ghcr.io/jbmay/zarf-packages/terraria:1.4.5.8-build.1 \
  --values deploy-values.local.yaml \
  --values deploy-values.vanilla.yaml \
  --confirm
```

Do not put a server password on the command line because it can be retained in
shell history. Keep local deploy-value files containing secrets out of source
control.

Check the deployment and discover the allocated NodePort with:

```sh
kubectl -n terraria get deployment,pod,pvc,service
kubectl -n terraria logs deployment/terraria -c import-worlds
kubectl -n terraria get service terraria
```

The default world name is `Example`, matching `Example.wld`. On a non-empty PVC,
the importer preserves existing files unless `worldImport.overwrite` is set to
`true`. Changed world data produces a new content-derived tag automatically;
the publishable Zarf package does not need to be rebuilt.

## Build locally (optional)

Building locally is only necessary when developing or testing package changes.
From this directory, use the Maru build task:

```sh
maru run build
```

The package is written beneath the ignored `build/` directory. Remove all
generated package artifacts with:

```sh
maru run clean
```

Set a different package architecture when needed with `--set
ARCHITECTURE=arm64`. Zarf pulls chart version `0.9.1` from
`oci://ghcr.io/jbmay/helm-charts/terraria`; the TShock, vanilla, and BusyBox
runtime images are embedded directly in the resulting package. Package creation
does not read or include local world data and does not require Docker.

Pull requests build the package as a CI artifact. Merges to `main` build it
again and publish the deployable OCI package used by the commands above. The
publish workflow skips the build when that exact package version already exists
in GHCR. Increment the `build.N` suffix in `zarf.yaml` before merging changes
that should produce a new package for the same Terraria version.
