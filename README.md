# Two-Phase Okteto Deployment Demo

This project demonstrates how to use Okteto to create a global base image and then use it in a separate application deployment at a different time. The project is organized into two distinct phases with separate folders for clarity.

## Architecture

```
📁 phase1-assets/                    📁 phase2-app/
┌─────────────────────────────────┐  ┌─────────────────────────────────┐
│ Phase 1: Build Global Asset     │  │ Phase 2: Use Global Asset       │
│ Image (NOT DEPLOYED)            │  │ Image in Application            │
│                                 │  │                                 │
│  ┌─────────────────────────────┐│  │  ┌─────────────────────────────┐│
│  │ okteto.yaml                 ││  │  │ okteto.yaml                 ││
│  │ Dockerfile.base             ││  │  │ Dockerfile.nginx            ││
│  └─────────────────────────────┘│  │  │ k8s.yaml                    ││
│           ↓ (okteto build)       │  │  └─────────────────────────────┘│
│  ┌─────────────────────────────┐│  │           ↓ (okteto deploy)     │
│  │ alpine:latest               ││  │  ┌─────────────────────────────┐│
│  │ + bash                      ││  │  │ nginx:alpine                ││
│  │ + HTML generation script    ││  │  │ + HTML from global image    ││
│  │ → /tmp/index.html           ││  │  └─────────────────────────────┘│
│  └─────────────────────────────┘│  └─────────────────────────────────┘
└─────────────────────────────────┘
           ↓ (pushed to registry)
┌─────────────────────────────────┐
│   Okteto Global Registry        │
│  html-generator:v1.0.0          │ ← Referenced by phase2 with build args
│  (contains HTML assets only)    │
└─────────────────────────────────┘
```

## Project Structure

```
/workspace/
├── phase1-assets/           # Phase 1: Asset Generation
│   ├── okteto.yaml          # Okteto manifest for building global asset image
│   └── Dockerfile.base      # Dockerfile that creates HTML generator image
├── phase2-app/              # Phase 2: Application Deployment  
│   ├── okteto.yaml          # Okteto manifest for deploying nginx application
│   ├── Dockerfile.nginx     # Dockerfile that copies assets using build args
│   └── k8s.yaml             # Kubernetes manifests for deployment/service/ingress
└── README.md                # This documentation
```

### Phase 1 - Global Asset Image Creation (Build Only)
- **`phase1-assets/okteto.yaml`**: Okteto manifest for building the global asset image (no deployment)
- **`phase1-assets/Dockerfile.base`**: Dockerfile that creates the HTML generator image with assets

### Phase 2 - Application Deployment
- **`phase2-app/okteto.yaml`**: Okteto manifest for deploying the nginx application with build arguments
- **`phase2-app/Dockerfile.nginx`**: Dockerfile that copies assets from the global image using build args
- **`phase2-app/k8s.yaml`**: Kubernetes manifests for deployment, service, and ingress

## Usage

### Phase 1: Create Global Asset Image

This step builds and pushes an asset image to the Okteto registry that contains generated HTML files. **This image is not deployed** - it only serves as a source of assets for other applications.

```bash
# Navigate to phase1-assets directory
cd phase1-assets/

# Build and push the global asset image (no deployment)
okteto build html-generator
```

The asset image will be available at:
`registry.${OKTETO_DOMAIN}/${OKTETO_NAMESPACE}/workspace-html-generator:v1.0.0`

**Note**: This image contains HTML assets at `/tmp/index.html` and is meant to be used as a source for copying assets, not for running containers.

**Admin Tip**: If you use the "okteto" namespace instead of your personal namespace (e.g., `registry.${OKTETO_DOMAIN}/okteto/workspace-html-generator:v1.0.0`), the image will be available for all users of the Okteto instance. However, this requires admin privileges and should only be done by cluster administrators.

### Phase 2: Deploy Application Using Global Asset Image

This step can be done at any time after Phase 1, even in a different session or by a different team member. The nginx application will copy HTML assets from the global image using build arguments and serve them.

```bash
# Navigate to phase2-app directory
cd phase2-app/

# Deploy the nginx application that copies assets from the global image
okteto deploy --wait
```

The application will be available at:
`https://nginx-ingress-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}`

## Dynamic Registry References

The application uses a single build argument to dynamically reference the base image:

```dockerfile
ARG BASE_IMAGE
FROM ${BASE_IMAGE} as html-source
```

This is set in the okteto.yaml:

```yaml
build:
  nginx:
    args:
      BASE_IMAGE: registry.${OKTETO_DOMAIN}/${OKTETO_NAMESPACE}/workspace-html-generator:v1.0.0
```

This approach ensures the application works across different Okteto environments without hardcoded registry URLs.

## Key Benefits

1. **Separation of Concerns**: Asset generation is separate from application deployment
2. **Reusability**: The asset image can be used by multiple applications to copy assets
3. **Efficiency**: Applications don't need to regenerate assets each time
4. **Team Collaboration**: Different teams can manage asset generation vs applications
5. **Caching**: Asset images are cached in the registry for faster builds
6. **Asset Management**: Centralized management of static assets across applications
7. **Portability**: Single BASE_IMAGE variable makes configuration cleaner and more portable

---

*This demo was developed in collaboration with **Okteto AI Agent Fleets**, showcasing the power of AI-assisted cloud-native development workflows and best practices for global asset management in Kubernetes environments.*

## Testing and Validation

### Run Tests

```bash
# Test Phase 1: Asset generation
cd phase1-assets/
okteto test validate

# Test Phase 2: Application functionality and endpoint accessibility
cd ../phase2-app/
okteto test validate
```

### Manual Verification

```bash
# Check deployed resources
kubectl get pods -n ${OKTETO_NAMESPACE}
kubectl get svc -n ${OKTETO_NAMESPACE}
kubectl get ingress -n ${OKTETO_NAMESPACE}

# Test the endpoint manually
curl https://nginx-ingress-${OKTETO_NAMESPACE}.${OKTETO_DOMAIN}
```

## Cleanup

```bash
# Destroy the application (from phase2-app directory)
cd phase2-app/
okteto destroy
```