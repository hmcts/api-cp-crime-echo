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

## One-time setup

### 1. Create the Azure service principal

Run this once, with permissions to create app registrations in your tenant and assign roles on the target APIM resource:

```bash
az ad sp create-for-rbac \
  --name "gha-api-cp-crime-echo-apim-publisher" \
  --role "API Management Service Contributor" \
  --scopes "/subscriptions/<SUB_ID>/resourceGroups/<RG>/providers/Microsoft.ApiManagement/service/<APIM_NAME>"
```

The output contains everything you need to populate the secrets section below:

```json
{
  "appId":       "…",  // → AZURE_CLIENT_ID
  "displayName": "…",
  "password":    "…",  // → AZURE_CLIENT_SECRET
  "tenant":      "…"   // → AZURE_TENANT_ID
}
```

The `<SUB_ID>` you used in `--scopes` is the value for `AZURE_SUBSCRIPTION_ID`.

Why `API Management Service Contributor`? It is the least-privilege built-in role that can import APIs and manage revisions / releases. Scoping it to the single APIM resource (not the whole subscription) limits blast radius if the credential leaks.

### 2. Add GitHub repository secrets and variables

Navigate to **Settings → Secrets and variables → Actions** in the GitHub repo and add the entries in the [Required values](#required-values) section below.

### 3. Trigger the workflow

Merge a change to `main`. The `Push-Draft-OpenAPI-Spec-Azure-APIM` job should appear in the run, and the spec should appear in the APIM Developer Portal within a minute.

---

## Required values

All values below must be set in the GitHub repository before the workflow can succeed. Secrets are masked in logs; variables are not (so do **not** put sensitive data in the Variables tab).

### Secrets (`Settings → Secrets and variables → Actions → Secrets`)

| Name                    | What it is                                                           | Where to obtain                                                                                                                              |
|-------------------------|----------------------------------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------------------|
| `AZURE_CLIENT_ID`       | Application (client) ID of the service principal that publishes.     | `appId` from `az ad sp create-for-rbac` output. Also visible in **Azure portal → Microsoft Entra ID → App registrations → <your SP>**.        |
| `AZURE_CLIENT_SECRET`   | Client secret for the service principal.                             | `password` from `az ad sp create-for-rbac` output. Only shown at creation — if lost, generate a new secret under the app registration.       |
| `AZURE_TENANT_ID`       | Microsoft Entra (Azure AD) tenant ID containing the service principal. | `tenant` from `az ad sp create-for-rbac` output. Also visible in **Azure portal → Microsoft Entra ID → Overview**.                            |
| `AZURE_SUBSCRIPTION_ID` | ID of the subscription that owns the APIM resource.                  | `az account show --query id -o tsv`. Also visible in **Azure portal → Subscriptions** or on the APIM resource overview blade.                |

### Variables (`Settings → Secrets and variables → Actions → Variables`)

| Name                         | What it is                                                  | Example          | Where to obtain                                                                                                  |
|------------------------------|-------------------------------------------------------------|------------------|------------------------------------------------------------------------------------------------------------------|
| `AZURE_APIM_RESOURCE_GROUP`  | Resource group containing the APIM instance.                | `rg-hmcts-apim-nonprod` | **Azure portal → API Management services → \<your APIM\> → Overview → Resource group**.                          |
| `AZURE_APIM_SERVICE_NAME`    | Name of the APIM instance to publish into.                  | `apim-hmcts-nonprod`    | **Azure portal → API Management services →** name column. Or `az apim list --query "[].name" -o tsv`.            |
| `AZURE_APIM_API_PATH`        | URL path consumers hit on the APIM gateway.                 | `cp/crime/echo`         | A naming decision for this repo. Convention: `<source-system>/<case-type>/<entity>` from the repo name.          |

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
