# NetSuite MCP Azure Operations Runbook

This runbook documents how we run LibreChat with the NetSuite MCP connector in Azure Container Apps.

It covers the local patch we carry, why it is needed, how to maintain the company fork, how to build and publish our patched image to GitHub Container Registry, how to deploy it manually through the Azure Portal, and how to verify or roll back the deployment.

## Overview

This repository carries a small LibreChat MCP compatibility patch for NetSuite.

The Azure deployment uses:

- An existing Azure Container App.
- Existing Container App environment variables.
- A storage-mounted `librechat.yaml`.
- A private GitHub Container Registry image.
- No Azure Container Registry.

The deployment flow is intentionally conservative:

- Keep the Azure environment variables as they are.
- Keep the existing storage-mounted YAML as it is.
- Change only the container image when deploying a patched LibreChat build.
- Use pinned image tags. Never deploy `latest`.

## Why This Patch Exists

NetSuite MCP uses the Streamable HTTP transport.

LibreChat uses the MCP TypeScript SDK internally. For Streamable HTTP, the SDK sends normal JSON-RPC messages via HTTP POST. It also attempts to open an optional GET SSE stream with:

```text
Accept: text/event-stream
```

NetSuite returns:

```text
HTTP 406 Not Acceptable
```

for that optional GET SSE stream.

The important detail is that POST request/response communication still works. `tools/list` returns the NetSuite tools correctly. The problem was that LibreChat treated the optional SSE stream failure as a fatal transport error and entered reconnect/circuit-breaker loops.

Our patch does two things:

- Logs detailed, redacted MCP HTTP failures for debugging.
- Treats `Failed to open SSE stream` with `400`, `404`, `405`, or `406` and no active MCP session as non-fatal, allowing request/response transport to continue.

Expected harmless warning after the patch:

```text
SSE stream not available (406), no session. Continuing with request/response transport.
```

The patch lives in:

```text
packages/api/src/mcp/connection.ts
```

## Repository Setup

We keep a company fork and track the official LibreChat repository as upstream.

Official upstream:

```text
https://github.com/danny-avila/LibreChat
```

Company fork:

```text
https://github.com/RSM-Technology-CZ/LibreChat
```

Expected git remotes:

```bash
git remote -v
```

```text
origin    https://github.com/RSM-Technology-CZ/LibreChat.git
upstream  https://github.com/danny-avila/LibreChat.git
```

If setting this up from a fresh clone of the official LibreChat repository:

```bash
git remote rename origin upstream
git remote add origin https://github.com/RSM-Technology-CZ/LibreChat.git
```

`Dockerfile.local` is only for local development. It is not required for the Azure production image.

## LibreChat Updates

When the official LibreChat maintainers release updates, merge them into the company fork deliberately.

```bash
git checkout main
git fetch upstream
git pull origin main
git merge upstream/main
```

If there is a conflict in:

```text
packages/api/src/mcp/connection.ts
```

preserve the NetSuite compatibility behavior:

- Detailed redacted MCP HTTP logging.
- Recognition of `Not Acceptable` as HTTP `406`.
- Non-fatal handling of optional SSE open failures without an active session.
- Continued request/response operation so `tools/list` can return tools.

After resolving any conflicts:

```bash
git add .
git commit -m "chore: merge LibreChat upstream update"
git push origin main
```

Build and deploy a new pinned image tag only after the merge has been tested.

## Build And Push Image To GHCR

We use a private GitHub Container Registry package:

```text
ghcr.io/rsm-technology-cz/librechat-company:<tag>
```

Use a pinned tag for every deploy. Example:

```text
0.8.5-netsuite.1
0.8.5-netsuite.2
0.8.6-netsuite.1
```

Do not use `latest` for Azure production.

### Login To GHCR

Create a GitHub classic personal access token with package access.

For pushing images from a local machine, use a classic PAT with:

```text
write:packages
read:packages
```

If the package is connected to a private repository, the token may also need:

```text
repo
```

Login:

```bash
docker login ghcr.io -u <github-username>
```

Use the GitHub PAT as the password.

### Build And Push

```bash
docker build -t ghcr.io/rsm-technology-cz/librechat-company:0.8.5-netsuite.1 .
docker push ghcr.io/rsm-technology-cz/librechat-company:0.8.5-netsuite.1
```

Verify the package in GitHub:

1. Open the company GitHub organization.
2. Open Packages.
3. Open `librechat-company`.
4. Verify the tag exists.

### Optional: Rebuild For linux/amd64

If Azure reports a platform or manifest mismatch, rebuild and push an amd64 image:

```bash
docker buildx build --platform linux/amd64 \
  -t ghcr.io/rsm-technology-cz/librechat-company:0.8.5-netsuite.1 \
  --push .
```

## Azure Portal Deployment

The current Azure deployment is managed manually in the Azure Portal.

The existing Container App already has:

- Environment variables.
- Secrets.
- Storage-mounted `librechat.yaml`.
- Ingress and runtime settings.

Do not change those during a normal NetSuite patch deployment. Change only the container image.

### Create A New Revision

1. Open the Azure Portal.
2. Open the existing LibreChat Container App.
3. Go to **Application**.
4. Open **Revisions and replicas**.
5. Click **Create new revision**.
6. Select the existing LibreChat container.
7. Select **Docker Hub or other registries**.
8. Select **Private**.

Private registry settings:

