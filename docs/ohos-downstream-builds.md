# OHOS HAR downstream App builds

The `EasyTier OHOS` workflow publishes the branch-scoped HAR first and only
then dispatches App builds. The canonical production stream is:

1. A push to `ohos/pro-runtime` builds
   `easytier-ohos_pro-runtime@<exact-version>`.
2. Core publishes that exact package to the CodeArts private OHPM registry.
3. Core sends `core-har-published` to the ArkTS and EasyTier Pro repositories.
4. Each consumer installs the exact package version from the dispatch payload,
   verifies its package metadata and Core commit, then repacks it under the
   stable local module name `easytier-ohrs` used by the application source.
5. Each consumer builds and verifies a signed `publish`/`release` App artifact.

No consumer selects a package by CodeArts list order or a moving latest tag.
`repository_dispatch` works across users and organizations; the token only
needs access to the destination repositories.

## Repository configuration

Configure these Actions secrets in the Core repository:

- `CODEARTS_PRIVATE_OHPM`: publish-capable `.ohpmrc` content.
- `DOWNSTREAM_DISPATCH_TOKEN`: fine-grained GitHub token with `Contents: Read
  and write` on `FrankHan052176/EasyTier-ArkTS` and
  `FrankHan052176/easytier-pro-app` (required by the repository-dispatch API).

Configure these Actions secrets in both App repositories:

- `CODEARTS_PRIVATE_OHPM_READ`: read-only `.ohpmrc` content for the private
  registry.
- `SIGNING_REPOSITORY_TOKEN`: fine-grained, read-only token for the signing
  repository.

The App workflows default to
`FrankHan052176/AppGallerySigning` on `main`. They can be changed with the
repository variables `SIGNING_REPOSITORY` and `SIGNING_REPOSITORY_REF`.

## Signing repository layout

The private signing repository must contain the following files. Its JSON
files may contain local absolute paths: the App workflows rewrite material
paths to the temporary checkout before invoking Hvigor.

```text
AppGallerySigning/
├── FrankHan.p12
├── FrankHan_Debug.cer
├── FrankHan_Publish.cer
├── EasyTier/
│   ├── signingConfigs.json
│   ├── EasyTier_DebugDebug.p7b
│   └── EasyTier_PublishRelease.p7b
└── EasyTierPro/
    ├── signingConfigs.json
    ├── EasyTierPro_DebugDebug.p7b
    └── EasyTierPro_PublishRelease.p7b
```

Both JSON files must contain `default` and `publish` signing entries. Signing
material is checked out under `RUNNER_TEMP`, is never committed to either App
repository, and is removed in an `always()` cleanup step.

## Activation order

Push the downstream workflow files to the default branches first, configure
their read/signing secrets, then configure the Core dispatch token and push the
Core workflow. A manual downstream run accepts the exact private package name
and version and can be used to retry a failed App build without republishing
the HAR.
