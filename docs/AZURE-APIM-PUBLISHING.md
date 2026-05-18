# Publishing the OpenAPI Spec to Azure API Management

This guide describes how the OpenAPI spec for this repository is published to **Azure API Management (APIM)**, where it is rendered in the APIM **Developer Portal** for consumers.

Publishing is fully automated by GitHub Actions:

| Trigger                                | Workflow                                                                | Behaviour                                                                  |
|----------------------------------------|-------------------------------------------------------------------------|----------------------------------------------------------------------------|
| Push / merge to `main`                 | [`ci-draft.yml`](../.github/workflows/ci-draft.yml) → `Push-Draft-OpenAPI-Spec-Azure-APIM` | Imports the spec into the **current revision** of the `v1` API in APIM (overwrites in place). |
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
                                              ├─ ensure versionSet exists (Header + Accept)
                                              ├─ az apim api import
                                              └─ write run summary
                                                              │
                                                              ▼
                                              Azure APIM Developer Portal
```

The release flow is identical except that the import targets a fresh revision and a subsequent `az apim api release create` promotes that revision to current.

---

## API model in APIM

The workflow creates and maintains the following objects in APIM:

| Object        | ID / naming                          | Notes                                                                            |
|---------------|--------------------------------------|----------------------------------------------------------------------------------|
| Version set   | `<repo-name>-vs`                     | Scheme = `Header`, header = `Accept`. Matches the project's media-type strategy. |
| API           | `<repo-name>-v1`                     | One API per semver MAJOR. Subsequent majors create `<repo-name>-v2`, etc.        |
| Revisions     | `1`, `2`, …                          | Drafts overwrite revision 1. Releases create a new revision and promote it.      |
| URL path      | value of `AZURE_APIM_API_PATH`       | The path consumers hit on the gateway (e.g. `cp/crime/echo`).                    |

Consumers select the API version via the `Accept` header, e.g.:

```
Accept: application/vnd.hmcts.cp.v1+json
```

See [`docs/API-VERSIONING-STRATEGY.md`](./API-VERSIONING-STRATEGY.md) for the full versioning rules.

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

Merge a change to `main`. The `Push-Draft-OpenAPI-Spec-Azure-APIM` job should appear in the run, and the spec should appear in the APIM Developer Portal within a minute.

---

## Ongoing responsibilities

After the one-time setup, the pipeline runs unattended on every merge to `main`. The two roles still own things long-term:

### DevOps

- **Rotate the SP secret** on a schedule (suggested: yearly, or immediately if it's suspected to have leaked). Use `az ad sp credential reset --id <AZURE_CLIENT_ID> --display-name "rotation-YYYY-MM"` and hand the new password to the developer.
- **Maintain the RBAC assignment.** Do not broaden the SP's scope beyond the single APIM resource without a documented reason.
- **Triage Azure-side failures** in CI runs — anything where the failing step is `Azure login`, `Ensure APIM versionSet exists`, or `Import spec to APIM`. The error message in the job log is usually self-explanatory (`AuthorizationFailed`, `ResourceNotFound`, etc.). See [Common failures](#common-failures-and-fixes).
- **Decommission the SP** when the repo is retired (`az ad sp delete --id <AZURE_CLIENT_ID>`).

### Developer

- **Maintain the OpenAPI spec** at `src/main/resources/openapi/openapi-spec.yml`. Linting and schema validation run on every PR.
- **Decide when to bump `apim_api_version_id`** in `.github/workflows/ci-draft.yml` (e.g. `v1` → `v2`) when introducing a breaking change per [`API-VERSIONING-STRATEGY.md`](./API-VERSIONING-STRATEGY.md). A new value creates a new API + version-set entry in APIM rather than overwriting the existing one.
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
| `apim_api_version_id`    | `v1` (literal)                                          | Bump to `v2` when introducing a breaking change (creates a new APIM API + version).  |
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
     --api-id        api-cp-crime-echo-v1
   ```
3. **APIM Developer Portal** — operations and schemas reflect the latest spec.

---

## Common failures and fixes

| Symptom                                                                  | Likely cause                                                                                | Fix                                                                                                              |
|--------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|------------------------------------------------------------------------------------------------------------------|
| `azure/login` step fails with `AADSTS7000215` (invalid client secret).   | `AZURE_CLIENT_SECRET` is wrong or expired.                                                  | Generate a new secret under the app registration and update the GitHub secret.                                   |
| `az apim api import` fails with `AuthorizationFailed`.                   | Service principal lacks role on the APIM resource.                                          | Re-run `az role assignment create --role "API Management Service Contributor" --assignee <appId> --scope <APIM>`. |
| Job fails on `versionset create` with `EntityAlreadyExists`.             | Race or version-set was created with a different scheme.                                    | Delete the existing version-set in APIM (if safe) or align the workflow's scheme to match.                       |
| Spec imports but appears under wrong API name.                           | `AZURE_APIM_API_PATH` collides with another API in the same APIM instance.                  | Pick a unique path or move other APIs.                                                                           |

---

## Related documents

- [`docs/API-VERSIONING-STRATEGY.md`](./API-VERSIONING-STRATEGY.md) — header-based versioning rules this workflow mirrors.
- [`docs/OPENAPI-SPEC-VERSIONING.md`](./OPENAPI-SPEC-VERSIONING.md) — how `Artefact-Version` and `Update-Spec-Version` stamp the spec before publish.
- [`docs/GITHUB-ACTIONS.md`](./GITHUB-ACTIONS.md) — overview of all CI workflows in this repo.