```text
Login server: ghcr.io
Registry user name: <github-username>
Registry password: GitHub classic PAT with read:packages
```

Important:

- The registry user name must be the GitHub user that owns the PAT.
- Do not use the GitHub organization name as the registry user name.
- A private GHCR package may require a classic PAT with both `read:packages` and `repo`.

Container image:

```text
ghcr.io/rsm-technology-cz/librechat-company:0.8.5-netsuite.1
```

Leave these unchanged:

- Environment variables.
- Secrets.
- Storage mounts.
- Mounted `librechat.yaml`.
- `CONFIG_PATH`, if it is already working.

Save the new revision.

## Required Azure Configuration

The existing Azure Container App configuration must provide the NetSuite OAuth values expected by `librechat.yaml`.

Required environment variables:

```text
NETSUITE_MCP_CLIENT_ID
NETSUITE_MCP_CLIENT_SECRET
```

The mounted `librechat.yaml` must be available at the path currently used by `CONFIG_PATH`.

The NetSuite server config should keep:

```yaml
mcpServers:
  netsuite:
    type: streamable-http
```

Do not change the NetSuite transport to `sse`. NetSuite uses Streamable HTTP, and the legacy SSE transport expects a different server behavior.

The Azure callback URI must be registered exactly in the NetSuite Integration record:

```text
https://<company-librechat-domain>/api/mcp/netsuite/oauth/callback
```

The local callback URI is different and should only be used for local development:

```text
http://localhost:3080/api/mcp/netsuite/oauth/callback
```

## Verification

After deploying the new Azure revision, open the Container App logs.

In the Azure Portal:

1. Open the Container App.
2. Go to **Monitoring**.
3. Open **Log stream** or **Logs**.

Expected success indicators:

```text
[MCP][netsuite] OAuth Required: true
[MCP Reinitialize] Response for netsuite
toolsCount: 16
```

Expected harmless warning:

```text
MCP HTTP response error ... status 406
SSE stream not available (406), no session. Continuing with request/response transport.
```

Failure indicators:

```text
toolsCount: 0
Circuit breaker is open
Stopping reconnection attempts
```

If tools are missing after deployment, trigger NetSuite MCP reinitialize from the LibreChat UI and check the logs again.

## Troubleshooting

### Azure Fails Immediately On Save Or Create Revision

If Azure fails immediately after saving the revision, the most likely issue is GHCR authentication.

Check:

- The registry login server is `ghcr.io`.
- The registry user name is a GitHub user, not the organization.
- The registry password is a GitHub classic PAT.
- The PAT has `read:packages`.
- For private packages, the PAT may also need `repo`.
- The GitHub user has access to the private GHCR package.

Test the same credentials locally:

```bash
docker logout ghcr.io
docker login ghcr.io -u <github-username>
docker pull ghcr.io/rsm-technology-cz/librechat-company:0.8.5-netsuite.1
```

If the local pull fails, Azure will fail too.

### Azure Reports Platform Or Manifest Mismatch

Rebuild and push a linux/amd64 image:

```bash
docker buildx build --platform linux/amd64 \
  -t ghcr.io/rsm-technology-cz/librechat-company:0.8.5-netsuite.1 \
  --push .
```

Then create a new Azure Container App revision with the same image tag or a new pinned tag.

### LibreChat Starts But NetSuite Tools Are Missing

Check:

- The mounted `librechat.yaml` is present.
- `CONFIG_PATH` points to the mounted YAML.
- `type` is `streamable-http`.
- `NETSUITE_MCP_CLIENT_ID` is set.
- `NETSUITE_MCP_CLIENT_SECRET` is set.
- The Azure callback URI is registered in NetSuite.
- OAuth completed for the user.
- Logs show `toolsCount: 16`.

### Reconnect Storm Or Circuit Breaker Appears

Check whether the running image is the patched company image.

The patched image should log:

```text
SSE stream not available (406), no session. Continuing with request/response transport.
```

and should continue to return tools.

If the logs show repeated reconnect attempts after the `406`, Azure may still be running an unpatched upstream LibreChat image.

## Rollback

Use Azure Container Apps revisions for rollback.

In the Azure Portal:

1. Open the Container App.
2. Open **Revisions and replicas**.
3. Find the previous healthy revision.
4. Route traffic back to the previous healthy revision.
5. Or create a new revision using the previous known-good image tag.

Keep old image tags available in GHCR so rollback stays simple.

Do not overwrite old tags. Create a new tag for every build.

## Future Improvement

Secrets are currently kept in the existing Container App configuration.

Later, migrate secrets to Azure Key Vault:

- `NETSUITE_MCP_CLIENT_ID`
- `NETSUITE_MCP_CLIENT_SECRET`
- LibreChat JWT and credential secrets
- API keys

This is a separate operations change. It is not required for the patched image deployment.

Keep the GHCR image flow unchanged unless the company later allows Azure Container Registry.

## Quick Checklist

Before deploy:

- Company fork contains the NetSuite MCP patch.
- Image is built from the patched repo.
- Image tag is pinned.
- Image is pushed to private GHCR.
- Local `docker pull` works with the same token planned for Azure.
- Azure Container App still has the existing env vars and storage-mounted YAML.

After deploy:

- New revision starts.
- LibreChat UI loads.
- NetSuite OAuth works.
- NetSuite MCP reinitialize succeeds.
- Logs show `toolsCount: 16`.
- No reconnect storm follows the expected `406` SSE warning.

