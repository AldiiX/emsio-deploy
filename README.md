# Deploy to Emsio Servers (GitHub Action)

Deploy your application to Emsio servers by calling the Emsio Deploy API endpoint.

This is a lightweight **composite action** that sends a `POST` request to the deploy endpoint with:
- `Authorization: Bearer <token>`
- `X-SHA: <commit sha>`

## Features

- 🔐 Auth via deploy token (`app_secret`)
- 🧩 Works as a composite action (Bash + cURL)
- 🧾 Fails the step on non-2xx / network errors (via `curl -f`)

## Inputs

| Name         | Required | Description                                  |
|--------------|----------|----------------------------------------------|
| `app_secret` | ✅       | Deployment token used to authenticate requests |

> **Note:** Pass secrets from your workflow via `with:` (composite actions cannot read `secrets.*` directly).

## Outputs

This action does not define outputs.

## Usage

### 1) Add repository secrets

Create a repository (or organization) secret:
- `APP_SECRET`
- `PROJECT_UUID`
- `REGISTRY_USER`
- `REGISTRY_PASSWORD`

> Your build/push secrets (registry user/password etc.) belong to your workflow, not this action.

### 2) Use the action in a workflow

Create a workflow file (e.g. `.github/workflows/deploy.yml`) like this:

```yaml
name: Deploy to Emsio Servers

on:
  push:
    branches: [ main ]
  workflow_dispatch:

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: docker/login-action@v3
        with:
          registry: registry.emsio.cz
          username: robot$${{ secrets.REGISTRY_USER }}+ci
          password: ${{ secrets.REGISTRY_PASSWORD }}

      - uses: docker/build-push-action@v6
        env:
          DOCKER_BUILD_RECORD_UPLOAD: false
        with:
          context: .
          push: true
          tags: |
            registry.emsio.cz/${{ secrets.REGISTRY_USER }}/${{ secrets.PROJECT_UUID }}:latest

      - name: Deploy to Emsio
        uses: AldiiX/emsio-deploy@v4
        with:
          app_secret: ${{ secrets.APP_SECRET }}
```

## What the action does

- Sends `POST` request to the deploy endpoint:
  - `Authorization: Bearer <app_secret>`
  - `X-SHA: <current commit sha>`
- The step succeeds when the endpoint returns a successful HTTP status code.
- The step fails when the endpoint returns `4xx/5xx` or the request fails on the network.

## Troubleshooting

- **HTTP 4xx/5xx:** The step fails. Check the job log to see the HTTP error.
- **Network / DNS / TLS issues:** `curl` fails and the step fails. Re-run the workflow and/or check connectivity to the deploy endpoint.

## Security Notes

- Always pass the token via GitHub **Secrets** (`with: app_secret: ${{ secrets.APP_SECRET }}`).
- Do not print the token in workflow logs.

---

*Author: Stanislav Škudrna*
