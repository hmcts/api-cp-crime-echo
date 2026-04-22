Create a GitHub Actions reusable workflow file at `.github/workflows/publish-openapi-redocly.yml` that publishes this project's OpenAPI spec to Redocly.

Follow the same structural conventions as the existing `.github/workflows/publish-openapi-spec.yml`:
- Use `on: workflow_call` so it is a reusable workflow, called from `ci-draft.yml` and `ci-released.yml`
- Accept these inputs:
  - `upload_artifact_name` (required, string) — name of the artifact produced by the `hmcts/update-openapi-version@main` step
  - `api_version` (required, string) — version string to tag the spec with in Redocly
  - `redocly_organization` (required, string) — Redocly organization slug
  - `api_name` (required, string) — Redocly API name (defaults to the repo name in callers)
  - `is_release` (optional, boolean, default false) — when true, publish without the `-draft` channel suffix
- Accept one secret:
  - `REDOCLY_AUTHORIZATION` (required) — Redocly personal API key

The single job should:
1. Download the versioned spec artifact using `actions/download-artifact@v4` into a temp directory
2. Push to Redocly using `redocly/cli@latest` (npm-based, via `npx @redocly/cli push`):
   - Destination format: `@{redocly_organization}/{api_name}@{api_version}`
   - When `is_release` is false, append `/draft` channel: `--branch draft`
   - Set the `REDOCLY_AUTHORIZATION` env var from the secret

After creating the workflow, also add caller steps to both `ci-draft.yml` and `ci-released.yml`, mirroring how `Push-Draft-OpenAPI-Spec` calls `publish-openapi-spec.yml`:
- In `ci-draft.yml`: add a `Push-Draft-OpenAPI-Spec-Redocly` job after `Test`, passing `is_release: false`
- In `ci-released.yml`: add a `Push-Release-OpenAPI-Spec-Redocly` job, passing `is_release: true`
- Wire the `REDOCLY_AUTHORIZATION` secret from `secrets.REDOCLY_AUTHORIZATION`
- Wire `redocly_organization` from `vars.REDOCLY_ORGANISATION`

Finally, run `./gradlew build -DAPI_SPEC_VERSION=1.0.0` to confirm the existing build still passes (the new workflow files don't affect compilation).
!!