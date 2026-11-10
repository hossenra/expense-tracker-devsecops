# Expense Tracker on Kubernetes

The same app as `docker-compose.prod.yml`, running on Kubernetes (KIND).
The images are the ones the GitHub Actions pipeline already builds, scans with Trivy and pushes to Docker Hub.

## Architecture

```
Browser -> :80 -> NodePort 30080 -> Service frontend -> 2x nginx pods
                                         | /api/
                                         v
                                 Service backend:5000 -> 2x Node.js pods
                                         |
                                         v
                                 Service mongo:27017 -> MongoDB StatefulSet + PVC
```

| docker-compose (prod) | Kubernetes |
|---|---|
| `mongo` service + `mongo-data` volume | `StatefulSet` + headless `Service` + PVC (`volumeClaimTemplates`) |
| `backend` service, `environment:` | `Deployment` (2 replicas) + `ClusterIP Service` + `ConfigMap` |
| `frontend` service, `ports: 80:80` | `Deployment` (2 replicas) + `NodePort Service` |
| `restart: unless-stopped` | Deployments/StatefulSet self-heal + liveness probes |
| `depends_on` | readiness probes (a pod only gets traffic when it is ready) |
| `expense-net` network | namespace `expense` + Kubernetes DNS (service names) |

Service names `mongo` and `backend` match docker-compose, so the app code and nginx config work unchanged.

## Files

| File | What it creates |
|---|---|
| `namespace.yaml` | Namespace `expense` |
| `mongo.yaml` | MongoDB StatefulSet, headless Service, 1Gi volume |
| `backend.yaml` | ConfigMap, Service `backend:5000`, Deployment (probes on `/api/health`) |
| `frontend.yaml` | Service (NodePort 30080), Deployment |
| `kustomization.yaml` | Applies everything; sets image names and tags |
| `kind-config.yaml` | 2-node KIND cluster; host port 80 -> NodePort 30080 |

## Run it on EC2

Use **t3.medium (4 GB)** and a **20 GB disk**. Security group inbound: 22 (SSH), 80 (HTTP).

1. Install Docker, KIND and kubectl, and raise the inotify limits:

```bash
echo -e "fs.inotify.max_user_watches=524288\nfs.inotify.max_user_instances=512" | sudo tee /etc/sysctl.d/99-kind.conf
sudo sysctl --system
```

2. Clone the repo and create the cluster:

```bash
git clone https://github.com/hossenra/expense-tracker-devsecops.git
cd expense-tracker-devsecops
kind create cluster --config=k8s/kind-config.yaml --name=expense
kubectl get nodes
```

3. Set the images in `k8s/kustomization.yaml`: replace `DOCKERHUB_USERNAME` with your Docker Hub username and `COMMIT_SHA` with a tag that exists on Docker Hub (the commit SHA pushed by the pipeline).

4. Deploy:

```bash
kubectl apply -k k8s/
kubectl get pods -n expense -w
```

5. Test:

```bash
curl -f http://localhost/api/health
```

Open `http://<EC2-public-IP>` in a browser.

## Demo commands

```bash
kubectl get all -n expense
kubectl get pvc -n expense

# Self-healing
kubectl delete pod -n expense -l app=backend --wait=false
kubectl get pods -n expense -w

# Scaling
kubectl scale deployment backend -n expense --replicas=3

# Rolling update to a new image version, and rollback
kubectl set image deployment/backend backend=DOCKERHUB_USERNAME/expense-backend:NEW_SHA -n expense
kubectl rollout status deployment/backend -n expense
kubectl rollout undo deployment/backend -n expense

# Logs
kubectl logs -n expense deploy/backend --tail=50
```

## Clean up

```bash
kind delete cluster --name expense
```

## Possible next steps

- CD to Kubernetes: after the image scan + push, run `kubectl set image ... :${{ github.sha }}` from the pipeline.
- Ingress controller + TLS instead of NodePort.
- MongoDB authentication via a Kubernetes `Secret`.
- Monitoring with Prometheus + Grafana.
- Infrastructure as Code: Terraform for the EC2 instance, Ansible for Docker/KIND installation.
