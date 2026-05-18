# Publishing the OpenAPI Spec to Azure API Management

This guide describes how the OpenAPI spec for this repository is published to **Azure API Management (APIM)**, where it is rendered in the APIM **Developer Portal** for consumers.

Publishing is fully automated by GitHub Actions:

| Trigger                                | Workflow                                                                | Behaviour                                                                  |
|----------------------------------------|-------------------------------------------------------------------------|----------------------------------------------------------------------------|
| Push / merge to `main`                 | [`ci-draft.yml`](../.github/workflows/ci-draft.yml) → `Push-Draft-OpenAPI-Spec-Azure-APIM` | Imports the spec into the **current revision** of the API in APIM (overwrites in place). |
| GitHub release published               | [`ci-released.yml`](../.github/workflows/ci-released.yml) → `Push-Release-OpenAPI-Spec-Azure-APIM` | Imports the spec into a **new revision**, then promotes it to current via `az apim api release create`. |

The shared logic lives in the reusable workflow [`publish-openapi-azure-apim.yml`](../.github/workflows/publish-openapi-azure-apim.yml).

---

## Publish flow at a glance

```
PR merged to main
        │
        ▼
ci-draft.yml ──► Artefact-Version ──► Update-Spec-Version ──► Test
                                                              │
                                                              ▼
                                              Push-Draft-OpenAPI-Spec-Azure-APIM
                                                              │
                                                              ▼
                                          publish-openapi-azure-apim.yml
                                              │
                                              ├─ download versioned spec artifact
                                              ├─ azure/login (service principal)
                                              ├─ az apim api import
                                              └─ write run summary
                                                              │
                                                              ▼
                                              Azure APIM Developer Portal
```

The release flow is identical except that the import targets a fresh revision and a subsequent `az apim api release create` promotes that revision to current.

---

## API model in APIM

The workflow creates and maintains one APIM object — a single API per repo:

| Object        | ID / naming                          | Notes                                                                              |
|---------------|--------------------------------------|------------------------------------------------------------------------------------|
| API           | `<repo-name>` (e.g. `api-cp-crime-echo`) | One API per repo. No APIM-level version set / version routing.                  |
| Revisions     | `1`, `2`, …                          | Drafts overwrite revision 1. Releases create a new revision and promote it.        |
| URL path      | value of `AZURE_APIM_API_PATH`       | The path consumers hit on the gateway (e.g. `cp/crime/echo`).                      |

APIM acts as a transparent gateway. Version negotiation is handled by the service via the `Accept` media-type (e.g. `Accept: application/vnd.hmcts.cp.v1+json`). See [`docs/API-VERSIONING-STRATEGY.md`](./API-VERSIONING-STRATEGY.md) for the application-side rules. If APIM-level version routing is ever required (e.g. v1 and v2 served from different backends), introduce a version set then.

---

## Roles and responsibilities

Setting up and operating this pipeline is shared between two roles. The split is the same whether one person plays both or they're separate teams.

| Role          | Owns                                                                                                              | Touches                                          |
|---------------|-------------------------------------------------------------------------------------------------------------------|--------------------------------------------------|
| **DevOps**    | Anything in Azure: the APIM instance, the service principal that publishes into it, the RBAC role assignment.     | Azure portal / `az` CLI.                         |
| **Developer** | Anything in the GitHub repo: workflow files, OpenAPI spec, repo secrets/variables, API naming decisions.          | GitHub repo settings, files in `src/`/`.github/`.|

