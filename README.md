# Basic Frontend Application with Docker and Kubernetes

**Objective**: Containerize a simple static website (HTML + CSS) with Nginx, push the image to Docker Hub, run it on a local Kubernetes cluster created with **Kind**, and access it via port-forwarding.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Project structure](#project-structure)
3. [Task 1 — Set up your project](#task-1---set-up-your-project)
4. [Task 2 — Initialize Git](#task-2---initialize-git)
5. [Task 3 — Git Commit](#task-3---git-commit)
6. [Task 4 — Dockerize the application](#task-4---dockerize-the-application)
7. [Task 5 — Push to Docker Hub](#task-5---push-to-docker-hub)
8. [Task 6 — Set up a Kind cluster](#task-6---set-up-a-kind-cluster)
9. [Task 7 — Deploy to Kubernetes](#task-7---deploy-to-kubernetes)
10. [Task 8 — Create a Service (ClusterIP)](#task-8---create-a-service-clusterip)
11. [Task 9 — Access the application](#task-9---access-the-application)
12. [Troubleshooting & tips](#troubleshooting--tips)
13. [Cleanup](#cleanup)

---

## Prerequisites

- Git installed and configured. (`git --version`)
- Docker installed and running. (`docker --version`)
- A Docker Hub account (username and password).
- `kubectl` installed. (`kubectl version --client`)
- `kind` installed. (`kind --version`) — Kubernetes-in-Docker.

> Note: Commands shown assume a Unix-like shell (Linux / macOS / WSL). On Windows PowerShell adapt syntax where necessary.

---

## Project structure

We'll create a compact project layout:

```
frontend-static/
├─ index.html
├─ styles.css
├─ Dockerfile
└─ k8s/
   ├─ deployment.yaml
   └─ service.yaml
```

---

## Task 1 - Set up your project

1. Create the project directory and files:

```bash
mkdir frontend-static
cd frontend-static
cat > index.html <<'EOF'
<!doctype html>
<html lang="en">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>Company Landing - Demo</title>
  <link rel="stylesheet" href="styles.css" />
</head>
<body>
  <header class="hero">
    <h1>Welcome to ExampleCorp</h1>
    <p>Your partner in reliable web experiences.</p>
    <a class="cta" href="#">Learn more</a>
  </header>
  <main class="content">
    <section>
      <h2>Our Product</h2>
      <p>We build fast, secure, and reliable websites for businesses of all sizes.</p>
    </section>
  </main>
  <footer class="footer">© ExampleCorp • Built for demo</footer>
</body>
</html>
EOF

cat > styles.css <<'EOF'
:root{--accent:#0b74de;--bg:#f7fbff;--text:#1a1a1a}
*{box-sizing:border-box}
body{font-family:Inter,system-ui,Segoe UI,Helvetica,Arial;padding:0;margin:0;background:var(--bg);color:var(--text)}
.hero{padding:4rem 1rem;text-align:center;background:white;margin:2rem auto;border-radius:12px;max-width:900px;box-shadow:0 8px 24px rgba(20,30,50,0.06)}
.hero h1{margin:0 0 .5rem;font-size:2.2rem}
.hero p{margin:0 0 1rem;color:#444}
.cta{display:inline-block;padding:.6rem 1rem;border-radius:8px;background:var(--accent);color:white;text-decoration:none}
.content{max-width:900px;margin:1.5rem auto;padding:0 1rem}
.footer{text-align:center;padding:1rem;color:#666;margin-top:2rem}
EOF
```

> These are simple, minimal files to demonstrate the flow. You can replace them with your real content later.

---

## Task 2 — Initialize Git

```bash
# inside frontend-static/
git init
git add .
```

---

## Task 3 — Git commit

```bash
git commit -m "chore: initial static site with index.html and styles.css"
```

If you want to push to a remote repository (GitHub/GitLab), create the remote repo and add it:

```bash
# example for GitHub (replace with your repo URL)
git remote add origin git@github.com:YOUR_USER/frontend-static.git
git branch -M main
git push -u origin main
```

---

## Task 4 — Dockerize the application

Create a `Dockerfile` that uses the official Nginx image and copies the static files into `/usr/share/nginx/html`.

```dockerfile
# Dockerfile
FROM nginx:stable-alpine

# Remove default nginx static content (optional)
RUN rm -rf /usr/share/nginx/html/*

# Copy static site files
COPY index.html /usr/share/nginx/html/index.html
COPY styles.css /usr/share/nginx/html/styles.css

# Expose port 80
EXPOSE 80

# Use default nginx command
CMD ["nginx", "-g", "daemon off;"]
```

### Build the Docker image locally

Replace `YOUR_DOCKERHUB_USERNAME` and `frontend-static` with your values.

```bash
docker build -t YOUR_DOCKERHUB_USERNAME/frontend-static:latest .
```

You can test the image locally:

```bash
docker run --rm -p 8080:80 YOUR_DOCKERHUB_USERNAME/frontend-static:latest
# then open http://localhost:8080
```

---

## Task 5 — Push to Docker Hub

1. Log in to Docker Hub from your shell:

```bash
docker login
# enter username and password when prompted
```

2. Tag the image (if you didn't tag during build):

```bash
docker tag frontend-static:latest YOUR_DOCKERHUB_USERNAME/frontend-static:latest
```

3. Push the image:

```bash
docker push YOUR_DOCKERHUB_USERNAME/frontend-static:latest
```

> If you need to create the repository on Docker Hub, go to https://hub.docker.com and create a new repository named `frontend-static` (or whatever you prefer) before pushing.

---

## Task 6 — Set up a Kind Kubernetes cluster

If `kind` isn't installed, follow the official instructions (https://kind.sigs.k8s.io). On Linux/macOS you can typically install with `curl`:

```bash
# install kind (example)
curl -Lo ./kind https://kind.sigs.k8s.io/dl/v0.25.0/kind-$(uname)-amd64
chmod +x ./kind
mv ./kind /usr/local/bin/kind

# verify
kind --version
```

Create a simple cluster:

```bash
kind create cluster --name frontend-cluster
# this uses Docker to create a local k8s cluster
```

Confirm kubectl can talk to it:

```bash
kubectl cluster-info --context kind-frontend-cluster || kubectl get nodes
```

---

## Task 7 — Deploy to Kubernetes

Create `k8s/deployment.yaml` with the following content. Replace `YOUR_DOCKERHUB_USERNAME/frontend-static:latest` with your image.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend-deployment
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: nginx
        image: YOUR_DOCKERHUB_USERNAME/frontend-static:latest
        ports:
        - containerPort: 80
```

Apply the deployment:

```bash
kubectl apply -f k8s/deployment.yaml
kubectl rollout status deployment/frontend-deployment
kubectl get pods -l app=frontend
```

> If you are using an image from Docker Hub and your cluster nodes need access, it should work (Docker Hub public images are accessible). If you use a private repo, you'll need to set up an imagePullSecret.

---

## Task 8 — Create a Service (ClusterIP)

Create `k8s/service.yaml`:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-service
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http
```

Apply the service:

```bash
kubectl apply -f k8s/service.yaml
kubectl get svc frontend-service
```

---

## Task 9 — Access the application

Since the service is a `ClusterIP`, use `kubectl port-forward` to access it locally.

```bash
kubectl port-forward svc/frontend-service 8080:80
```

Open your browser and visit:

```
http://localhost:8080
```

You should see the static landing page served by the containers in your Kind cluster.

---

## Troubleshooting & tips

- **Image can't be pulled**: Verify the image name is correct and the tag exists on Docker Hub. If private, create an `imagePullSecret` and reference it in the deployment.

- **Pods crash immediately**: Check pod logs and describe the pod:

```bash
kubectl describe pod <pod-name>
kubectl logs <pod-name>
```

- **`kubectl port-forward` errors**: Ensure `kubectl` is using the correct context `kubectl config current-context` and the service exists.

- **Testing locally without Docker Hub**: You can `kind load docker-image YOUR_DOCKERHUB_USERNAME/frontend-static:latest --name frontend-cluster` to load a local image into the kind nodes and skip pushing to Docker Hub.

- **Scaling**: Change `replicas` in the Deployment and re-apply, or run `kubectl scale deployment frontend-deployment --replicas=4`.

- **Expose externally** (optional, not required here): Create a `NodePort` or `LoadBalancer` service type or use an ingress with an ingress controller.

---

## Cleanup

When you're done and want to remove everything locally:

```bash
kubectl delete -f k8s/service.yaml
kubectl delete -f k8s/deployment.yaml
kind delete cluster --name frontend-cluster
# optionally remove local docker image
docker rmi YOUR_DOCKERHUB_USERNAME/frontend-static:latest
```

---

## Final notes

- Replace every placeholder of the form `YOUR_DOCKERHUB_USERNAME` with your actual Docker Hub username or full image path.
- This README is intentionally explicit so you can follow step-by-step as a hands-on DevOps task. Extend it with CI/CD pipelines (GitHub Actions) to build/push images automatically and apply to the cluster for Phase 2.

---

Good luck — ping me if you want this extended into:
- A GitHub Actions CI to build/push the image and run `kubectl apply`.
- An Ingress + cert-manager for TLS.

