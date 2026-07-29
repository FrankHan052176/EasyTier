# OHOS workflow

The `ohos` workflow builds the Core HAR on pushes, pull requests, tags, and
manual runs. A build is published only when a branch push is the merge commit
of a pull request. Direct pushes and manual runs remain build-only.

## Package

Each Core branch has one private OHPM package:

| Core branch | Package |
| --- | --- |
| `main` | `easytier-main` |
| `ohos/pro-runtime` | `easytier-ohos_pro-runtime` |

The version format is:

```text
<core-version>-<commits-since-tag>-<run-number>-<run-attempt>-g<short-sha>
```

Merged pull requests publish with the moving tag `arkts-latest` and then send
`core-har-published` to the ArkTS and Pro repositories. The payload contains
only `core_repository`, `core_ref`, and `package_name`.

## App install sequence

ArkTS and Pro use the same three OHPM commands:

```bash
ohpm uninstall easytier-ohrs
ohpm install "$CORE_HAR_PACKAGE@tag:arkts-latest" \
  --registry "$CORE_HAR_REGISTRY"
ohpm install
```

The App workflow then reads the installed version from:

```text
oh_modules/<package_name>/oh-package.json5
```

The existing `oh-package-lock.json5` and `oh_modules` directory are not
manually deleted. The source import is changed from `easytier-ohrs` to the
branch package selected by `package_name` only inside the Actions checkout.

## Secrets

Core requires:

- `CODEARTS_PRIVATE_OHPM`: publish-capable OHPM configuration.
- `DOWNSTREAM_DISPATCH_TOKEN`: permission to dispatch both App repositories.

ArkTS and Pro require:

- `CODEARTS_PRIVATE_OHPM_READ`: read-only private OHPM authentication.
- `SIGNING_REPOSITORY_TOKEN`: read access to the corresponding private signing
  repository.

ArkTS defaults to `FrankHan052176/AppGallerySigning`; Pro defaults to
`FrankHan052176/EasyTierProSigning`. Both App workflows build a signed
`publish/release` App and leave AGC upload and test release as TODO items.
