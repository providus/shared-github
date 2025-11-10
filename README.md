# Build & Deploy Reusable Workflow (shared.yaml)

This workflow is a reusable GitHub Actions pipeline for building, scanning, and deploying containerized applications. It supports multiple languages and frameworks, including Go, JavaScript (Node.js), PHP, and .NET.

## Inputs

| Name                  | Required | Default Value                                 | Description                                      |
|-----------------------|----------|-----------------------------------------------|--------------------------------------------------|
| cr-host               | No       | docker.io                                     | Container registry hostname                      |
| cr-login              | No       | providus                                      | Username for container registry                  |
| build-context         | No       | .                                             | Build context for Docker build                   |
| dockerfile            | No       | Dockerfile                                    | Path to Dockerfile                               |
| image-name            | Yes      |                                               | Name of the image (e.g., providus/tenant-service)|
| argocd-host           | Yes      |                                               | ArgoCD hostname                                  |
| argocd-version        | No       | 2.11.7                                        | ArgoCD binary version                            |
| tenant                | Yes      |                                               | Tenant name                                      |
| app-name              | Yes      |                                               | Application name                                 |
| sonar-host            | No       | https://sonarqube.infra.providus.rs/          | SonarQube server address                         |
| sonar-scanner-version | No       | 6.2.1.4610                                    | Sonar scanner version                            |
| env-name              | No       | dev                                           | Environment name                                 |
| runner-label          | No       | ubuntu-24.04                                  | Runner label                                     |
| debug                 | No       | false                                         | Enable debug mode                                |
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

### prepare
- Detects project type and version.
- Sets outputs for use in later jobs.

### scan
- Runs SonarQube scan if `sonar-token` is provided.
- Supports .NET, Java, and generic projects.
- Publishes test reports.

### build
- Builds and pushes Docker images.
- Optionally runs Snyk scan if `snyk-token` is provided.

### deploy
- Deploys the application using ArgoCD.

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
2. **scan**: Runs SonarQube scan if `sonar-token` is provided, supports .NET, Java, and generic projects.
3. **build**: Builds and pushes Docker images, optionally runs Snyk scan if `snyk-token` is provided.

## Inputs

| Name                  | Required | Default Value                                 | Description                                      |
|-----------------------|----------|-----------------------------------------------|--------------------------------------------------|
| cr-host               | No       | docker.io                                     | Container registry hostname                      |
| cr-login              | No       | providus                                      | Username for container registry                  |
| build-context         | No       | .                                             | Build context for Docker build                   |
| dockerfile            | No       | Dockerfile                                    | Path to Dockerfile                               |
| image-name            | Yes      |                                               | Name of the image (e.g., providus/tenant-service)|
| tag-suffix            | No       | SEED                                          | Suffix for image tag                             |
| app-name              | Yes      |                                               | Application name                                 |
| sonar-host            | No       | https://sonarqube.infra.providus.rs/          | SonarQube server address                         |
| sonar-scanner-version | No       | 6.2.1.4610                                    | Sonar scanner version                            |
| runner-label          | No       | ubuntu-24.04                                  | Runner label                                     |
| debug                 | No       | false                                         | Enable debug mode                                |

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
