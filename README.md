# carrel-deploy

> Deploy a GitHub repository to a carrel machine or function on every push.

This action uploads a built artifact to a [carrel](https://github.com/NYCU-WinLab/carrel) app and
waits until the machine serves it. The run fails when the machine reports a failed deploy.

The action signs in with a GitHub OIDC token. The repository stores no carrel secret.

## Usage

1. Ask your agent to link the repository. It calls the carrel MCP tool `app_link`.
2. Run the returned install line on the machine over SSH.
3. Commit the returned workflow to `.github/workflows/carrel.yml`.

```yaml
name: carrel
on:
  push:
    branches: [main]
  workflow_dispatch:
permissions:
  contents: read
  id-token: write
jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci && npm run build && tar czf app.tar.gz -C .next/standalone .
      - uses: NYCU-WinLab/carrel-deploy@v1
        with:
          tenant: alice
          app: site
          artifact: app.tar.gz
```

## Functions

A carrel function deploys an image instead of an artifact. Ask your agent to call `function_link`
after the first `function_deploy`, then commit the returned workflow. It builds the image, pushes
it to ghcr.io and sends its digest:

```yaml
      - id: image
        uses: docker/build-push-action@v6
        with:
          push: true
          platforms: linux/amd64
          tags: ghcr.io/alice/hook:${{ github.sha }}
      - uses: NYCU-WinLab/carrel-deploy@v1
        with:
          tenant: alice
          function: hook
          image: ghcr.io/alice/hook@${{ steps.image.outputs.digest }}
```

The new version keeps the function's settings and environment. The run fails when the build
fails. A ghcr.io package is private by default: make it public, or give the function credentials
with `function_registry`.

## Inputs

| Input | Required | Default | Meaning |
|---|---|---|---|
| `tenant` | yes | | carrel tenant that owns the app or function |
| `app` | for an app | | app name given to `app_link` |
| `artifact` | for an app | | `.tar.gz` with what the start command runs, at most 500 MB |
| `function` | for a function | | function name given to `function_link` |
| `image` | for a function | | the pushed image named by its digest |
| `url` | no | `https://carrel.winlab.tw` | carrel origin |
| `timeout` | no | `300` | seconds to wait for `live` or `failed` |

## What carrel checks

1. The token comes from GitHub Actions and names this carrel as its audience.
2. The repository and branch match the linked app or function.
3. The event is `push` or `workflow_dispatch`.
4. The repository id matches the first accepted deploy.

The design is in carrel PLAN sections 7.9 (apps) and 7.11 (functions).
