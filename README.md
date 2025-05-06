# Platform Application Manifest Dispatch

![Latest Release](https://img.shields.io/github/v/release/p6m-actions/platform-application-manifest-dispatch?style=flat-square&label=Latest%20Release&color=blue)

## Description

A GitHub Action for dispatching repository events to update application manifests with Docker image digests. This action is typically used after building and publishing Docker images to trigger updates to Kubernetes manifests or other deployment configurations.

## Usage

```yaml
- name: Update Application Manifest
  uses: p6m-actions/platform-application-manifest-dispatch@v1
  with:
    repository: ${{ github.repository }}
    directory-name: "fe-my-app"
    resource-directory-name: "my-monorepo/apps/my-app"
    image-name: "fe-my-app"
    environment-dir: "dev"
    digest: ${{ steps.build-image.outputs.digest }}
    update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
    platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
```

## Inputs

| Name | Description | Required | Default |
|------|-------------|----------|---------|
| `repository` | The repository name in owner/repo format | Yes | |
| `directory-name` | The directory name for the application in the manifest repository | Yes | |
| `resource-directory-name` | The resource directory name where the application is located | Yes | |
| `image-name` | The name of the Docker image | Yes | |
| `environment-dir` | The environment directory (e.g., dev, stage, prod) | Yes | `dev` |
| `digest` | The Docker image digest to update | Yes | |
| `update-manifest-token` | Token used to update image manifests | Yes | |
| `platform-dispatch-url` | URL to dispatch platform updates to | Yes | |

## Outputs

| Name | Description |
|------|-------------|
| `status` | The status of the dispatch request (success or failed) |

## Examples

### Basic Example

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
          docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .
          echo "digest=$(docker inspect --format='{{index .RepoDigests 0}}' myapp:latest | cut -d'@' -f2)" >> $GITHUB_OUTPUT

      - name: Update Application Manifest
        uses: p6m-actions/platform-application-manifest-dispatch@v1
        with:
          repository: ${{ github.repository }}
          directory-name: "fe-myapp"
          resource-directory-name: "my-repo/apps/myapp"
          image-name: "fe-myapp"
          environment-dir: "dev"
          digest: ${{ steps.build-image.outputs.digest }}
          update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
          platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
```

### Example with PNPM NX Monorepo

```yaml
name: Build and Deploy

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout code
        uses: actions/checkout@v3

      - name: Setup PNPM
        uses: p6m-actions/js-pnpm-setup@v1

      - name: Build Applications
        uses: p6m-actions/js-pnpm-build@v1
        with:
          build-command: "nx run-many --target=build --all --parallel=5 --prod"

      - name: Get Affected Apps
        id: affected-apps
        run: |
          APPS=$(pnpm nx show projects --type app | cut -d, -f1)
          echo "affected-apps=$APPS" >> $GITHUB_OUTPUT

      - name: Build and Push Docker Images
        id: docker-build
        uses: p6m-actions/js-pnpm-docker-build-publish@v1
        with:
          affected-apps: ${{ steps.affected-apps.outputs.affected-apps }}
          version: "latest"
          github-token: ${{ secrets.GITHUB_TOKEN }}
          update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
          platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
          registry: ${{ env.ARTIFACTORY_REGISTRY }}

      # For each app, update the application manifest
      - name: Update Application Manifests
        run: |
          IFS=';' read -ra DIGESTS <<< "${{ steps.docker-build.outputs.image-digests }}"
          for DIGEST_ENTRY in "${DIGESTS[@]}"; do
            if [ -n "$DIGEST_ENTRY" ]; then
              APP=$(echo $DIGEST_ENTRY | cut -d':' -f1)
              DIGEST=$(echo $DIGEST_ENTRY | cut -d':' -f2-)
              
              echo "Updating manifest for $APP with digest $DIGEST"
              
              # Use the platform-application-manifest-dispatch action
              uses: p6m-actions/platform-application-manifest-dispatch@v1
              with:
                repository: ${{ github.repository }}
                directory-name: "fe-$APP"
                resource-directory-name: "$(basename ${{ github.repository }})/apps/$APP"
                image-name: "fe-$APP"
                environment-dir: "dev"
                digest: "$DIGEST"
                update-manifest-token: ${{ secrets.UPDATE_MANIFEST_TOKEN }}
                platform-dispatch-url: ${{ vars.PLATFORM_DISPATCH_URL }}
            fi
          done
```