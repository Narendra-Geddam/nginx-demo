Understood. Here is the **production-grade documentation** for your setup:

**Jenkins (master) → Kaniko (build) → Private Registry → Kubernetes Deployment**

---

# 1. Architecture Overview

```text
Developer → GitHub
            ↓
        Jenkins Master (Kubernetes Pod)
            ↓
    Jenkins creates Kaniko Build Pod
            ↓
        Kaniko builds Docker image
            ↓
    Push image → registry.iximiuz.com
            ↓
Jenkins Master runs kubectl deployment
            ↓
     Kubernetes pulls image
            ↓
        Application deployed
```

---

# 2. Environment Details

Cluster type: kubeadm
Runtime: containerd
Nodes: 3

```bash
Control plane: cplane-01
Workers:
  node-01
  node-02
```

Namespace:

```bash
jenkins
```

Registry:

```bash
registry.iximiuz.com
```

Repository:

```bash
https://github.com/Narendra-Geddam/nginx-demo.git
```

---

# 3. Jenkins Deployment

Jenkins runs inside Kubernetes.

Verify:

```bash
kubectl get pods -n jenkins
```

Example:

```text
jenkins-78b56c9ddc-lx2pc Running
```

Service:

```bash
kubectl get svc -n jenkins
```

Access Jenkins:

```text
http://NODE-IP:30080
```

---

# 4. Required Jenkins Plugins

Install:

```text
Kubernetes Plugin
Pipeline Plugin
Git Plugin
Docker Pipeline Plugin (optional)
Credentials Plugin
```

---

# 5. Service Account Configuration

File: jenkins-serviceaccount.yaml

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
  namespace: jenkins
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: jenkins-admin
subjects:
- kind: ServiceAccount
  name: jenkins
  namespace: jenkins
roleRef:
  kind: ClusterRole
  name: cluster-admin
  apiGroup: rbac.authorization.k8s.io
```

Apply:

```bash
kubectl apply -f jenkins-serviceaccount.yaml
```

---

# 6. Application Files

GitHub repo contains:

Dockerfile

```Dockerfile
FROM nginx:alpine
COPY index.html /usr/share/nginx/html/
EXPOSE 80
```

Deployment YAML:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-demo
  namespace: jenkins
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-demo
  template:
    metadata:
      labels:
        app: nginx-demo
    spec:
      containers:
      - name: nginx-demo
        image: registry.iximiuz.com/nginx-demo:latest
        ports:
        - containerPort: 80
```

Service YAML:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-demo-service
  namespace: jenkins
spec:
  type: NodePort
  selector:
    app: nginx-demo
  ports:
  - port: 80
    targetPort: 80
    nodePort: 30081
```

---

# 7. Jenkins Pipeline (FINAL WORKING)

Jenkinsfile:

```groovy
pipeline {

    agent none

    environment {
        REGISTRY = "registry.iximiuz.com"
        IMAGE = "nginx-demo"
        NAMESPACE = "jenkins"
    }

    stages {

        stage('Clone') {
            agent any
            steps {
                git 'https://github.com/Narendra-Geddam/nginx-demo.git'
            }
        }

        stage('Build and Push Image') {

            agent {
                kubernetes {
                    yaml """
apiVersion: v1
kind: Pod
spec:
  serviceAccountName: jenkins
  containers:
  - name: kaniko
    image: gcr.io/kaniko-project/executor:debug
    command:
    - /busybox/cat
    tty: true
"""
                }
            }

            steps {
                container('kaniko') {

                    sh '''
                    /kaniko/executor \
                      --context $(pwd) \
                      --dockerfile Dockerfile \
                      --destination $REGISTRY/$IMAGE:latest \
                      --insecure \
                      --skip-tls-verify
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {

            agent any

            steps {

                sh '''
                kubectl apply -f deployment.yaml
                kubectl apply -f service.yaml
                '''
            }
        }

        stage('Verify') {

            agent any

            steps {

                sh '''
                kubectl get pods -n $NAMESPACE
                kubectl get svc -n $NAMESPACE
                kubectl get deployment -n $NAMESPACE
                '''
            }
        }
    }
}
```

---

# 8. Deployment Verification

Check pods:

```bash
kubectl get pods -n jenkins
```

Example:

```text
nginx-demo-7b8885d9cc-ss5vf Running
```

Check service:

```bash
kubectl get svc -n jenkins
```

Example:

```text
nginx-demo-service NodePort 30081
```

---

# 9. Access Application

Browser:

```text
http://NODE-IP:30081
```

Example:

```text
http://172.16.0.2:30081
```

---

# 10. Why Kaniko is Used

Kaniko builds Docker images without Docker daemon.

Benefits:

• Works with containerd
• Secure
• No privileged mode
• Kubernetes native

---

# 11. Why Jenkins Master Runs kubectl

Best practice separation:

Kaniko pod → build only
Jenkins master → deploy only

Benefits:

• security
• stability
• proper architecture

---

# 12. Final Production Workflow

```text
GitHub push
   ↓
Jenkins pipeline triggered
   ↓
Kaniko pod builds image
   ↓
Push image → registry
   ↓
Jenkins master runs kubectl
   ↓
Kubernetes pulls image
   ↓
Application deployed
```

---

# 13. Status: SUCCESS

Your deployment is now fully functional:

```text
Image build: SUCCESS
Image push: SUCCESS
Deployment: SUCCESS
Service exposure: SUCCESS
Pipeline: SUCCESS
```

---

If you want, I can convert this into:

• PDF documentation
• Markdown documentation
• Architecture diagram
• Resume-ready project description

Just tell which one.
