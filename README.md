# Build & Deploy Reusable Workflow (shared.yaml)

This workflow is a reusable GitHub Actions pipeline for building, scanning, and deploying containerized applications. It orchestrates the `build.yaml` and `deploy.yaml` workflows, supporting multiple languages and frameworks, including JavaScript (Node.js), PHP, .NET, Java, and Python.

## Inputs

| Name                  | Required | Default Value                                 | Description                                      |
|-----------------------|----------|-----------------------------------------------|--------------------------------------------------|
| app-name              | Yes      |                                               | Application name                                 |
| argocd-host           | Yes      |                                               | ArgoCD hostname                                  |
| argocd-version        | No       | 2.14.11                                       | ArgoCD binary version                            |
| build-context         | No       | .                                             | Build context for Docker build                   |
| cr-host               | No       | docker.io                                     | Container registry hostname                      |
| cr-login              | No       | providus                                      | Username for container registry                  |
| debug                 | No       | false                                         | Enable debug mode (passed to deploy only)        |
| dockerfile            | No       | Dockerfile                                    | Path to Dockerfile                               |
| enable-scan           | No       | false                                         | Enable SonarQube code scan (see note below)      |
| env-name              | No       | dev                                           | Environment name (passed to deploy only)         |
| image-name            | Yes      |                                               | Name of the image, e.g. `quay.io/providus/tenant-service` or `providus/tenant-service` |
| runner-label          | No       | ubuntu-24.04                                  | Runner label (passed to deploy only)             |
| scan-image            | No       |                                               | Image vulnerability scan threshold (see note below) |
| sonar-host            | No       | https://sonarqube.infra.providus.rs/          | SonarQube server address                         |
| sonar-scanner-version | No       | 6.2.1.4610                                    | Sonar scanner version                            |
| tenant                | Yes      |                                               | Tenant name                                      |
| test-report-path      | No       | target/surefire-reports/**/TEST-*.xml         | Path to test report files                        |
| test-report-type      | No       | java-junit                                    | Type of test report files                        |

> **Note:** `shared.yaml` still forwards `enable-scan` and `scan-image` to `build.yaml`, but `build.yaml` no longer declares these inputs (they were replaced by `sonar-token` presence and `scan-security`). The `build.yaml` inputs `tag-suffix`, `sonar-root`, `scan-security` and `build-args` are not exposed through `shared.yaml`; call `build.yaml` directly to use them.

## Secrets

| Name         | Required | Description                |
|--------------|----------|----------------------------|
| cr-pass      | Yes      | Registry token             |
| argocd-token | Yes      | ArgoCD token               |
| sonar-token  | No       | SonarQube token            |
| snyk-token   | No       | Snyk token                 |

## Jobs

### build
- Calls `build.yaml` workflow to build and push Docker images.
- Outputs `image-tag` for use in the deploy job.

### deploy
- Depends on the `build` job completion.
- Calls `deploy.yaml` workflow to deploy the application using ArgoCD.
- Updates the image tag in ArgoCD application parameters.

## Usage

Reference this workflow in your repository:

```yaml
jobs:
  build-deploy:
    uses: shared-github/.github/workflows/shared.yaml@main
    with:
      image-name: providus/tenant-service
      argocd-host: your-argocd.example.com
      tenant: your-tenant
      app-name: your-app
    secrets:
      cr-pass: ${{ secrets.CR_PASS }}
      argocd-token: ${{ secrets.ARGOCD_TOKEN }}
      sonar-token: ${{ secrets.SONAR_TOKEN }}
      snyk-token: ${{ secrets.SNYK_TOKEN }}
```

# Build Workflow (`build.yaml`)

This reusable GitHub Actions workflow builds, scans, and pushes Docker images for containerized applications. Project type is detected from files in `build-context`:

| Type    | Detected via                 | Version source                    | Framework version (`FRAMEWORK_VERSION` build arg) |
|---------|------------------------------|-----------------------------------|---------------------------------------------------|
| .NET    | `*.sln` or `*.csproj`        | `.version` file                   | Highest `<TargetFramework(s)>` in `*.*proj` / `*.props` |
| Node.js | `package*.json`              | `version` in `package.json`       |                                                   |
| Java    | `pom*.xml`                   | `<version>` in `pom.xml`          | `<java.version>` property in `pom.xml`            |
| Python  | `requirements*.txt`          | `.version` file                   |                                                   |
| PHP     | `composer*.json`             | `version` in `composer.json`      |                                                   |
| Unknown | anything else (e.g. Go)      | `.version` file                   |                                                   |

The SonarQube project key is read from `projectKey` in `<sonar-root>/sonar-project.properties`, or for .NET from `sonar.projectKey` in `SonarQube.Analysis.xml`.

## Main Steps

1. **prepare**: Detects project type, version and Sonar project key; sets outputs for later jobs.
2. **scan-code**: Runs only when `sonar-token` is provided. Sets up Java 17 and runs the SonarQube scan from `sonar-root`:
   - .NET: `dotnet-sonarscanner`
   - Java: `sonar-maven-plugin` via `mvn verify`
   - Others: `sonarqube-scan-action` using `sonar-scanner-version`

   Then reports the quality gate result (decorating the PR when run on a pull request) and, if `test-report-path` is set, publishes the unit test report.
