# NGINX Demo on Kubernetes with Jenkins CI/CD

Minimal lab setup (cleaned):
1. Jenkins runs in Kubernetes.
2. Kaniko builds and pushes image.
3. Helm deploys app.
4. App is exposed with Service `NodePort` on port `30081`.

---

## Repository Layout

```text
.
├── Dockerfile
├── index.html
├── Jenkinsfile
├── helm/nginx-demo/
│   ├── Chart.yaml
│   ├── values.yaml
│   └── templates/
│       ├── _helpers.tpl
│       ├── deployment.yaml
│       └── service.yaml
└── k8s/
    ├── jenkins-serviceaccount.yaml
    ├── jenkins-deployment.yaml
    └── jenkins-helm-cluster-rbac.yaml
```

---

## 1) Install Jenkins

```bash
helm repo add jenkins https://charts.jenkins.io
helm repo update

kubectl create namespace jenkins --dry-run=client -o yaml | kubectl apply -f -

helm upgrade --install jenkins jenkins/jenkins \
  -n jenkins \
  --set controller.serviceType=NodePort
```

Verify:

```bash
kubectl get pods -n jenkins
kubectl get svc -n jenkins
```

---

## 2) Apply Jenkins RBAC

```bash
kubectl apply -f k8s/jenkins-serviceaccount.yaml
kubectl apply -f k8s/jenkins-helm-cluster-rbac.yaml
```

Verify:

```bash
kubectl auth can-i list secrets --as=system:serviceaccount:jenkins:jenkins -n dev
```

Expected: `yes`

---

## 3) Jenkins Pipeline Parameters

- `BRANCH` (default: `main`)
- `IMAGE_REPOSITORY` (default: `privatergistry/nginx-demo`)
- `IMAGE_TAG` (empty = `BUILD_NUMBER`)
- `RELEASE_NAME` (default: `nginx-demo`)
- `K8S_NAMESPACE` (default: `dev`)
- `HELM_CHART_PATH` (default: `helm/nginx-demo`)

---

## 4) Current App Exposure

Helm chart defaults:
- `service.type: NodePort`
- `service.port: 80`
- `service.targetPort: 8080`
- `service.nodePort: 30081`

No ingress resource is created in this cleaned lab setup.

---

## 5) Validate Deployment

```bash
kubectl get deploy,po,svc -n dev
helm list -n dev
kubectl get deployment nginx-demo -n dev -o jsonpath='{.spec.template.spec.containers[0].image}'; echo
```

Access app:

```text
http://<NODE_IP>:30081
```

---

## 6) Troubleshooting

### Pod fails with permission errors on NGINX temp/cache dirs
Use the current Dockerfile base image:
- `nginxinc/nginx-unprivileged:stable-alpine`

### `secrets is forbidden` or `serviceaccounts is forbidden`
Re-apply:
- `k8s/jenkins-helm-cluster-rbac.yaml`

---

## 7) Quick Checklist

1. `dockerhub-creds` exists in Jenkins.
2. Jenkins SA exists in `jenkins` namespace.
3. Cluster RBAC applied.
4. Pipeline completed successfully.
5. `kubectl get svc -n dev` shows `NodePort` `30081`.
