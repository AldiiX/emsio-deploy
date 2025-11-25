# Deploy to Emsio Servers (GitHub Action)

Deploy your application to Emsio servers by calling the Emsio Deploy API.  
This composite action streams server logs to the workflow, writes a step summary, and handles intermittent network hiccups gracefully.

## Features

- 🔐 Authenticates with a per-project deploy token and project UUID  
- 🌿 Optional **branch** input (defaults to the current branch) sent as `X-GHRP-Branch`  
- 📡 Streams server output directly in the job log  
- 🧾 Publishes a concise summary to the GitHub Step Summary (`$GITHUB_STEP_SUMMARY`)  
- 🛡️ Tolerates `curl` exit 56 (dropped HTTP/1.1 stream) when the server returned `200` and no error marker is present  
- 🧰 Pure Bash + cURL, no external dependencies beyond the runner

## Inputs

| Name              | Required | Description                                                                |
|-------------------|----------|----------------------------------------------------------------------------|
| `app_secret`      | ✅       | Deployment token used to authenticate requests.                            |
| `env_b64`         | ❌       | Base64 encoded .env content that will be forwarded in request body.        |
| `branch`          | ❌       | Branch name to deploy. Defaults to the current branch (`github.ref_name`). |

> **Note:** Pass secrets from your workflow via `with:` (composite actions cannot read `secrets.*` directly).

## Outputs

This action does not set formal outputs. It writes:
- a human-readable summary to the **Step Summary**,
- the full raw response (including HTTP status and `curl` exit code) to a file named **`full_response.txt`** in the workspace.

## Usage

### 1) Add repository secrets
Create two repository or organization secrets:
- `DEPLOY_TOKEN`
- `PROJECT_UUID`

### 2) Use the action in a workflow

Create a workflow file (e.g., `.github/workflows/deploy.yml`) with the following content:

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
      - name: Deploy to Emsio
        uses: AldiiX/emsio-deploy@v4
        with:
          app_secret: ${{ secrets.APP_SECRET }}
          # branch is optional; if it is not specified, the current branch is used.
          # branch: main
          env_b64: ${{ secrets.ENV_B64 }}
```

> If you set `branch`, it will be sent as the `X-GHRP-Branch` header. If omitted, the action uses the current branch.

## What the action does

- Issues a `POST` to the Emsio Deploy API endpoint.
- Streams the response to the job log and also to `full_response.txt`.
- Captures:
  - `HTTP_CODE` (e.g., `200`, `500`, …)  
  - `CURL_EXIT` (the `curl` exit code)
- Determines success/failure using:
  - last non-empty log line contains a **❌** → fail
  - non-200 HTTP status → fail
  - `curl` exit **56** with HTTP `200` and no error marker → **warn but pass**
  - otherwise → pass
- Publishes a short step summary so you can see the result at a glance in the run UI.



## Troubleshooting

- **HTTP 4xx/5xx:** The step will fail and the Step Summary shows the HTTP status. Check the job log for details.
- **`curl` exit 56:** This is treated as a transient read error on an otherwise successful stream. If the server returned `200` and the log does not include a ❌ marker, the job passes with a warning.
- **No HTTP code captured:** If the network failed before headers were received, `HTTP_CODE` may be `0`. The action will mark the step as failed.
- **Full raw response:** Inspect `full_response.txt` in the workspace to see exactly what was received. You can also upload it as an artifact:

  ```yaml
  - name: Upload deploy log
    if: always()
    uses: actions/upload-artifact@v4
    with:
      name: deploy-response
      path: full_response.txt
  ```

## Security Notes

- Always provide `deploy_token` and `project_uuid` via **secrets**.  
- The action masks the token in logs and never echoes it directly.


  
## Notes

- The action sends headers `X-GHRP-Name` (repository), `X-GHRP-Branch` (branch), and uses HTTP/1.1 streaming for near‑real‑time logs.
- If you ever see an empty branch in the headers, ensure the `branch` input is passed through (see examples above).

--- 

*Author: Stanislav Škudrna*