3. **build**: Builds the Docker image with buildx (passing `FRAMEWORK_VERSION` and any `build-args`). If `scan-security` is set, runs Trivy filesystem (vulnerabilities + secrets) and image scans at the given severities and publishes the output to the job summary. Finally pushes the image to the registry.

## Inputs

| Name                  | Required | Default Value                                 | Description                                      |
|-----------------------|----------|-----------------------------------------------|--------------------------------------------------|
| app-name              | Yes      |                                               | Application name                                 |
| build-args            | No       |                                               | Additional Docker build args separated by semicolon (or newline), e.g. `ARG1=value1;ARG2=value2` |
| build-context         | No       | .                                             | Build context for docker build                   |
| cr-host               | No       | docker.io                                     | Container registry hostname                      |
| cr-login              | No       | providus                                      | Username for container registry                  |
| debug                 | No       | false                                         | Enable debug mode                                |
| dockerfile            | No       | Dockerfile                                    | Path to Dockerfile                               |
| env-name              | No       | dev                                           | Environment name                                 |
| image-name            | Yes      |                                               | Name of the image, e.g. `quay.io/providus/tenant-service` or `providus/tenant-service` |
| runner-label          | No       | ubuntu-24.04                                  | Runner label                                     |
| scan-security         | No       |                                               | Enable image/code vulnerability scanning by selecting severity levels, e.g. `HIGH,CRITICAL` or `MEDIUM,HIGH,CRITICAL` |
| sonar-host            | No       | https://sonarqube.infra.providus.rs/          | SonarQube server address                         |
| sonar-root            | No       | .                                             | Root directory for Sonar scanner (location of `sonar-project.properties`) |
| sonar-scanner-version | No       | 8.5.0.16571                                   | Sonar scanner version (generic scan only)        |
| tag-suffix            | No       | (empty)                                       | Suffix added to the image tag, e.g. `-SEED` results in: `providus/tenant-service:1.0-abc1234-SEED` |
| test-report-path      | No       | (empty)                                       | Path to test report files; test report is published only when set, e.g. `target/surefire-reports/**/TEST-*.xml` |
| test-report-type      | No       | (empty)                                       | Type of test report files (`dorny/test-reporter` reporter), e.g. `java-junit` |

## Outputs

| Name      | Description                                |
|-----------|--------------------------------------------|
| image-tag | Built image tag (`<version>-<short sha>`, without `tag-suffix`) |

## Secrets

| Name         | Required | Description                                       |
|--------------|----------|---------------------------------------------------|
| cr-pass      | Yes      | Registry token                                    |
| sonar-token  | No       | SonarQube token; when set, the code scan is run   |
| snyk-token   | No       | Snyk token                                        |

## Usage

Call this workflow from another workflow using `workflow_call` and provide the required inputs and secrets.

Example:

```yaml
jobs:
  build:
    uses: shared-github/.github/workflows/build.yaml@main
    with:
      image-name: providus/tenant-service
      app-name: tenant-service
      scan-security: HIGH,CRITICAL
      build-args: "ARG1=value1;ARG2=value2"
      test-report-path: target/surefire-reports/**/TEST-*.xml
      test-report-type: java-junit
    secrets:
      cr-pass: ${{ secrets.CR_PASS }}
      sonar-token: ${{ secrets.SONAR_TOKEN }}
      snyk-token: ${{ secrets.SNYK_TOKEN }}
```

# Deploy Workflow (`deploy.yaml`)

This reusable GitHub Actions workflow deploys applications using ArgoCD. It updates the image tag in ArgoCD applications and supports both single-source and multi-source application configurations.

## Main Steps

1. **Deploy**: Downloads the ArgoCD CLI binary and updates the application image tag in ArgoCD.
   - Finds all ArgoCD apps matching `<tenant>--<app-name>--<env-name>*`.
   - Detects if the application has multiple sources and applies the appropriate configuration (`--source-position 1` for multi-source apps).
   - Updates the `image.tag` parameter.
   - Runs in the GitHub environment `env-name`, using the app's first external URL (if any) as the environment URL.

## Inputs

| Name              | Required | Default Value | Description                    |
|-------------------|----------|---------------|--------------------------------|
| app-name          | Yes      |               | Application name               |
| argocd-host       | Yes      |               | ArgoCD hostname                |
| argocd-version    | No       | 2.14.11       | ArgoCD binary version          |
| debug             | No       | false         | Enable debug output            |
| env-name          | No       | dev           | Environment name               |
| image-tag         | Yes      |               | Image tag to deploy            |
| runner-label      | No       | ubuntu-slim   | Runner label                   |
| tenant            | Yes      |               | Tenant name                    |

## Secrets

| Name           | Required | Description       |
|----------------|----------|-------------------|
| argocd-token   | Yes      | ArgoCD token      |

## Usage

Call this workflow from another workflow using `workflow_call`:

```yaml
jobs:
  deploy:
    uses: shared-github/.github/workflows/deploy.yaml@main
    with:
      argocd-host: your-argocd.example.com
      tenant: your-tenant
      app-name: your-app
      image-tag: "1.0.0-abc1234"
    secrets:
      argocd-token: ${{ secrets.ARGOCD_TOKEN }}
```
