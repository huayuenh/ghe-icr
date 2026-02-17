# IBM Cloud Container Registry GitHub Action

A comprehensive GitHub Action for managing container images in IBM Cloud Container Registry. This action supports pushing, pulling, tagging, retagging images, managing namespaces, and running vulnerability scans.

## Features

- 🚀 **Push images** to IBM Cloud Container Registry
- 📥 **Pull images** from IBM Cloud Container Registry
- 🏷️ **Tag images** with additional tags
- 🔄 **Retag images** to move tags between versions
- 🗑️ **Delete images** from the registry
- 📦 **Manage namespaces** (create, delete, list)
- 🔒 **Vulnerability scanning** with IBM Cloud Vulnerability Advisor (enabled by default)
- ⚙️ **Configurable scan behavior** - choose whether to fail on vulnerabilities
- 🔄 **Automatic retry logic** - waits up to 5 minutes for scan completion
- 🌍 **Multi-region support** with automatic region detection
- ✅ **Comprehensive error handling** and logging
- 🔧 **Uses IBM Cloud CLI marketplace action** for streamlined setup

## Prerequisites

- IBM Cloud account with Container Registry access
- IBM Cloud API key with appropriate permissions
- Docker image built and available locally (for push operations)

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `apikey` | Yes | - | IBM Cloud API key for authentication |
| `image` | Conditional | - | Full image path (e.g., `us.icr.io/namespace/image:tag`). Required for push, pull, tag, retag, and delete actions |
| `local-image` | No | - | Local image name to tag and push (e.g., `myapp:latest`). If specified, this image will be tagged with the target image path before pushing |
| `action` | Yes | - | Operation to perform: `push`, `pull`, `tag`, `retag`, `delete`, or `namespace` |
| `scan` | No | `true` | Enable vulnerability scanning after push/pull operations |
| `scan-fail-on-vulnerability` | No | `true` | Fail the build if FAIL status is returned from vulnerability scan |
| `region` | No | Auto-detect | IBM Cloud region (us-south, eu-gb, etc.). Auto-detected from image path if not specified |
| `source-tag` | Conditional | - | Source tag for retag operation |
| `target-tag` | Conditional | - | Target tag for tag/retag operations |
| `namespace` | Conditional | - | Namespace name for namespace operations |
| `namespace-action` | Conditional | - | Namespace operation: `create`, `delete`, or `list` |

## Outputs

| Output | Description |
|--------|-------------|
| `scan-result` | Vulnerability scan results in JSON format |
| `image-digest` | Image digest after push/pull operation |
| `namespaces` | List of namespaces (for namespace list action) |
| `operation-status` | Status of the operation (success/failure) |

## Usage Examples

### Push Image

Push a Docker image to IBM Cloud Container Registry:

```yaml
- name: Push to IBM Cloud Container Registry
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
```

### Push Locally Built Image

Build and push a local Docker image to IBM Cloud Container Registry:

```yaml
- name: Build Docker image
  run: docker build -t myapp:latest .

- name: Push to IBM Cloud Container Registry
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    local-image: myapp:latest
    action: push
```

This will automatically tag your local `myapp:latest` image as `us.icr.io/my-namespace/my-app:v1.0.0` before pushing it to the registry.

### Push Image with Vulnerability Scanning

Push an image and run a vulnerability scan:

```yaml
- name: Push and scan image
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: true
```

### Pull Image

Pull an image from IBM Cloud Container Registry:

```yaml
- name: Pull from IBM Cloud Container Registry
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: pull
```

### Tag Image

Add a new tag to an existing image:

```yaml
- name: Tag image as latest
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: tag
    target-tag: latest
```

### Retag Image

Move a tag from one version to another:

```yaml
- name: Promote staging to production
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app
    action: retag
    source-tag: staging
    target-tag: production
```

### Delete Image

Delete an image from the registry:

```yaml
- name: Delete old image
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:old-tag
    action: delete
```

### Create Namespace

Create a new namespace in IBM Cloud Container Registry:

```yaml
- name: Create namespace
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    action: namespace
    namespace-action: create
    namespace: my-new-namespace
    region: us-south
```

### List Namespaces

List all namespaces in your IBM Cloud account:

```yaml
- name: List namespaces
  uses: ./ibmcloud-cr-action
  id: list-ns
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    action: namespace
    namespace-action: list
    region: us-south

- name: Display namespaces
  run: echo "${{ steps.list-ns.outputs.namespaces }}"
```

### Delete Namespace

Delete a namespace (this will remove all images in the namespace):

```yaml
- name: Delete namespace
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    action: namespace
    namespace-action: delete
    namespace: old-namespace
    region: us-south
```

## Complete CI/CD Pipeline Example

Here's a complete example showing a typical CI/CD workflow:

