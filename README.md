# basic-runner

A basic FastAPI application with a fully automated CI/CD pipeline deploying to a self-hosted Raspberry Pi 5 Kubernetes cluster using a self-hosted Github runner.

Built to demonstrate end-to-end platform engineering skills including container orchestration, secure networking, multi-stage builds, and self-hosted runner management.

## Pipeline Overview
```
Push to main
    │
    ▼
┌─────────────────────────────┐
│   Build Job (GitHub-hosted) │
│                             │
│  Docker build stage         │
│    └── Install dependencies │
│    └── Copy source          │
│    └── Run pytest           │
│         │                   │
│         ▼ (tests pass)      │
│  Docker prod stage          │
│    └── Strip dev deps       │
│    └── Build ARM64 image    │
│    └── Push to Pi registry  │
└─────────────┬───────────────┘
              │
              ▼ (build succeeds)
┌─────────────────────────────┐
│  Deploy Job (self-hosted Pi)│
│                             │
│  kubectl apply              │
│    └── Pull from registry   │
│    └── Rolling update       │
│    └── Verify rollout       │
└─────────────────────────────┘
```

## Stack

| Layer | Technology |
|---|---|
| Runtime | Python 3.12, FastAPI, uvicorn |
| Package management | uv |
| Containerisation | Docker, k3s containerd |
| Orchestration | k3s (Kubernetes) |
| Networking | Tailscale |
| CI/CD | GitHub Actions |
| Registry | Docker Registry v2 |

---

## Infrastructure

The cluster runs on a local Raspberry Pi 5 using k3s. The cluster is namespaced with RBAC, giving each user their own isolated namespace scoped via x509 certificates.

A private Docker registry runs as a k3s workload in the cluster, backed by a hostPath volume on the Pi's local storage. TLS is enabled using a self-signed certificate, with the cert distributed to both the GitHub Actions runner (via repository secret) and k3s (via registries.yaml) so both can push and pull securely.

Tailscale provides the network layer between the Pi and the outside world. Rather than exposing any ports or configuring port forwarding.

## CI/CD Pipeline

The workflow is split into two jobs:

**Build job** (GitHub-hosted runner, ubuntu-latest):
1. Connects to the Pi's Tailscale network using an ephemeral auth key
2. Trusts the private registry's self-signed TLS certificate
3. Builds the `builder` stage of the Dockerfile for AMD64 and runs pytest inside it
4. If tests pass, builds the production image for ARM64 and pushes to the Pi registry

**Deploy job** (self-hosted runner on the Pi):
1. Installs kubectl at runtime (see limitations)
2. Decodes the kubeconfig from a repository secret
3. Applies the Kubernetes deployment manifest
4. Verifies the rollout completes successfully

The two jobs are linked with `needs: build` — the deploy job only runs if the build and tests pass.

## Testing

Tests run inside the Docker build stage rather than directly on the runner. The `builder` stage installs all dependencies including dev dependencies, copies the full application, and pytest is executed inside the container. The production stage inherits from builder but runs `uv sync --no-dev` to strip test dependencies from the final image.

This approach means:
- The production image never contains test dependencies
- Tests run in an environment identical to the build environment
- A failing test aborts the workflow before the production image is built or pushed

## Security

- Branch protection ruleset prevents direct pushes to `main`
- All changes require a pull request — workflows only trigger on push to `main`, never on PRs
- CODEOWNERS requires explicit approval for any changes to `.github/`
- Secrets are never exposed to fork PR workflows

## Limitations & Known Improvements

**Self-hosted runner:**
The runner is deployed using the `myoung34/github-runner` image which does not include kubectl by default. As a workaround, the deploy job installs kubectl at runtime via curl on every run. A production setup would use a custom runner image with kubectl pre-installed, removing the runtime dependency on external downloads and reducing job time.

**Registry storage:**
The private registry uses a hostPath volume backed by the Pi's SD card. This means the registry is tied to a single node and is vulnerable to SD card failure. A production setup would use a networked storage solution or an external registry with proper replication.

**Self-signed TLS:**
The registry certificate is self-signed and distributed manually via a repository secret. This requires manual rotation before expiry. A production setup would use cert-manager with Let's Encrypt or a proper internal CA.
