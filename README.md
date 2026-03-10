# nginx-demo

Kubernetes + Jenkins lab project.

## What this repo does

1. Builds a container image with Kaniko in Jenkins Kubernetes agents.
2. Pushes the image to Docker Hub.
3. Deploys the app with Helm.
4. Exposes the app using Service `NodePort`.

## Current Runtime Defaults

- Image base: `nginxinc/nginx-unprivileged:stable-alpine`
- Container port: `8080`
- Service type: `NodePort`
- Service port: `80`
- NodePort: `30081`
- Namespace default in pipeline: `dev`

## Core Files

- `Jenkinsfile`: CI/CD pipeline
- `Dockerfile`: app image
- `helm/nginx-demo`: Helm chart
- `k8s/jenkins-serviceaccount.yaml`: Jenkins service account
- `k8s/jenkins-helm-cluster-rbac.yaml`: Jenkins cluster RBAC
- `k8s.md`: full setup and runbook

## Quick Access

After deployment:

```bash
kubectl get svc -n dev
```

Open:

```text
http://<NODE_IP>:30081
```

## Notes

- Ingress resources are intentionally not used in the current cleaned lab flow.
- If you need ingress later, add it back as a separate optional path.
