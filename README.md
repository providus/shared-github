# Build & Deploy Reusable Workflow (shared.yaml)

This workflow is a reusable GitHub Actions pipeline for building, scanning, and deploying containerized applications. It orchestrates the `build.yaml` and `deploy.yaml` workflows, supporting multiple languages and frameworks, including Go, JavaScript (Node.js), PHP, .NET, Java, and Python.

## Inputs

| Name                  | Required | Default Value                                 | Description                                      |
|-----------------------|----------|-----------------------------------------------|--------------------------------------------------|
| app-name              | Yes      |                                               | Application name                                 |
| argocd-host           | Yes      |                                               | ArgoCD hostname                                  |
| argocd-version        | No       | 2.14.11                                       | ArgoCD binary version                            |
| build-context         | No       | .                                             | Build context for Docker build                   |
| cr-host               | No       | docker.io                                     | Container registry hostname                      |
| cr-login              | No       | providus                                      | Username for container registry                  |
| debug                 | No       | false                                         | Enable debug mode                                |
| dockerfile            | No       | Dockerfile                                    | Path to Dockerfile                               |
| enable-scan           | No       | false                                         | Enable SonarQube code scan                       |
| env-name              | No       | dev                                           | Environment name                                 |
| image-name            | Yes      |                                               | Name of the image (e.g., providus/tenant-service)|
| runner-label          | No       | ubuntu-24.04                                  | Runner label                                     |
| scan-image            | No       |                                               | Enable image vulnerability scanning by selecting threshold level (e.g., HIGH or CRITICAL) |
| sonar-host            | No       | https://sonarqube.infra.providus.rs/          | SonarQube server address                         |
| sonar-scanner-version | No       | 6.2.1.4610                                    | Sonar scanner version                            |
| tenant                | Yes      |                                               | Tenant name                                      |
| test-report-path      | No       | target/surefire-reports/**/TEST-*.xml         | Path to test report files                        |
| test-report-type      | No       | java-junit                                    | Type of test report files                        |

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
- Detects project type (Java, .NET, Node.js, Python, Go, etc.).
- Runs SonarQube code scan if `enable-scan` is true and `sonar-token` is provided.
- Optionally runs image vulnerability scanning if `scan-image` threshold is specified.
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

This reusable GitHub Actions workflow builds, scans, and pushes Docker images for containerized applications. It supports multiple languages and frameworks, including:
- Go (detected via `go.mod`)
- JavaScript/Node.js (detected via `package.json`)
- PHP
- .NET (detected via `.csproj`)
- Java (detected via `pom.xml`)
- Python (detected via `requirements.txt`)

## Main Steps

1. **prepare**: Detects project type and version, sets outputs for later jobs.
2. **scan-code**: Runs SonarQube scan if `enable-scan` is true and `sonar-token` is provided. Supports .NET, Java, and generic projects. Publishes test reports.
3. **build**: Builds Docker images using buildx, optionally scans with Trivy if `scan-image` threshold is specified, and pushes to registry.

## Inputs

| Name                  | Required | Default Value                                 | Description                                      |
|-----------------------|----------|-----------------------------------------------|--------------------------------------------------|
| app-name              | Yes      |                                               | Application name                                 |
| build-context         | No       | .                                             | Build context for docker build                   |
| cr-host               | No       | docker.io                                     | Container registry hostname                      |
| cr-login              | No       | providus                                      | Username for container registry                  |
| debug                 | No       | false                                         | Enable debug mode                                |
| dockerfile            | No       | Dockerfile                                    | Path to Dockerfile                               |
| enable-scan           | No       | false                                         | Enable SonarQube code scan                       |
| env-name              | No       | dev                                           | Environment name                                 |
| image-name            | Yes      |                                               | Name of the image (e.g., providus/tenant-service)|
| runner-label          | No       | ubuntu-24.04                                  | Runner label                                     |
| scan-image            | No       |                                               | Enable image vulnerability scanning by selecting threshold level (e.g., HIGH or CRITICAL) |
| sonar-host            | No       | https://sonarqube.infra.providus.rs/          | SonarQube server address                         |
| sonar-scanner-version | No       | 6.2.1.4610                                    | Sonar scanner version                            |
| tag-suffix            | No       | (empty)                                       | Suffix added to the image tag, e.g., -SEED results in: providus/tenant-service:1.0-abc1234-SEED |
| test-report-path      | No       | target/surefire-reports/**/TEST-*.xml         | Path to test report files                        |
| test-report-type      | No       | java-junit                                    | Type of test report files                        |

## Outputs

| Name      | Description                                |
|-----------|--------------------------------------------|
| image-tag | Built image tag with version and commit SHA |

## Secrets

| Name         | Required | Description                |
|--------------|----------|----------------------------|
| cr-pass      | Yes      | Registry token             |
| sonar-token  | No       | SonarQube token            |
| snyk-token   | No       | Snyk token                 |

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
    secrets:
      cr-pass: ${{ secrets.CR_PASS }}
      sonar-token: ${{ secrets.SONAR_TOKEN }}
      snyk-token: ${{ secrets.SNYK_TOKEN }}
```

# Deploy Workflow (`deploy.yaml`)

This reusable GitHub Actions workflow deploys applications using ArgoCD. It updates the image tag in ArgoCD applications and supports both single-source and multi-source application configurations.

## Main Steps

1. **Deploy**: Downloads the ArgoCD CLI binary and updates the application image tag in ArgoCD.
   - Detects if the application has multiple sources and applies the appropriate configuration.
   - Updates the `image.tag` parameter for the specified tenant, app-name, and environment.

## Inputs

| Name              | Required | Default Value | Description                    |
|-------------------|----------|----------------|--------------------------------|
| app-name          | Yes      |                | Application name               |
| argocd-host       | Yes      |                | ArgoCD hostname                |
| argocd-version    | No       | 2.14.11        | ArgoCD binary version          |
| debug             | No       | false          | Enable debug output            |
| env-name          | No       | dev            | Environment name               |
| image-tag         | Yes      |                | Image tag to deploy            |
| runner-label      | No       | ubuntu-24.04   | Runner label                   |
| tenant            | Yes      |                | Tenant name                    |

## Secrets

| Name           | Required | Description      |
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

