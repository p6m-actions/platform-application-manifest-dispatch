# Platform Application Manifest Dispatch

![Latest Release](https://img.shields.io/github/v/release/p6m-actions/platform-application-manifest-dispatch?style=flat-square&label=Latest%20Release&color=blue)

## Description

A GitHub Action for dispatching repository events to update application manifests
with Docker image digests. This action is typically used after building and
publishing Docker images to trigger updates to Kubernetes manifests or other
deployment configurations.

## Usage

```yaml
- name: Update Application Manifest
  uses: p6m-actions/platform-application-manifest-dispatch@v1
  with:
    repository: ${{ github.repository }}
    image-name: "my-application"
    environment: "dev"
    digest: ${{ steps.build-image.outputs.digest }}
    update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
    platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
```

## Inputs

| Name                    | Description                                         | Required | Default |
| ----------------------- | --------------------------------------------------- | -------- | ------- |
| `repository`            | The repository name in owner/repo format            | Yes      |         |
| `image-name`            | The name of the Docker image                        | Yes      |         |
| `environment`           | The deployment environment (e.g., dev, stage, prod) | Yes      | `dev`   |
| `digest`                | The Docker image digest to update                   | Yes      |         |
| `update-manifest-token` | Token used to update image manifests                | Yes      |         |
| `platform-dispatch-url` | URL to dispatch platform updates to                 | Yes      |         |

## Outputs

| Name     | Description                                            |
| -------- | ------------------------------------------------------ |
| `status` | The status of the dispatch request (success or failed) |

## Example

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Set up Docker Buildx
        uses: p6m-actions/docker-buildx-setup@v1

      - name: Login to Docker Registry
        uses: p6m-actions/docker-repository-login@v1
        with:
          registry: ${{ env.ARTIFACTORY_REGISTRY }}
          username: ${{ secrets.ARTIFACTORY_USERNAME }}
          password: ${{ secrets.ARTIFACTORY_IDENTITY_TOKEN }}

      - name: Build and Push Docker Image
        id: build-image
        run: |
          docker buildx build --platform linux/amd64,linux/arm64 -t ${{ env.ARTIFACTORY_REGISTRY }}/my-application:latest --push .
          echo "digest=$(docker inspect --format='{{index .RepoDigests 0}}' ${{ env.ARTIFACTORY_REGISTRY }}/my-application:latest | cut -d'@' -f2)" >> $GITHUB_OUTPUT

      - name: Update Application Manifest
        uses: p6m-actions/platform-application-manifest-dispatch@v1
        with:
          repository: ${{ github.repository }}
          image-name: "my-application"
          environment: "dev"
          digest: ${{ steps.build-image.outputs.digest }}
          update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
          platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
```