The two roles meet at one **handoff** (after [step 4](#4-devops-create-the-service-principal)): DevOps hands the developer a short list of credentials and identifiers, and the developer plugs them into the repo. After that handoff, day-to-day publishing is fully automated.

### Quick task list

| Step                                                          | Role      | Output                                                                  |
|---------------------------------------------------------------|-----------|-------------------------------------------------------------------------|
| [1. Sign in & locate the APIM instance](#1-devops-sign-in-and-locate-the-apim-instance) | DevOps    | APIM name, resource group, subscription resource path                   |
| [2. Switch subscription context](#2-devops-switch-context-to-the-apims-subscription)    | DevOps    | `AZURE_SUBSCRIPTION_ID`, `AZURE_TENANT_ID`                              |
| [3. Confirm APIM path is free](#3-developer-confirm-the-apim-path-is-free)               | Developer | Chosen `AZURE_APIM_API_PATH` is unique in the APIM                      |
| [4. Create the service principal](#4-devops-create-the-service-principal)                | DevOps    | `AZURE_CLIENT_ID`, `AZURE_CLIENT_SECRET`                                |
| [Handoff](#handoff-devops--developer)                                                    | both      | Secure transfer of six values                                           |
| [5. Add values to GitHub](#5-developer-add-the-values-to-github)                         | Developer | Four secrets + three variables configured on the repo                   |
| [6. Trigger the workflow](#6-developer-trigger-the-workflow)                             | Developer | First successful publish to the APIM Developer Portal                   |
| [7. Make the API visible in the Developer Portal](#7-devops-make-the-api-visible-in-the-developer-portal) | DevOps    | API listed in the dev portal catalog                                    |
| [8. Enable Try-It mock responses](#8-devops-enable-try-it-mock-responses)                 | DevOps    | Consumers can invoke operations from the dev portal without a backend  |

---

## One-time setup

This walkthrough produces every value listed in [Required values](#required-values). Prerequisite for DevOps steps: [Azure CLI](https://learn.microsoft.com/cli/azure/install-azure-cli) installed and you can sign in to Azure with permissions to create app registrations in the target tenant and to assign roles on the target APIM resource.

### 1. [DevOps] Sign in and locate the APIM instance

```bash
# Sign in. Opens a browser unless you already have a session.
az login

# Find the APIM instance you want to publish into. Replace <APIM_NAME>
# with the name you see in the Azure portal or in the dev portal URL
# (e.g. https://<APIM_NAME>.developer.azure-api.net/).
az apim list \
  --query "[?name=='<APIM_NAME>'].{name:name, rg:resourceGroup, sub:id}" \
  -o table
```

Output gives you two values straight away:

| Column | Maps to                       | Notes                                                                                                                |
|--------|-------------------------------|----------------------------------------------------------------------------------------------------------------------|
| `name` | `AZURE_APIM_SERVICE_NAME`     | Same as the value you passed in.                                                                                     |
| `rg`   | `AZURE_APIM_RESOURCE_GROUP`   | Resource group hosting the APIM instance.                                                                            |
| `sub`  | (parse for subscription)      | The full resource path. The subscription ID is the GUID between `/subscriptions/` and `/resourceGroups/`.            |

### 2. [DevOps] Switch context to the APIM's subscription

Your active subscription is **not** necessarily the one hosting the APIM. Switch to it explicitly so subsequent commands act in the right place:

```bash
# Subscription ID extracted from step 1's `sub` column.
az account set --subscription <APIM_SUBSCRIPTION_ID>

# Confirm subscription + tenant context.
az account show --query "{subscriptionId:id, tenantId:tenantId}" -o table
```

This output gives you two more values:

| Column           | Maps to                  |
|------------------|--------------------------|
| `subscriptionId` | `AZURE_SUBSCRIPTION_ID`  |
| `tenantId`       | `AZURE_TENANT_ID`        |

### 3. [Developer] Confirm the APIM path is free

`AZURE_APIM_API_PATH` is a naming decision **owned by the developer** (suggested convention: `<source-system>/<case-type>/<entity>`, e.g. `cp/crime/echo`). It must be unique within the APIM instance — APIM will reject an import that collides with an existing API's path. DevOps can run this check on the developer's behalf if the developer doesn't have read access on the APIM.

```bash
az apim api list \
  --resource-group <AZURE_APIM_RESOURCE_GROUP> \
  --service-name  <AZURE_APIM_SERVICE_NAME> \
  --query "[].{name:name, displayName:displayName, path:path}" \
  -o table
```

If your intended path appears, pick a different one or delete the colliding API (see [Common failures](#common-failures-and-fixes)).

### 4. [DevOps] Create the service principal

```bash
az ad sp create-for-rbac \
  --name "gha-api-cp-crime-echo-apim-publisher" \
  --role "API Management Service Contributor" \
  --scopes "/subscriptions/<AZURE_SUBSCRIPTION_ID>/resourceGroups/<AZURE_APIM_RESOURCE_GROUP>/providers/Microsoft.ApiManagement/service/<AZURE_APIM_SERVICE_NAME>"
```

Output:

```json
{
  "appId":       "…",  // → AZURE_CLIENT_ID
  "displayName": "…",
  "password":    "…",  // → AZURE_CLIENT_SECRET   (shown once only)
  "tenant":      "…"   // → AZURE_TENANT_ID       (confirms step 2)
}
```

> **Secret hygiene.** `password` is printed exactly once. Copy it straight into GitHub repository secrets — don't paste it into chats, tickets, or anywhere else. If it leaks, rotate immediately:
> ```bash
> az ad sp credential reset --id <AZURE_CLIENT_ID> --display-name "gha-rotated"
> ```

Why `API Management Service Contributor`? It is the least-privilege built-in role that can import APIs and manage revisions / releases. Scoping it to the single APIM resource (not the whole subscription) limits blast radius if the credential leaks.

### Handoff: DevOps → Developer

After completing steps 1, 2, and 4, DevOps hands over **six values** to the developer. Use a secure channel (1Password share, sealed envelope vault, etc.) — not chat, email, or tickets:

| Value to hand over           | Comes from                                             | Developer puts it in                                                 |
|------------------------------|--------------------------------------------------------|----------------------------------------------------------------------|
| Subscription ID              | [Step 2](#2-devops-switch-context-to-the-apims-subscription) | Secret `AZURE_SUBSCRIPTION_ID`                                       |
| Tenant ID                    | [Step 2](#2-devops-switch-context-to-the-apims-subscription) | Secret `AZURE_TENANT_ID`                                             |
| Client ID (`appId`)          | [Step 4](#4-devops-create-the-service-principal)             | Secret `AZURE_CLIENT_ID`                                             |
| Client Secret (`password`)   | [Step 4](#4-devops-create-the-service-principal) — shown once | Secret `AZURE_CLIENT_SECRET`                                         |
| APIM service name            | [Step 1](#1-devops-sign-in-and-locate-the-apim-instance)     | Variable `AZURE_APIM_SERVICE_NAME`                                   |
| APIM resource group          | [Step 1](#1-devops-sign-in-and-locate-the-apim-instance)     | Variable `AZURE_APIM_RESOURCE_GROUP`                                 |

⚠️ **Common gotcha.** The example values in the [Required values](#required-values) tables (`apim-hmcts-nonprod`, `rg-hmcts-apim-nonprod`) are *placeholders*. The developer must overwrite them with the real handed-over values, or the workflow will fail with `AuthorizationFailed` (the SP only has rights on the real APIM, not the placeholder name).

### 5. [Developer] Add the values to GitHub

In the GitHub repo: **Settings → Secrets and variables → Actions**.

- **Secrets** tab — add the four `AZURE_*` secrets listed under [Required values](#required-values).
- **Variables** tab — add the three `AZURE_APIM_*` variables.

### 6. [Developer] Trigger the workflow

Merge a change to `main`. The `Push-Draft-OpenAPI-Spec-Azure-APIM` job should appear in the run and `az apim api show` will return the API within a minute.

### 7. [DevOps] Make the API visible in the Developer Portal

`az apim api import` adds the API to APIM but does **not** associate it with any Product. The Developer Portal's catalog only lists APIs that belong to at least one published Product, so without this step the API is invisible to consumers — even though it exists in APIM and the spec is rendered correctly when opened by direct URL.

This is a one-time per-APIM-instance step. Pick the product(s) that match your access model; the default APIM instance ships with two published products: `starter` and `unlimited`.

```bash
# Replace <PRODUCT_ID> with the product the API should appear under
# (e.g. starter, unlimited, or a bespoke one).
az apim product api add \
  --resource-group <AZURE_APIM_RESOURCE_GROUP> \
  --service-name  <AZURE_APIM_SERVICE_NAME> \
  --product-id    <PRODUCT_ID> \
  --api-id        <repo-name>
```

The call is idempotent — running it again when the API is already in the product is a no-op.

Verify:

```bash
az apim product api list \
  --resource-group <AZURE_APIM_RESOURCE_GROUP> \
  --service-name  <AZURE_APIM_SERVICE_NAME> \
  --product-id    <PRODUCT_ID> \
  --query "[].name" -o tsv
```

> **Why this isn't in the workflow.** Product choice is a deliberate per-API decision (access tier, rate limits, terms of use) and shouldn't be hard-coded into a reusable workflow. Picking and running this once per APIM instance keeps the choice explicit.

Version bumps happen at the spec layer (`Accept` header), not in APIM — the API object in APIM is reused across major versions, so step 7 does not need to be re-run when bumping `v1 → v2` at the application layer.

### 8. [DevOps] Enable Try-It mock responses

The dev portal's **Try It** button issues a real HTTP request to the API's backend URL. There is no backend deployed for this API yet, so without a mock, every Try-It call fails. The fastest fix is APIM's built-in `mock-response` policy, which makes APIM itself return the `example` payloads from the OpenAPI spec — no external mock server, no Spring Boot deployment needed.

This is a one-time per-API per-APIM-instance step. It uses the same `example:` blocks already in the spec (the ones we maintain for Redocly).

The Azure CLI does not expose policy management directly, so this goes through `az rest`:

```bash
BODY=$(mktemp); cat > "$BODY" <<'EOF'
{
  "properties": {
    "format": "xml",
    "value": "<policies><inbound><base /><mock-response status-code=\"200\" /></inbound><backend><base /></backend><outbound><base /></outbound><on-error><base /></on-error></policies>"
  }
}
EOF
az rest --method put \
  --uri "https://management.azure.com/subscriptions/<AZURE_SUBSCRIPTION_ID>/resourceGroups/<AZURE_APIM_RESOURCE_GROUP>/providers/Microsoft.ApiManagement/service/<AZURE_APIM_SERVICE_NAME>/apis/<repo-name>/policies/policy?api-version=2022-08-01" \
  --headers "Content-Type=application/json" \
  --body "@$BODY"; rm -f "$BODY"
```

Verify:

```bash
az rest --method get \
  --uri "https://management.azure.com/subscriptions/<AZURE_SUBSCRIPTION_ID>/resourceGroups/<AZURE_APIM_RESOURCE_GROUP>/providers/Microsoft.ApiManagement/service/<AZURE_APIM_SERVICE_NAME>/apis/<repo-name>/policies/policy?api-version=2022-08-01"
```

Open the dev portal, click any operation, click **Try It**, **Send**. Response body matches the spec's example.

> **When to remove the policy.** Once a real backend is deployed and the API's `serviceUrl` points at it, drop the `<mock-response />` line so requests actually hit the backend. The other tags can stay as-is — they're harmless `<base />` placeholders.

Step 8 does not need to be re-run on a spec-level major version bump — the policy is attached to the API object, which is reused.

---

## Ongoing responsibilities

After the one-time setup, the pipeline runs unattended on every merge to `main`. The two roles still own things long-term:

### DevOps

- **Rotate the SP secret** on a schedule (suggested: yearly, or immediately if it's suspected to have leaked). Use `az ad sp credential reset --id <AZURE_CLIENT_ID> --display-name "rotation-YYYY-MM"` and hand the new password to the developer.
- **Maintain the RBAC assignment.** Do not broaden the SP's scope beyond the single APIM resource without a documented reason.
- **Triage Azure-side failures** in CI runs — anything where the failing step is `Azure login` or `Import spec to APIM`. The error message in the job log is usually self-explanatory (`AuthorizationFailed`, `ResourceNotFound`, etc.). See [Common failures](#common-failures-and-fixes).
- **Decommission the SP** when the repo is retired (`az ad sp delete --id <AZURE_CLIENT_ID>`).

### Developer

- **Maintain the OpenAPI spec** at `src/main/resources/openapi/openapi-spec.yml`. Linting and schema validation run on every PR.
- **Apply [`API-VERSIONING-STRATEGY.md`](./API-VERSIONING-STRATEGY.md)** for breaking-change handling. Versioning is done at the application layer via the `Accept` header; the same APIM API object serves all versions, so no workflow change is needed when bumping.
- **Triage developer-side failures** in CI runs — Spectral lint errors, JSON-schema validation, Gradle build, Java test failures.
- **Review APIM Developer Portal output** after each merge for unintended changes (e.g. an operation accidentally removed).

---

## Required values

All values below must be set in the GitHub repository before the workflow can succeed. Secrets are masked in logs; variables are not (so do **not** put sensitive data in the Variables tab).

### Secrets (`Settings → Secrets and variables → Actions → Secrets`)

| Name                    | What it is                                                             | Where to obtain                                                                                                                                                                              |
|-------------------------|------------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `AZURE_CLIENT_ID`       | Application (client) ID of the service principal that publishes.       | `appId` field from `az ad sp create-for-rbac` ([step 4](#4-create-the-service-principal)). To re-read later: `az ad sp list --display-name "gha-api-cp-crime-echo-apim-publisher" --query "[0].appId" -o tsv`. |
| `AZURE_CLIENT_SECRET`   | Client secret for the service principal.                               | `password` field from `az ad sp create-for-rbac` ([step 4](#4-create-the-service-principal)). Shown **once**. To rotate: `az ad sp credential reset --id <AZURE_CLIENT_ID> --display-name "gha-rotated"`.       |
| `AZURE_TENANT_ID`       | Microsoft Entra (Azure AD) tenant ID hosting the service principal.    | `tenantId` from `az account show` ([step 2](#2-switch-context-to-the-apims-subscription)). Or `tenant` from the `sp create-for-rbac` output ([step 4](#4-create-the-service-principal)).                       |
| `AZURE_SUBSCRIPTION_ID` | ID of the subscription that owns the APIM resource.                    | `subscriptionId` from `az account show` after `az account set` ([step 2](#2-switch-context-to-the-apims-subscription)). Or extract from the `sub` column of `az apim list` ([step 1](#1-sign-in-and-locate-the-apim-instance)). |

### Variables (`Settings → Secrets and variables → Actions → Variables`)

| Name                         | What it is                                              | Example                  | Where to obtain                                                                                                                                                                                          |
|------------------------------|---------------------------------------------------------|--------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `AZURE_APIM_RESOURCE_GROUP`  | Resource group containing the APIM instance.            | `rg-hmcts-apim-nonprod`  | `rg` column from `az apim list` ([step 1](#1-sign-in-and-locate-the-apim-instance)). To look up directly: `az apim show --name <APIM_NAME> --query resourceGroup -o tsv` *(scans all subs in current context)*. |
| `AZURE_APIM_SERVICE_NAME`    | Name of the APIM instance to publish into.              | `apim-hmcts-nonprod`     | The `<APIM_NAME>` you passed to `az apim list` ([step 1](#1-sign-in-and-locate-the-apim-instance)). Also the subdomain in the dev-portal URL `https://<name>.developer.azure-api.net/`.                       |
| `AZURE_APIM_API_PATH`        | URL path consumers hit on the APIM gateway.             | `cp/crime/echo`          | A naming decision for this repo — not derived from Azure. Convention: `<source-system>/<case-type>/<entity>`. Verify it isn't already taken with the path-check query in [step 3](#3-confirm-the-apim-path-is-free). |

### Inputs hard-coded in the workflow call

These are not user-supplied; they come from the workflow itself or from repo metadata. Listed here for completeness.

| Input                    | Source                                                  | Why fixed                                                                            |
|--------------------------|---------------------------------------------------------|--------------------------------------------------------------------------------------|
| `apim_api_id`            | `github.event.repository.name`                          | Keeps the APIM API ID aligned with the repo name automatically.                      |
| `apim_subscription_required` | `true` (workflow default)                           | Requires consumers to send `Ocp-Apim-Subscription-Key`. Override per-call if needed. |
| `upload_artifact_name`   | Output of the `Update-Spec-Version` job                 | Ensures the version-stamped spec is published, not the raw repo file.                |
| `api_version`            | Output of the `Artefact-Version` job                    | Used in release notes and `release-id` on promotion.                                 |

---

## Verifying a successful publish

After a merge to `main`, check in this order:

1. **GitHub Actions run** — the `Push-Draft-OpenAPI-Spec-Azure-APIM` job should be green. The job summary lists the APIM service, API ID, path, and spec version.
2. **Azure CLI:**
   ```bash
   az apim api show \
     --resource-group "$AZURE_APIM_RESOURCE_GROUP" \
     --service-name  "$AZURE_APIM_SERVICE_NAME" \
     --api-id        api-cp-crime-echo
   ```
3. **APIM Developer Portal** — operations and schemas reflect the latest spec. If the API doesn't appear in the catalog at all, it hasn't been added to a Product — see [step 7](#7-devops-make-the-api-visible-in-the-developer-portal).
4. **Try It in the dev portal** — clicking **Send** on any operation returns the example payload from the spec. If it returns a backend error, the mock policy isn't applied — see [step 8](#8-devops-enable-try-it-mock-responses).

---

## Common failures and fixes

| Symptom                                                                  | Likely cause                                                                                | Fix                                                                                                              |
|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `azure/login` step fails with `AADSTS7000215` (invalid client secret).   | `AZURE_CLIENT_SECRET` is wrong or expired.                                                  | Generate a new secret under the app registration and update the GitHub secret.                                   |
| `az apim api import` fails with `AuthorizationFailed`.                   | Service principal lacks role on the APIM resource.                                          | Re-run `az role assignment create --role "API Management Service Contributor" --assignee <appId> --scope <APIM>`. |
| Spec imports but appears under wrong API name.                           | `AZURE_APIM_API_PATH` collides with another API in the same APIM instance.                  | Pick a unique path or move other APIs.                                                                           |
| Publish job succeeds, `az apim api show` returns the API, but it doesn't appear in the dev portal catalog. | API isn't associated with any published Product. The dev portal only lists APIs in at least one Product. | Run [step 7](#7-devops-make-the-api-visible-in-the-developer-portal) to add the API to `starter` / `unlimited` (or your bespoke product). |
| Dev portal Try-It returns a backend error (`500`, `failed to fetch`, `connection refused`).            | No real backend is deployed; the `serviceUrl` points at a non-existent or unreachable host.                  | Run [step 8](#8-devops-enable-try-it-mock-responses) to apply the `mock-response` policy so APIM returns the spec's example payloads. |

---

## Related documents

- [`docs/AZURE-APIM-RELEASE-FLOW.md`](./AZURE-APIM-RELEASE-FLOW.md) — design proposal for the multi-tenant (non-live + live) publish flow built on top of this guide.
- [`docs/API-VERSIONING-STRATEGY.md`](./API-VERSIONING-STRATEGY.md) — header-based versioning rules this workflow mirrors.
- [`docs/OPENAPI-SPEC-VERSIONING.md`](./OPENAPI-SPEC-VERSIONING.md) — how `Artefact-Version` and `Update-Spec-Version` stamp the spec before publish.
- [`docs/GITHUB-ACTIONS.md`](./GITHUB-ACTIONS.md) — overview of all CI workflows in this repo.
