Git Source Code Repo – Holds all application code, infrastructure‑as‑code (Helm/K8s manifests), and pipeline definitions.

Trigger a pipeline – A webhook or manual action starts the CI/CD process (e.g., Jenkins, GitLab CI, ArgoCD).

Build artifact and push image – Compiles the code, runs tests, builds a Docker image, and pushes it to the Docker Registry (private).

Docker Registry – Securely stores all container images; used by Kubernetes to pull images during deployment.

Pull docker image – Worker nodes fetch the new image from the registry when a deployment is updated.

Deploy (kubectl apply) – Applies the new Kubernetes manifests to the cluster(s).
➡ Why: Automates builds and deployments, ensures traceability, and keeps images secure.