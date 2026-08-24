# build-deploy-docker

Reusable GitHub Actions workflow that builds a multi-arch (arm64 + amd64) Docker image from a **pre-built application artifact** and pushes it to a Docker registry.

Derived from `p6m-dev/github-actions/.github/workflows/build-deploy-docker.yaml`. The key difference is that this workflow does **not** invoke `yarn install` / `yarn build` (or any equivalent) inside the docker build — the caller is expected to build the arch-independent application payload once, upload it as an artifact, and pass the artifact name in. Each per-arch job then only performs the arch-specific assembly (COPY the artifact + runtime install of native modules).

## Why

The upstream workflow ran the full application build twice — once per arch — which for large JS/TS builds is the dominant cost. Splitting the arch-independent build out of the docker step cuts wall-clock roughly in half for the docker phase and eliminates cache-scope duplication.

## Usage

```yaml
jobs:
  build-core:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v6
      # ... run your arch-independent build here ...
      - uses: actions/upload-artifact@v7
        with:
          name: app-artifact
          path: build-output/

  docker:
    needs: build-core
    uses: Ashley-Furniture/build-deploy-docker/.github/workflows/build.yaml@v1
    with:
      context: my-app
      dockerfile: my-app/Dockerfile.assemble
      image-name: my-org/my-app
      artifact-name: app-artifact
      tags: |
        type=raw,value=latest
        type=sha,prefix=
    secrets:
      username: ${{ secrets.REGISTRY_USERNAME }}
      password: ${{ secrets.REGISTRY_PASSWORD }}
```

`Dockerfile.assemble` should COPY the artifact from `<context>/.build-artifact/` (default) and perform only the arch-specific runtime install.

## Inputs

| Input             | Required | Default             | Description                                                                          |
| ----------------- | -------- | ------------------- | ------------------------------------------------------------------------------------ |
| `image-name`      | no       | `${{ github.repository }}` | Namespaced image name (no registry prefix).                                    |
| `context`         | no       | `.`                 | Docker build context path.                                                           |
| `dockerfile`      | no       | `./Dockerfile.assemble` | Path to the assemble Dockerfile.                                                 |
| `tags`            | yes      | —                   | Tags for the final multi-arch manifest (`docker/metadata-action` format).            |
| `artifact-name`   | yes      | —                   | Name of the artifact uploaded via `actions/upload-artifact`.                         |
| `artifact-path`   | no       | `.build-artifact`   | Path under the context where the artifact is extracted before docker build.          |
| `build-args`      | no       | `""`                | Build args (`key=value` per line).                                                   |
| `registry`        | no       | `vars.P6M_ARTIFACTORY_HOSTNAME` | Docker registry hostname.                                                |
| `linux-arm-runner`| no       | `ubuntu-24.04-arm`  | `runs-on` tag for the arm64 runner.                                                  |
| `push`            | no       | `true`              | Push built images (set false to only refresh cache).                                 |
| `github_ref`      | no       | `""`                | Ref to checkout.                                                                     |

## Outputs

| Output   | Description                                        |
| -------- | -------------------------------------------------- |
| `digest` | The digest of the final multi-arch manifest.       |
| `tags`   | The tags produced by `docker/metadata-action`.     |
