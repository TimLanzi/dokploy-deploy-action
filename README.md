# Dokploy Deploy Action

Deploy your applications on Dokploy with git commit information via Dokploy's webhook urls.
The action collects metadata from the GitHub runner (ref, sha, repo, actor, and a commit message) and POSTs it to your Dokploy webhook.

## What it sends

```json
{
  "ref": "<GITHUB_REF>",
  "head_commit": {
    "id": "<GITHUB_SHA>",
    "message": "<commit message>"
  },
  "repository": { "full_name": "<owner/repo>" },
  "pusher": { "name": "<actor>" }
}
```

> The HTTP header `X-GitHub-Event` is included and reflects the current workflow event (e.g., `push`, `workflow_dispatch`, `pull_request`, …).

## Inputs

| Name           | Required | Default | Description                                                                                                                         |
|----------------|----------|---------|-------------------------------------------------------------------------------------------------------------------------------------|
| webhook_url    | Yes      | —       | Dokploy webhook URL. Store it as a secret in caller repos.                                                                          |
| commit_message | No       | ""      | Optional override. If not provided and the repo is checked out, the action attempts `git log -1 --pretty=%B`. Empty string otherwise. |

## Outputs

| Name    | Description                                                                                              |
|---------|----------------------------------------------------------------------------------------------------------|
| payload | The exact JSON body posted to the webhook (string). Useful for debugging or logging in downstream steps. |


## Usage

### Minimal (push-only)

```yaml
name: Deploy (push)
on:
  push:
    branches: [ main ]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Dokploy Deployment
        uses: your-org/dokploy-deploy-action@v1
        with:
          webhook_url: ${{ secrets.DOKPLOY_WEBHOOK_URL }}
```

### Explicit commit message (recommended when you know the event)

```yaml
- name: Dokploy Deployment
  uses: TimLanzi/dokploy-deploy-action@v1
  with:
    webhook_url: ${{ secrets.DOKPLOY_WEBHOOK_URL }}
    commit_message: ${{ github.event.head_commit.message }} # for push
```


### Pull requests:

```yaml
commit_message: ${{ github.event.pull_request.title }}
```


### Manual runs:

```yaml
commit_message: "Manual deploy of ${{ github.sha }}"
```

### Capture the posted payload (for logs)

```yaml
- name: Dokploy Deployment
  id: dokploy
  uses: TimLanzi/dokploy-deploy-action@v1
  with:
    webhook_url: ${{ secrets.DOKPLOY_WEBHOOK_URL }}

- name: Show payload
  run: echo '${{ steps.dokploy.outputs.payload }}'
```

## Requirements & Environment

- **Secrets**: Store the webhook URL as a repository/organization secret (e.g., DOKPLOY_WEBHOOK_URL).

- **Checkout**: If you want the action to auto-detect the commit message, include actions/checkout@v4. Otherwise, pass commit_message.

- **jq**: The action checks for jq and attempts to install it on common runners (Ubuntu/macOS) and Alpine containers. If you prefer, you can remove that and install jq in the caller workflow instead.

## Behavior by Event Type

`push`: Works out-of-the-box; you can use `${{ github.event.head_commit.message }}` if desired.

`pull_request`: No `head_commit` in payload. Provide commit_message (e.g., PR title).

`workflow_dispatch` / `schedule` / others: Provide `commit_message`, or the action will try `git log -1` if checkout is present.

## Failure modes & troubleshooting

**HTTP 4xx/5xx**: The step fails the job (uses `--fail-with-body` / `--fail`). Check the step logs for the server’s response body.

**No commit message**: Ensure `actions/checkout` ran, or pass `commit_message`.

**jq not found**: Either rely on the action’s installer (default) or install `jq` in the caller workflow:

```yaml
- run: sudo apt-get update -y && sudo apt-get install -y jq
```

