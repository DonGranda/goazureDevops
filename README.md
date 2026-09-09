<!-- <!-- # myapp

A minimal Go HTTP server (stdlib only, no dependencies) with:

- `GET /`        → `{"message": "hello from myapp"}`
- `GET /healthz` → `{"status": "ok"}`
- `GET /readyz`  → `{"status": "ready"}`

These match the liveness/readiness probe paths in the Helm chart from the
CI/CD pipeline, so this app deploys as-is.

# -->
# GitHub Actions → Azure Container Registry using OIDC

This project uses **GitHub Actions OIDC** to securely build and push Docker images to Azure Container Registry (ACR).

No Azure client secret is required.

## 1. Set Variables

```bash
APP_NAME="github-actions-acr"
GITHUB_ORG="<YOUR_GITHUB_ORG>"
GITHUB_REPO="<YOUR_GITHUB_REPO>"
ACR_NAME="<YOUR_ACR_NAME>"
GITHUB_ENVIRONMENT="<YOUR_GITHUB_ENVIRONMENT>"
```

## 2. Create Azure Application

```bash
APP_ID=$(az ad app create \
  --display-name "$APP_NAME" \
  --query appId -o tsv)

az ad sp create --id "$APP_ID"
```

## 3. Give Access to ACR

```bash
ACR_ID=$(az acr show \
  --name "$ACR_NAME" \
  --query id -o tsv)

az role assignment create \
  --assignee "$APP_ID" \
  --role AcrPush \
  --scope "$ACR_ID"
```

## 4. Create OIDC Credential

```bash
az ad app federated-credential create \
  --id "$APP_ID" \
  --parameters '{
    "name": "github-actions-environment",
    "issuer": "https://token.actions.githubusercontent.com",
    "subject": "repo:<YOUR_GITHUB_ORG>/<YOUR_GITHUB_REPO>:environment:<YOUR_GITHUB_ENVIRONMENT>",
    "audiences": ["api://AzureADTokenExchange"]
  }'
```

## 5. Get Azure IDs

```bash
TENANT_ID=$(az account show --query tenantId -o tsv)
SUBSCRIPTION_ID=$(az account show --query id -o tsv)
```

## 6. Add GitHub Environment Secrets

```bash
gh secret set AZURE_CLIENT_ID \
  --body "$APP_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env "$GITHUB_ENVIRONMENT"

gh secret set AZURE_TENANT_ID \
  --body "$TENANT_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env "$GITHUB_ENVIRONMENT"

gh secret set AZURE_SUBSCRIPTION_ID \
  --body "$SUBSCRIPTION_ID" \
  --repo "$GITHUB_ORG/$GITHUB_REPO" \
  --env "$GITHUB_ENVIRONMENT"
```

## 7. GitHub Actions Workflow

Create:

```text
.github/workflows/build-and-push.yml
```

```yaml
name: Build and Push to ACR

on:
  push:
    branches:
      - main

permissions:
  id-token: write
  contents: read

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    environment: <YOUR_GITHUB_ENVIRONMENT>

    steps:
      - uses: actions/checkout@v4

      - name: Login to Azure
        uses: azure/login@v2
        with:
          client-id: ${{ secrets.AZURE_CLIENT_ID }}
          tenant-id: ${{ secrets.AZURE_TENANT_ID }}
          subscription-id: ${{ secrets.AZURE_SUBSCRIPTION_ID }}

      - name: Login to ACR
        run: az acr login --name <YOUR_ACR_NAME>

      - name: Build and Push
        run: |
          docker build \
            -t <YOUR_ACR_LOGIN_SERVER>/<YOUR_IMAGE_NAME>:${{ github.run_number }} .

          docker push \
            <YOUR_ACR_LOGIN_SERVER>/<YOUR_IMAGE_NAME>:${{ github.run_number }}
```

## Result

GitHub Actions authenticates to Azure using **OIDC**, logs into ACR, builds the Docker image, and pushes it to your registry.

No Azure client secret or password is stored in GitHub.