```yaml
name: Build and Deploy

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Build Docker image
        id: build
        uses: ./docker-build-action
        with:
          image-name: myapp:${{ github.sha }}
          build-args: |
            VERSION=${{ github.sha }}
            BUILD_DATE=${{ github.event.head_commit.timestamp }}
          labels: |
            org.opencontainers.image.version=${{ github.sha }}
            org.opencontainers.image.created=${{ github.event.head_commit.timestamp }}
      
      - name: Push image to IBM Cloud Container Registry
        uses: ./
        with:
          apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
          image: us.icr.io/my-namespace/my-app:${{ github.sha }}
          local-image: ${{ steps.build.outputs.image-name }}
          action: push
          scan: true
      
      - name: Tag as latest
        uses: ./
        with:
          apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
          image: us.icr.io/my-namespace/my-app:${{ github.sha }}
          action: tag
          target-tag: latest
      
      - name: Promote to production (main branch only)
        if: github.ref == 'refs/heads/main'
        uses: ./
        with:
          apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
          image: us.icr.io/my-namespace/my-app
          action: retag
          source-tag: ${{ github.sha }}
          target-tag: production
```

This example uses the complementary [Docker Build Action](./docker-build-action) to build the image before pushing it to IBM Cloud Container Registry.

## Complementary Actions

This repository includes three additional actions for a complete CI/CD pipeline:

### Docker Build Action

The [Docker Build Action](./docker-build-action) provides advanced Docker image building capabilities:

- 🐳 Docker Buildx integration
- 🌍 Multi-platform builds (linux/amd64, linux/arm64, etc.)
- 🚀 Build arguments and custom labels
- 💾 Layer caching support
- 🎯 Multi-stage build support

### Deploy Action

The [Deploy Action](./deploy-action) deploys container images to Kubernetes or Red Hat OpenShift:

- 🚀 Deploy to Kubernetes or OpenShift clusters
- 🔐 Multiple authentication methods (kubeconfig or IBM Cloud API key)
- 🏥 Health checks with configurable timeout
- 🌐 Automatic URL generation (LoadBalancer, NodePort, Routes, Ingress)
- 📊 Status verification with pod monitoring
- ⚙️ Resource management with CPU/memory limits

### Commit Status Action

The [Commit Status Action](./commit-status-action) sets GitHub commit status for pull requests:

- ✅ Set commit status (success, failure, error, pending)
- 📝 Customizable status context and description
- 🔗 Link to workflow run for easy debugging
- 🎯 Perfect for PR workflows and CI/CD pipelines

### Using All Actions Together

Complete build, push, and deploy pipeline:

```yaml
- name: Build Docker image
  id: build
  uses: ./docker-build-action
  with:
    image-name: myapp:v1.0.0
    build-args: |
      NODE_VERSION=18
      APP_ENV=production

- name: Push to IBM Cloud Container Registry
  uses: ./
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    local-image: ${{ steps.build.outputs.image-name }}
    action: push
    scan: true

- name: Deploy to Kubernetes
  id: deploy
  uses: ./deploy-action
  with:
    image: us.icr.io/my-namespace/my-app:v1.0.0
    ibmcloud-apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    cluster-name: my-cluster
    deployment-name: myapp
    namespace: production
    replicas: 3

- name: Display application URL
  run: echo "App URL: ${{ steps.deploy.outputs.application-url }}"

- name: Set success commit status
  if: success()
  uses: ./commit-status-action
  with:
    state: success
    description: "Deployment successful ✓"
    context: "CI/CD Pipeline"
    sha: ${{ github.sha }}
    github_repository: ${{ github.repository }}
```

### Documentation

- [Docker Build Action README](./docker-build-action/README.md)
- [Deploy Action README](./deploy-action/README.md)
- [Commit Status Action README](./commit-status-action/README.md)
- [Integration Guide](./INTEGRATION.md)

### Workflow Examples

Complete workflow examples are available in the [.github/workflows](./.github/workflows) directory:

- **[build-and-push.yml](./.github/workflows/build-and-push.yml)**: Complete build, push, and deploy workflow
- **[pr.yml](./.github/workflows/pr.yml)**: Pull request workflow with commit status updates
- **[deploy-openshift.yml](./.github/workflows/deploy-openshift.yml)**: OpenShift deployment example
- **[multi-platform-build.yml](./.github/workflows/multi-platform-build.yml)**: Multi-platform build example

## Supported Regions

The action supports automatic region detection from the image path. Supported regions include:

| Registry Domain | Region Code | Region Name |
|----------------|-------------|-------------|
| `us.icr.io` | `us-south` | US South (Dallas) |
| `eu.icr.io` | `eu-gb` | UK South (London) |
| `uk.icr.io` | `uk-south` | UK South (London) |
| `au.icr.io` | `au-syd` | Sydney |
| `jp.icr.io` | `jp-tok` | Tokyo |
| `de.icr.io` | `eu-de` | Frankfurt |

