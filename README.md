# Basic Github runner & Action showcase

A FastAPI application with a fully automated CI/CD pipeline deploying to a self-hosted Raspberry Pi 5 Kubernetes cluster.

## Infrastructure

- **Raspberry Pi 5** running k3s (lightweight Kubernetes)
- **Namespaced cluster** with RBAC — users are scoped to their own namespace via x509 certificates
- **Private Docker registry** deployed as a k3s workload with TLS, backed by local storage
- **Tailscale** for secure private networking between the Pi and external services — no open ports or port forwarding required

## CI/CD Pipeline

Pushes to `main` trigger a GitHub Actions workflow that:

1. Connects to the Pi's private network via Tailscale
2. Builds a multi-arch Docker image (ARM64) using Buildx
3. Pushes the image to the private registry on the Pi
4. Applies the Kubernetes deployment manifest via `kubectl`
5. Verifies rollout success before completing

GitHub-hosted runners handle the build and deploy steps — the Pi only receives and runs the final workload.
