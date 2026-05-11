# bundle-roc-library

A GitHub Action to bundle and release a Roc library.

The action works with both the legacy Roc compiler (which exposes
`roc build --bundle`) and the new Roc compiler (which exposes `roc bundle`).
The CLI is autodetected at runtime by probing for the `roc bundle`
subcommand — no configuration is required.

## Inputs

| Input         | Required | Default                                | Notes                                                                                                                            |
| ------------- | -------- | -------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `library`     | yes      | —                                      | Path to the library's entrypoint file.                                                                                           |
| `roc-path`    | no       | `roc`                                  | Path to the Roc executable.                                                                                                      |
| `bundle-type` | no       | `.tar.br`                              | Legacy CLI only. One of `.tar`, `.tar.gz`, `.tar.br`. Ignored with a warning on the new CLI, which always produces `.tar.zst`.   |
| `compression` | no       | _(unset)_                              | New CLI only. zstd compression level (1–22). Ignored with a warning on the legacy CLI. If unset, the compiler's default is used. |
| `release`     | no       | `true` on release events, else `false` | Whether to upload the bundle to the repository's releases.                                                                       |
| `tag`         | no       | `${{ github.ref }}`                    | Tag of the release to upload to.                                                                                                 |
| `token`       | no       | `${{ github.token }}`                  | GitHub token used to upload the release asset.                                                                                   |

## Usage

```yaml
name: Example workflow

on:
  # Run when a release is published
  release:
    types:
      - published

jobs:
  bundle-and-release:
    name: Bundle and release library
    runs-on: ubuntu-latest
    permissions:
      contents: write # Used to upload the bundled library
    steps:
      - name: Check out the repository
        uses: actions/checkout@v3
      - name: Install Roc
        uses: roc-lang/setup-roc@main
        with:
          version: nightly-new-compiler # or e.g. alpha4-rolling for the legacy compiler
      - name: Bundle and release the library
        uses: hasnep/bundle-roc-library@main
        with:
          library: path/to/main.roc # Path to the library's entrypoint
          # compression: 19          # optional, new CLI only
          token: ${{ github.token }}
```