If the region cannot be detected from the image path, you can specify it explicitly using the `region` input.

## Vulnerability Scanning

Vulnerability scanning is **enabled by default** for push and pull operations. The action will:

1. Initiate IBM Cloud Vulnerability Advisor scan on the image
2. Poll for scan completion every 10 seconds (up to 5 minutes)
3. Parse scan results and check status (OK, WARN, FAIL, UNSUPPORTED, INCOMPLETE, UNSCANNED)
4. Output scan results in JSON format
5. Set the `scan-result` output with detailed findings

### Scan Status Behavior

- **OK**: No vulnerabilities found - build passes
- **WARN**: Warnings found - build passes
- **UNSUPPORTED**: Image type not supported for scanning - build passes
- **FAIL**: Critical vulnerabilities found - build fails (configurable)
- **INCOMPLETE/UNSCANNED**: Scan still in progress - action retries

### Configuring Scan Behavior

**Enable/Disable Scanning:**
```yaml
- name: Push without scanning
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: false  # Disable vulnerability scanning
```

**Allow Build to Continue Despite Vulnerabilities:**
```yaml
- name: Push and scan (don't fail on vulnerabilities)
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: true
    scan-fail-on-vulnerability: false  # Report but don't fail
```

**Access Scan Results:**
```yaml
- name: Push and scan
  id: push-scan
  uses: ./ibmcloud-cr-action
  with:
    apikey: ${{ secrets.IBM_CLOUD_API_KEY }}
    image: us.icr.io/my-namespace/my-app:v1.0.0
    action: push
    scan: true

- name: Check scan results
  run: |
    echo "Scan results: ${{ steps.push-scan.outputs.scan-result }}"
```

### Retry Logic

The vulnerability scan includes automatic retry logic:
- Polls every 10 seconds for scan completion
- Maximum wait time: 5 minutes (30 attempts)
- Continues while status is INCOMPLETE or UNSCANNED
- Exits immediately when scan completes with final status

## Error Handling

The action includes comprehensive error handling:

- **Input validation**: Validates all required inputs before execution
- **Authentication errors**: Clear messages for API key or login failures
- **Image not found**: Checks if images exist before operations
- **Network failures**: Handles connection issues gracefully
- **Operation failures**: Provides detailed error messages with exit codes

## Security Best Practices

1. **Store API keys securely**: Always use GitHub Secrets for the IBM Cloud API key
2. **Use least privilege**: Grant only necessary permissions to the API key
3. **Enable vulnerability scanning**: Use `scan: true` for production images
4. **Review scan results**: Check vulnerability reports before deploying
5. **Use specific tags**: Avoid using `latest` tag in production

## Troubleshooting

### Authentication Failed

If you see authentication errors:
- Verify your IBM Cloud API key is correct
- Ensure the API key has Container Registry permissions
- Check that the region is correct

### Image Not Found

If push fails with "image not found":
- Ensure the Docker image is built before pushing
- Verify the image name matches exactly
- Check that Docker is running

### Namespace Already Exists

When creating a namespace that already exists:
- The action will report the namespace exists and continue
- Use the `list` action to check existing namespaces first

### Region Detection Issues

If region auto-detection fails:
- Specify the region explicitly using the `region` input
- Ensure the image path follows the format: `<region>.icr.io/namespace/image:tag`

## Contributing

Contributions are welcome! Please feel free to submit issues or pull requests.

## License

This project is licensed under the MIT License.

## Support

For issues related to:
- **This action**: Open an issue in this repository
- **IBM Cloud Container Registry**: Consult [IBM Cloud documentation](https://cloud.ibm.com/docs/Registry)
- **IBM Cloud CLI**: See [CLI documentation](https://cloud.ibm.com/docs/cli)

## Changelog

### v1.2.0
- **New**: Added Commit Status Action for setting GitHub commit status on PRs
- **New**: Added PR workflow example with commit status updates
- **Improved**: Refactored to use IBM Cloud CLI marketplace action (`ibm-cloud-docs/github-actions/ibmcloud-cli@v1`)
- **Improved**: Simplified CLI installation and plugin setup
- **Enhanced**: Better action composition and reusability

### v1.1.0
- Added delete action for removing images from registry
- Vulnerability scanning now enabled by default
- Added configurable scan failure behavior (`scan-fail-on-vulnerability`)
- Implemented automatic retry logic for vulnerability scans (5-minute timeout)
- Fixed scan polling to ignore exit codes and only check JSON status
- Improved scan status handling (OK, WARN, FAIL, UNSUPPORTED, INCOMPLETE, UNSCANNED)

### v1.0.0
- Initial release
- Support for push, pull, tag, retag operations
- Namespace management (create, delete, list)
- Vulnerability scanning integration
- Multi-region support with auto-detection
- Comprehensive error handling and logging