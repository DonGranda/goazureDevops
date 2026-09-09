# myapp

A minimal Go HTTP server (stdlib only, no dependencies), built and shipped to
Azure Container Registry via GitHub Actions using OIDC. No Azure client
secret is required.

## Endpoints

| Method | Path       | Response                          |
| ------ | ---------- | ---------------------------------- |
| GET    | `/`        | `{"message": "hello from myapp"}` |
| GET    | `/healthz` | `{"status": "ok"}`                |
| GET    | `/readyz`  | `{"status": "ready"}`             |

These match the liveness/readiness probe paths used by the deployment's Helm
chart, so the container deploys as-is.

## Table of Contents

- [1. Run the app locally](#1-run-the-app-locally)
  - [Build a binary](#build-a-binary)
  - [Run with Docker](#run-with-docker)
- [2. CI/CD: GitHub Actions to Azure Container Registry (OIDC)](#2-cicd-github-actions-to-azure-container-registry-oidc)
  - [Configuration used in this project](#configuration-used-in-this-project)
  - [Architecture](#architecture)
  - [Step 1: Confirm the ACR exists](#step-1-confirm-the-acr-exists)
  - [Step 2: Create the Azure AD application and service principal](#step-2-create-the-azure-ad-application-and-service-principal)
  - [Step 3: Grant the app `AcrPush` on the registry](#step-3-grant-the-app-acrpush-on-the-registry)
  - [Step 4: Create the GitHub Environments (DEV, UAT, PROD)](#step-4-create-the-github-environments-dev-uat-prod)
  - [Step 5: Create an OIDC federated credential for each environment](#step-5-create-an-oidc-federated-credential-for-each-environment)
  - [Step 6: Collect the Azure IDs](#step-6-collect-the-azure-ids)
  - [Step 7: Store secrets in each GitHub Environment](#step-7-store-secrets-in-each-github-environment)
  - [Step 8: Add the workflow](#step-8-add-the-workflow)
  - [Step 9: Open a pull request, or trigger it manually](#step-9-open-a-pull-request-or-trigger-it-manually)
---

## 1. Run the app locally

**Requirements:** Go 1.23+

```bash
go run ./cmd/app
```

By default it listens on port `8080`. Override with the `PORT` env var:

```bash
PORT=9090 go run ./cmd/app
```

Test it:

```bash
curl localhost:8080/
curl localhost:8080/healthz
curl localhost:8080/readyz
```

### Build a binary

```bash
go build -o bin/app ./cmd/app
./bin/app
```

### Run with Docker

```bash
docker build -t myapp:local .
docker run -p 8080:8080 myapp:local
```

---

## 2. CI/CD: GitHub Actions to Azure Container Registry (OIDC)

Once the app runs locally, the rest of this README sets up the pipeline that
builds the Docker image on every pull request and pushes it to Azure
Container Registry (ACR). It authenticates using GitHub's OIDC token, so no
Azure password or client secret ever needs to be stored in GitHub.

### Configuration used in this project

This project uses three GitHub Environments named `DEV`, `UAT`, and `PROD`.
Each one has its own federated credential and its own copy of the Azure
secrets, so a deploy to one environment can never authenticate as another.

| Component           | Value                                                     |
| -------------------- | ---------------------------------------------------------- |
| GitHub Organization | `<GITHUB_ORG>`                                            |
| GitHub Repository   | `<GITHUB_REPO>`                                           |
| GitHub Environments | `DEV`, `UAT`, `PROD`                                      |
| ACR Name            | `<ACR_NAME>`                                              |
| ACR Login Server    | `<ACR_NAME>.azurecr.io`                                   |
| Azure Role          | `AcrPush`                                                 |
| OIDC Issuer         | `https://token.actions.githubusercontent.com`             |
| OIDC Subject        | `repo:<GITHUB_ORG>/<GITHUB_REPO>:environment:<DEV\|UAT\|PROD>` |

Replace `<GITHUB_ORG>`, `<GITHUB_REPO>`, and `<ACR_NAME>` with your actual
values everywhere they appear below. You can reuse the same Azure AD
application and service principal across all three environments. It just
needs one federated credential and one role assignment per environment
scope, so swap `<ACR_NAME>` per environment if each one has its own
registry.

### Architecture

```text
TO BE COMPLETED  
```

### Step 1: Confirm the ACR exists

```bash
az acr show \
  --name <ACR_NAME> \
  --query "{name:name,loginServer:loginServer,id:id}" \
  -o table
```

### Step 2: Create the Azure AD application and service principal

```bash
APP_NAME="github-actions-acr"
GITHUB_ORG="<GITHUB_ORG>"
GITHUB_REPO="<GITHUB_REPO>"
ACR_NAME="<ACR_NAME>"

APP_ID=$(az ad app create \
  --display-name "$APP_NAME" \
  --query appId -o tsv)

az ad sp create --id "$APP_ID"

echo "$APP_ID"
```

### Step 3: Grant the app `AcrPush` on the registry

```bash
ACR_ID=$(az acr show \
  --name "$ACR_NAME" \
  --query id -o tsv)

az role assignment create \
  --assignee "$APP_ID" \
  --role AcrPush \
  --scope "$ACR_ID"

# verify
az role assignment list \
  --assignee "$APP_ID" \
  --scope "$ACR_ID" \
  -o table
```

## Step 4: Create the GitHub Environments (DEV, UAT, PROD)

Environments need to exist before you can attach envirunment secrets to them or reference them in a federated credential, so create all three first.

1. Go to your repo on github.com, then **Settings**, then **Environments** (in the left sidebar, under "Code and automation").
2. Click **New enviornment**.
3. Type the name `DEV`, then click **Configure environment**.
4. Optionally, for `UAT` and `PROD`, add some protections:
   * **Required reveiwers** so someone has to approve before the job runs
   * **Deployment branches and tags** to restrict which branches can deploy to this environment (for exampel, only letting `main` deploy to `PROD`)
   * **Wait timer** to add a delay before the job is allowed to start
5. Click **Save protection rules**.
6. Repeat steps 2 throuh 5 for `UAT` and `PROD`.

Once youre done, **Settings** -> **Environments** should list all three: `DEV`, `UAT`, `PROD`. Each one manages its own secrets and protection rules on its own, so a secret added to `DEV` wont be visible to `UAT` or `PROD`.


### Step 5: Create an OIDC federated credential for each environment

Each GitHub Environment needs its own federated credential, since the
credential's `subject` encodes the environment name. Run these one at a
time for `DEV`, `UAT`, and `PROD`:

```bash
GITHUB_ORG="<GITHUB_ORG>"
GITHUB_REPO="<GITHUB_REPO>"

# DEV
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-actions-DEV",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG"'/'"$GITHUB_REPO"':environment:DEV",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# UAT
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-actions-UAT",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG"'/'"$GITHUB_REPO"':environment:UAT",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# PROD
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-actions-PROD",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:'"$GITHUB_ORG"'/'"$GITHUB_REPO"':environment:PROD",
    "audiences": ["api://AzureADTokenExchange"]
  }'

# verify, you should see 3 credentials, one per environment
az ad app federated-credential list --id "$APP_ID" -o table
```

Each subject needs to read exactly:

```text
repo:<GITHUB_ORG>/<GITHUB_REPO>:environment:DEV
repo:<GITHUB_ORG>/<GITHUB_REPO>:environment:UAT
repo:<GITHUB_ORG>/<GITHUB_REPO>:environment:PROD
```

**Portal equivalent:** go to App registrations, then your app, then
**Certificates & secrets**, then the **Federated credentials** tab, then
**Add credential**. Do this three times, once per environment, each with
entity type *Environment* and the matching environment name.

This one-per-environment approach is the best strategy here because Azure
matches federated credentials on an exact subject string, so a single
credential can never satisfy both `DEV` and `PROD` at once. Splitting them
also keeps the trust boundaries separate: if the `DEV` credential were ever
misused, it still cannot authenticate as `UAT` or `PROD`.

### Step 6: Collect the Azure IDs

```bash
TENANT_ID=$(az account show --query tenantId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)

echo "$APP_ID"
echo "$TENANT_ID"
echo "$SUBSCRIPTION_ID"
```

### Step 7: Store secrets in each GitHub Environment

Add the same three secrets to `DEV`, `UAT`, and `PROD`. If each
environment uses a different app registration or ACR, swap in the right
values for that environment's block. The example below assumes one shared
app registration across all three:

```bash
# DEV
gh secret set AZURE_CLIENT_ID \
  --body "$APP_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env DEV

gh secret set AZURE_TENANT_ID \
  --body "$TENANT_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env DEV

gh secret set AZURE_SUBSCRIPTION_ID \
  --body "$SUBSCRIPTION_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env DEV

# UAT
gh secret set AZURE_CLIENT_ID \
  --body "$APP_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env UAT

gh secret set AZURE_TENANT_ID \
  --body "$TENANT_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env UAT

gh secret set AZURE_SUBSCRIPTION_ID \
  --body "$SUBSCRIPTION_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env UAT

# PROD
gh secret set AZURE_CLIENT_ID \
  --body "$APP_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env PROD

gh secret set AZURE_TENANT_ID \
  --body "$TENANT_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env PROD

gh secret set AZURE_SUBSCRIPTION_ID \
  --body "$SUBSCRIPTION_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env PROD

# verify each environment
gh secret list --repo "$GITHUB_ORG/$GITHUB_REPO" --env DEV
gh secret list --repo "$GITHUB_ORG/$GITHUB_REPO" --env UAT
gh secret list --repo "$GITHUB_ORG/$GITHUB_REPO" --env PROD
```

You should see `AZURE_CLIENT_ID`, `AZURE_TENANT_ID`, and
`AZURE_SUBSCRIPTION_ID` listed under each environment.

**Portal equivalent:** go to the repo, then **Settings**, then
**Environments**, then select `DEV` (and later `UAT`, then `PROD`), then
**Add environment secret** for all three, once per environment.

### Step 8: Add the workflow

Create `.github/workflows/build-and-push.yml`. Pull requests always build
against `DEV`. Deploying to `UAT` or `PROD` is done manually, through
**Actions, Push to ACR, Run workflow**, picking the environment from the
dropdown:

A few things this version does differently from earlier drafts:

- **`vars.ACR_NAME`, `vars.ACR_LOGIN_SERVER`, `vars.IMAGE_NAME`** are
  [GitHub Environment variables](https://docs.github.com/actions/learn-github-actions/variables)
  rather than secrets, since none of them are sensitive. Add them under
  **Settings, Environments, `DEV`/`UAT`/`PROD`, Environment variables**,
  one set per environment if each has its own registry or image name, or
  the same values across all three if they share one.
- **Pinned action versions.** `actions/checkout@v7` and `azure/login` are
  pinned to a commit SHA rather than a tag, so a moving tag can't quietly
  get repointed to different code later.
- **No branch to environment mapping.** Every pull request builds against
  `DEV` no matter which branch it targets. `UAT` and `PROD` builds only
  happen when someone deliberately runs the workflow through
  `workflow_dispatch` and picks that environment.

### Step 9: Open a pull request, or trigger it manually

```bash
git add .
git commit -m "Add CI/CD workflow"
git push origin my-feature-branch

# open a PR against any branch, it always builds and pushes against DEV
```

To deploy to `UAT` or `PROD`, trigger the workflow manually instead. Go to
**Actions, Push to ACR, Run workflow**, choose the environment from the
dropdown, and run it.

Every run tags the image with just the run number (the environment isn't
baked into the tag in this version), for example:

```text
<ACR_LOGIN_SERVER>/<IMAGE_NAME>:1
<ACR_LOGIN_SERVER>/<IMAGE_NAME>:2
```

### Step 10: Verify the pushed image

```bash
az acr repository list --name <ACR_NAME> -o table

az acr repository show-tags \
  --name <ACR_NAME> \
  --repository <IMAGE_NAME> \
  -o table
```
