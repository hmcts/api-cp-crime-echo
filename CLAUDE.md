# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`api-cp-crime-echo` is an HMCTS Marketplace API template — an **OpenAPI-first** Spring Boot project where **all Java source code is generated from the OpenAPI spec**. There are no hand-written Java files under `src/main/java/`; implementations are expected to be added when using this template.

Naming convention: `api-{source-system}-[case-type]-{business-domain}-{entity}`. This repo uses `cp` (Common Platform) and `crime` case type.

## Build, Lint, Test

The `-DAPI_SPEC_VERSION` property is required for all Gradle tasks:

```bash
# Full build with tests
./gradlew build -DAPI_SPEC_VERSION=1.0.0

# Build without tests
./gradlew build -x test -DAPI_SPEC_VERSION=1.0.0

# Run tests only
./gradlew test -DAPI_SPEC_VERSION=1.0.0

# Coverage report (HTML + XML)
./gradlew jacocoTestReport -DAPI_SPEC_VERSION=1.0.0

# PMD static analysis
./gradlew pmdMain -DAPI_SPEC_VERSION=1.0.0

# Publish to GitHub Packages
./gradlew publish -DAPI_SPEC_VERSION=1.0.0 -DGITHUB_REPOSITORY=<repo> -DGITHUB_ACTOR=<actor> -DGITHUB_TOKEN=<token>
```

OpenAPI linting (requires Node.js 22+):
```bash
npm install -g @stoplight/spectral-cli
spectral lint "src/main/resources/openapi/*.{yml,yaml}"
```

JSON schema + payload validation:
```bash
npm install -g jsonlint ajv-cli
jsonlint -q ./openapi/path/to/example_payload.json
ajv --spec=draft2020 --strict=false -s "./openapi/path/to/schema.json" -d "./openapi/path/to/example_payload.json"
```

## Architecture

### Code Generation Flow

The OpenAPI Generator (Spring plugin) reads `src/main/resources/openapi/openapi-spec.yml` and emits Java into `build/generated/src/main/java/`:

- **API interfaces** → `uk.gov.hmcts.cp.openapi.api.*`
- **Model classes** → `uk.gov.hmcts.cp.openapi.model.*`

Key generator settings (in `gradle/openapi.gradle`):
- Spring generator with `useSpringBoot3` and `useTags`
- `OffsetDateTime` mapped to `java.time.Instant`
- Lombok annotations (`@Builder`, `@AllArgsConstructor`, `@NoArgsConstructor`) applied to all models

Implementations should extend the generated API interfaces. Don't modify generated code directly.

### OpenAPI Spec Conventions

- Spec lives at: `src/main/resources/openapi/openapi-spec.yml` (filename must end in `.openapi.yml` in general; see `docs/OPENAPI-FILE-CONVENTIONS.md`)
- JSON schemas for request/response payloads go under `src/main/resources/openapi/schema/`
- One spec per repository

### API Versioning

Media-type versioning via `Accept` header — **not** URL path versioning:
```
Accept: application/vnd.hmcts.cp.v<MAJOR>[.<MINOR>[.<PATCH>]]+json
```
See `docs/API-VERSIONING-STRATEGY.md` for details on breaking vs. non-breaking change rules.

### Artefact Versioning

Version is injected at build time via `-DAPI_SPEC_VERSION` (defaults to `0.0.999` if omitted). CI produces:
- Draft builds: `vX.Y.Z-<short-sha>` (on PR / push to main)
- Release builds: `X.Y.Z` (on GitHub release publish)

### Build Configuration Structure

Gradle config is split across `gradle/`:
- `java.gradle` — Java 25 toolchain, `-Werror`, `-Xlint:unchecked`
- `openapi.gradle` — OpenAPI Generator config
- `test.gradle` — JUnit 5, JaCoCo, fast-fail
- `pmd.gradle` — PMD using `.github/pmd-ruleset.xml` (skips generated code)
- `repositories.gradle` — Maven repos + GitHub Packages / Azure Artifacts publishing
- `jar.gradle` — JAR includes `CHANGELOG.md` and `bom.json` SBOM in `META-INF/`

### SBOM & Supply Chain

CycloneDX generates `bom.json` embedded in every JAR at `META-INF/sbom/bom.json`. See `docs/CHAIN_OF_CUSTODY.md`.

### Logging

Logback with Logstash JSON encoder (console output). Config: `src/main/resources/logback.xml`. Root level: INFO.

## Key Tests

`src/test/java/uk/gov/hmcts/cp/config/OpenApiObjectsTest.java` validates the generated code — confirms expected API methods exist, model fields are present, and `OffsetDateTime` fields are mapped to `Instant`.
