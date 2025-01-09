# blockjoy-protocols

This repository contains Docker configurations for blockchain protocol nodes and their associated clients. It includes automated build processes and version management for both protocol implementations and client software.

## Repository Structure

```
.
├── clients/                # Client-specific Dockerfiles and configurations
│   ├── consensus/         # Consensus client implementations
│   │   └── lighthouse/    # Lighthouse client
│   └── exec/             # Execution client implementations
│       ├── erigon/       # Erigon client
│       └── reth/         # Reth client
├── ethereum/             # Ethereum protocol configurations
│   ├── ethereum-erigon/  # Erigon-based Ethereum node
│   └── ethereum-reth/    # Reth-based Ethereum node
├── example-chain/        # Example of a new protocol structure
│   ├── example-client1/  # First client implementation
│   └── example-client2/  # Second client implementation
└── node-base/           # Base image with common utilities and monitor
```

## Automation Features

### GitHub Actions Workflows

The repository uses GitHub Actions for continuous integration and deployment. The main workflow is `docker-build.yml` which handles:

1. Building and pushing Docker images for:
   - Base images
   - Client images
   - Protocol images

2. Protocol Image Validation:
   - Each protocol image is built from its Dockerfile
   - The workflow checks all variants defined in the protocol's `babel.yaml`
   - Each variant is validated using `nib image check`
   - The workflow will fail if any variant check fails

3. Version-Based Deployment:
   - The workflow tracks the `version` field in each protocol's `babel.yaml`
   - On every run, it compares the current version with the previous commit
   - The image is only pushed to the API if:
     - The version number has changed
     - AND all variant checks have passed
   - This ensures that protocol images are only deployed when intentionally versioned

4. Automated Updates:
   - Renovate bot monitors dependencies in:
     - Dockerfiles
     - All `babel.yaml` files (container URIs)
   - Creates pull requests for available updates

This versioning system ensures that protocol images are only pushed to production when explicitly versioned and validated, preventing accidental deployments.

### Renovate Bot Integration

Renovate bot manages dependency updates with the following features:

1. **Version Control**:
   - Prevents major version updates for clients by default
   - Allows excluding specific problematic versions using regex patterns
2. **Docker Updates**:
   - Monitors and updates base images
   - Updates client versions in Dockerfiles
3. **Automated PRs**: Creates pull requests for version updates that pass the configured rules

## Creating New Images

### Adding a New Client

1. Create a new directory under `clients/`:
```bash
clients/
└── [consensus|exec]/
    └── your-client/
        ├── Dockerfile
        └── ... (additional files)
```

2. Dockerfile requirements:
```dockerfile
# 1. Use the base image
ARG BASE_IMAGE=ghcr.io/blockjoy/node-base:v176
FROM ${BASE_IMAGE}

# 2. Set client version (required for automation)
ENV YOUR_CLIENT_VERSION=v1.2.3

# 3. Add your build steps
RUN ...
```

### Adding a New Protocol

1. Create a new directory in the root for your protocol:
```bash
.
└── your-protocol/
    └── your-protocol-client/
        ├── Dockerfile
        └── ... (additional files)
```

2. Dockerfile requirements:
```dockerfile
# 1. Reference client images using ARG
ARG CLIENT_IMAGE=ghcr.io/blockjoy/your-client:latest
FROM ${CLIENT_IMAGE}

# 2. Add protocol-specific configurations
COPY . .
```

## Build Process

1. **Local Testing**:
```bash
# Build node-base image
docker build \
  -t blockjoy/node-base:latest \
  node-base/

# Build client image using local node-base
docker build \
  --build-arg BASE_IMAGE=blockjoy/node-base:latest \
  -t blockjoy/your-client:latest \
  clients/[consensus|exec]/your-client/

# Build protocol image using local client
docker build \
  --build-arg CLIENT_IMAGE=blockjoy/your-client:latest \
  -t blockjoy/your-protocol:latest \
  your-protocol/your-protocol-client/
```

2. **Automated Builds**:
- Push changes to a branch
- Create a pull request
- GitHub Actions will:
  - Build affected images
  - Tag with version and SHA
  - Push to ghcr.io

## Version Management

- Client versions are managed in their respective Dockerfiles using `*_VERSION` environment variables
- To exclude problematic versions, add regex patterns to `renovate.json`:
```json
{
  "packageRules": [
    {
      "matchPaths": ["clients/your-client/**"],
      "matchPackagePatterns": ["YOUR_CLIENT_VERSION"],
      "ignoreVersions": ["/^1\\.2\\.3-.*/" ]
    }
  ]
}
```

## Notes

- All images are built from the `node-base` image which provides common utilities and monitoring capabilities
- Version tags are generated daily with an incremental counter (e.g., `v20250108.1`, `v20250108.2`)
- Each image also includes the git SHA for traceability
- Protocol images are named using their protocol directory name (e.g., `ethereum-erigon`)
