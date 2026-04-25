# Shiny to Electron

[![Test](https://github.com/coatless-actions/shiny-to-electron/actions/workflows/test.yml/badge.svg)](https://github.com/coatless-actions/shiny-to-electron/actions/workflows/test.yml)

A composite GitHub Action that builds a Shiny app as an Electron desktop installer using the [`shinyelectron`](https://github.com/coatless-rpkg/shinyelectron) R package.

The action wraps the full build pipeline so a workflow shrinks from ~200 lines of YAML to a few. It picks up the runner's OS and architecture, installs R (and Python when needed), installs `shinyelectron`, runs `shinyelectron::export()`, and uploads the build output as a workflow artifact.

## Quickstart

```yaml
name: Build Electron app

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      matrix:
        os: [ubuntu-latest, macos-latest, windows-latest]
    steps:
      - uses: actions/checkout@v6
      - uses: coatless-actions/shiny-to-electron@v1
        with:
          appdir: app
```

That is it. Each runner produces a platform-specific installer as a workflow artifact named `<app-name>-<platform>-<arch>`.

## Inputs

| Input | Description | Default |
|-------|-------------|---------|
| `appdir` | Path to the Shiny app directory inside the repository | `'app'` |
| `app-name` | Display name of the application | `appdir` basename |
| `app-type` | `r-shiny` or `py-shiny` | autodetect from app files |
| `runtime-strategy` | `shinylive`, `bundled`, `system`, `auto-download`, or `container` | from `_shinyelectron.yml` or `shinylive` |
| `platform` | `mac`, `win`, or `linux` | derived from `runner.os` |
| `arch` | `x64` or `arm64` | derived from `runner.arch` |
| `destdir` | Build output directory | `'build'` |
| `node-version` | Node.js version to install | `'22'` |
| `r-version` | R version to install (`release`, `devel`, `oldrel`, or specific) | `'release'` |
| `python-version` | Python version (used only for `py-shiny` apps) | `'3.12'` |
| `sign` | Code-sign the build (`'true'` or `'false'`) | `'false'` |
| `artifact-name` | Name for the uploaded artifact | `<app-name>-<platform>-<arch>` |
| `upload-artifact` | Upload the build as a workflow artifact (`'true'` or `'false'`) | `'true'` |
| `shinyelectron-source` | Remotes-style source spec for the `shinyelectron` R package. Pin a tag or branch with `@v0.2.0` or `@some-branch`. Switch to `any::shinyelectron` once a CRAN release is available. | `'github::coatless-rpkg/shinyelectron'` |

## Outputs

| Output | Description |
|--------|-------------|
| `electron-app-path` | Path to the built Electron application directory |
| `dist-path` | Path to the `dist/` directory containing installers |
| `artifact-name` | Name used for the uploaded artifact |
| `app-type` | Resolved app type (`r-shiny` or `py-shiny`) |

## Examples

### Cross-platform release with a tag

Cuts a release whenever a `v*` tag is pushed.

```yaml
name: Release

on:
  push:
    tags: ['v*']

jobs:
  build:
    runs-on: ${{ matrix.os }}
    strategy:
      fail-fast: false
      matrix:
        include:
          - os: macos-latest
          - os: macos-15-intel
          - os: windows-latest
          - os: windows-11-arm
          - os: ubuntu-latest
          - os: ubuntu-24.04-arm
    steps:
      - uses: actions/checkout@v6
      - uses: coatless-actions/shiny-to-electron@v1
        with:
          appdir: app
          app-name: MyApp

  release:
    needs: build
    if: startsWith(github.ref, 'refs/tags/v')
    runs-on: ubuntu-latest
    permissions:
      contents: write
    steps:
      - uses: actions/download-artifact@v8
        with:
          path: artifacts
      - uses: softprops/action-gh-release@v3
        with:
          files: artifacts/**/*
          generate_release_notes: true
```

### Mixed runtime strategies per platform

Linux ships as `shinylive` (no portable R for Linux); macOS and Windows bundle R alongside the app.

```yaml
strategy:
  matrix:
    include:
      - os: ubuntu-latest
        runtime: shinylive
      - os: macos-latest
        runtime: bundled
      - os: windows-latest
        runtime: bundled
steps:
  - uses: actions/checkout@v6
  - uses: coatless-actions/shiny-to-electron@v1
    with:
      appdir: app
      runtime-strategy: ${{ matrix.runtime }}
```

### Code signing

Pass certificates and Apple credentials as secrets in `env`. The action passes `sign = TRUE` to `shinyelectron::export()`, and `electron-builder` reads the standard environment variables.

```yaml
- uses: coatless-actions/shiny-to-electron@v1
  with:
    appdir: app
    sign: 'true'
  env:
    # macOS
    CSC_LINK: ${{ secrets.MAC_CERTIFICATE }}
    CSC_KEY_PASSWORD: ${{ secrets.MAC_CERTIFICATE_PASSWORD }}
    APPLE_ID: ${{ secrets.APPLE_ID }}
    APPLE_APP_SPECIFIC_PASSWORD: ${{ secrets.APPLE_APP_SPECIFIC_PASSWORD }}
    # Windows
    WIN_CSC_LINK: ${{ secrets.WIN_CERTIFICATE }}
    WIN_CSC_KEY_PASSWORD: ${{ secrets.WIN_CERTIFICATE_PASSWORD }}
```

See the [Code Signing vignette](https://r-pkg.thecoatlessprofessor.com/shinyelectron/articles/code-signing.html) for the full setup.

### Python (py-shiny) app

The action autodetects `py-shiny` when it sees `app.py`, `requirements.txt`, or `pyproject.toml` in `appdir`. Pass `app-type` explicitly when autodetect needs help:

```yaml
- uses: coatless-actions/shiny-to-electron@v1
  with:
    appdir: app
    app-type: py-shiny
    python-version: '3.12'
```

### Reading outputs

```yaml
- id: build
  uses: coatless-actions/shiny-to-electron@v1
  with:
    appdir: app

- name: Show build outputs
  run: |
    echo "Electron app: ${{ steps.build.outputs.electron-app-path }}"
    echo "Dist dir:     ${{ steps.build.outputs.dist-path }}"
    echo "Artifact:     ${{ steps.build.outputs.artifact-name }}"
```

## How it works

The action is a [composite action](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action) that runs the same recipe as the bundled `inst/templates/github-actions-build.yml` workflow shipped with `shinyelectron`:

1. Resolve platform, architecture, app type, and names from inputs and runner context.
2. Set up R via [`r-lib/actions/setup-r`](https://github.com/r-lib/actions).
3. Set up Python via [`actions/setup-python`](https://github.com/actions/setup-python) when the app is `py-shiny`.
4. Set up Node.js via [`actions/setup-node`](https://github.com/actions/setup-node).
5. Install Linux system dependencies (`libcurl`, `libxml2`, `libssl`).
6. Install `shinyelectron` and `shinylive` via [`r-lib/actions/setup-r-dependencies`](https://github.com/r-lib/actions).
7. Run `shinyelectron::export()` with the resolved arguments.
8. Upload the build output as a workflow artifact.

If you need finer control than the action provides, use the [bundled template](https://github.com/coatless-rpkg/shinyelectron/blob/main/inst/templates/github-actions-build.yml) directly.

## Versioning

Pin the action by major version (`@v1`) for painless upgrades, by full tag (`@v1.0.0`) for strict reproducibility, or by SHA for the strictest pin.

## Documentation

- [shinyelectron home](https://r-pkg.thecoatlessprofessor.com/shinyelectron/)
- [Building with GitHub Actions](https://r-pkg.thecoatlessprofessor.com/shinyelectron/articles/github-actions.html)
- [Configuration reference](https://r-pkg.thecoatlessprofessor.com/shinyelectron/articles/configuration.html)
- [Code signing](https://r-pkg.thecoatlessprofessor.com/shinyelectron/articles/code-signing.html)

## License

[AGPL-3.0](LICENSE).
