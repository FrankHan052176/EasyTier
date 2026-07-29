# OHOS HAR downstream App builds

The `EasyTier OHOS` workflow separates validation from publication:

- Ordinary branch pushes, pull requests, tag pushes, and manual runs build and
  upload a HAR artifact, but never publish it to the private registry.
- A branch push may publish only when GitHub confirms that the pushed commit is
  the merged result of a pull request for the same repository and target
  branch. The workflow requires an exact `merge_commit_sha` match, rejects
  forced pushes, and fails closed to a build-only run when the association
  cannot be confirmed.
- After the HAR is built, Core publishes it to the CodeArts private OHPM
  registry with the moving `arkts-latest` tag. Core waits until that tag
  resolves to the version just published before dispatching either App build.

This policy keeps private publishing credentials out of unmerged pull-request
builds and prevents direct commits, tags, and manual retries from publishing.

## Package identity and version

Each target branch has its own package stream. The package name is
`easytier-<branch-id>`: branch names are lowercased, `/` becomes `_`, and other
unsupported characters become `_`. Examples:

| Core branch | Private OHPM package |
| --- | --- |
| `main` | `easytier-main` |
| `ohos/pro-runtime` | `easytier-ohos_pro-runtime` |

The existing version algorithm is unchanged:

```text
<core-version>-<commits-since-tag>-<actions-run-number>-<run-attempt>-g<short-sha>
```

The Core version is selected from `easytier/Cargo.toml` and the latest Git tag
using the workflow's existing comparison rules. The commit count is calculated
from that tag to the commit actually checked out and built.

## Why `arkts-latest` is required

Do not replace the moving tag with `latest`:

- In OHPM 6.1.2.285, `latest` is a reserved tag, so
  `package@tag:latest` fails with `00623002`.
- Ordinary `latest` resolution is normalized to `*` by OHPM's
  `semverMaxSatisfying` path. CodeArts can then compare numeric-looking
  prerelease fields lexicographically and select an older package, for example
  `2.6.4-9` instead of `2.6.4-52`.

Core therefore publishes with:

```bash
ohpm publish --tag arkts-latest "${EASYTIER_PACKAGE_NAME}.har"
```

It then polls `ohpm dist-tags list` and requires `arkts-latest:` to equal the
newly generated version. A successful publish is not dispatched until that
check passes.

## Dispatch contract

After publication and tag verification, Core sends the
`core-har-published` repository dispatch to:

- `FrankHan052176/EasyTier-ArkTS`
- `FrankHan052176/easytier-pro-app`

The payload contains only package identity and source provenance:

```json
{
  "core_repository": "EasyTier/EasyTier",
  "core_ref": "refs/heads/ohos/pro-runtime",
  "package_name": "easytier-ohos_pro-runtime"
}
```

It deliberately contains no package version. The consumer always resolves the
package named by `package_name` through the verified moving tag.

## Downstream install sequence

ArkTS and Pro isolate the only private-registry request from the App project:

1. Create a minimal OHPM project under `RUNNER_TEMP`. It has no App or public
   dependencies.
2. Create temporary read-only CodeArts authentication for this resolver project
   without changing OHPM's default registry.
3. From the resolver project, install exactly one package from CodeArts:

   ```bash
   ohpm install "${package_name}@tag:arkts-latest" \
     --registry "https://devrepo.devcloud.cn-north-4.huaweicloud.com/artgalaxy/api/ohpm/cn-north-4_c07b1b38744f424b8d87a86532d38003_ohpm_1/"
   ```

4. After the install succeeds, follow the installed package link with `cd` and
   `pwd -P` to obtain its real local directory.
5. Replace the App manifest's local `easytier-ohrs` entry with the dynamic
   branch package name and a local directory spec:

   ```text
   <package_name>: file:<real-installed-package-directory>
   ```

   The App therefore consumes the package already installed in the resolver
   project; it does not contact CodeArts itself.
6. Remove the temporary private authentication/configuration.
7. From the App project, run ordinary `ohpm install` with no `--registry`
   argument. The Core dependency is local through `file:`, while every other
   dependency resolves through its normal default registry.

The CodeArts URL is scoped to the explicit Core HAR command above. Do not run
`ohpm config set registry` for CodeArts, persist CodeArts as the user or project
default, pass the private URL to the second `ohpm install`, or leave a private
`registry=` override behind. If authentication is supplied through a temporary
`.ohpmrc`, remove it (or restore the previous config) immediately after the
tagged Core package has been installed.

Consumers do not download a HAR from a GitHub artifact, extract it, rename it,
or rebuild/repack it. The package installed by OHPM is the package compiled by
Core. Consumers trust that published package rather than repeating Core's
artifact checks: they do not inspect its package metadata or resolved version,
cross-check a resolver lock, probe the native library, or validate registry
configuration. Package acquisition is allowed to fail naturally at the private
`ohpm install`; later dependency or integration problems fail the ordinary
`ohpm install` or App build.

`repository_dispatch` and manual App runs accept only `package_name`. Both
consumers log only the complete requested package spec. The concrete specs are:

| Selection | Logged/requested spec |
| --- | --- |
| Default `main` | `easytier-main@tag:arkts-latest` |
| Current Pro runtime test | `easytier-ohos_pro-runtime@tag:arkts-latest` |

There are no `package_version`, `core_sha`, or `core_version` inputs, and the
consumers do not record a resolved Core version.

## Repository configuration

Configure these Actions secrets in the Core repository:

- `CODEARTS_PRIVATE_OHPM`: publish-capable `.ohpmrc` content.
- `DOWNSTREAM_DISPATCH_TOKEN`: fine-grained GitHub token with permission to
  create repository dispatches in both destination repositories.

Configure these Actions secrets in both App repositories:

- `CODEARTS_PRIVATE_OHPM_READ`: read-only CodeArts authentication material for
  a temporary OHPM config used only by the explicit Core HAR install. It must
  not persist CodeArts as the default registry.
- `SIGNING_REPOSITORY_TOKEN`: fine-grained token with read-only access to that
  App's private signing repository.

The signing defaults are intentionally different:

| Consumer | Default signing repository | Ref | Expected configuration |
| --- | --- | --- | --- |
| ArkTS | `FrankHan052176/AppGallerySigning` | `main` | `EasyTier/signingConfigs.json`, shared root certificate/keystore files, and the EasyTier profiles |
| Pro | `FrankHan052176/EasyTierProSigning` | `main` | Root `sign.json`, publish certificate/keystore/profile, and `material/fd`, `material/ac`, `material/ce` |

Each App can override its defaults with the repository variables
`SIGNING_REPOSITORY` and `SIGNING_REPOSITORY_REF`. Signing repositories are
checked out under `RUNNER_TEMP`, rewritten into temporary build configuration,
and removed by an `always()` cleanup step.

## AGC placeholders

Both App workflows already reserve two disabled steps for future AppGallery
Connect integration:

- `ENABLE_AGC_SIGNED_APP_UPLOAD` will gate signed App upload.
- `ENABLE_AGC_TEST_RELEASE` will gate test-release publication after an upload
  returns a release ID.

These are placeholders, not working AGC integration. Leave both variables
unset or false until credentials, upload parameters, response handling, and the
test-release API are implemented; enabling either placeholder currently fails
the job deliberately.

## Activation order

Land the ArkTS and Pro consumer workflows on their default branches first and
configure their private-OHPM and signing secrets. Then configure the Core
publish and dispatch secrets. A merged Core PR will publish and dispatch only
after its HAR and `arkts-latest` verification succeed; a failed App build can
be retried manually with only the same `package_name`, without republishing or
pinning a version.
