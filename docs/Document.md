# ShopFlow DevOps Capstone

> Draft README. Replace every `TODO` with your own details as you go. Sections for later phases are stubs so the structure matches the brief.

## 1. Project Overview and Problem Statement

ShopFlow is a small e-commerce platform (Nginx frontend, Flask API, PostgreSQL, Redis). This project takes it through a full DevOps lifecycle: source control, containerization, CI/CD, Kubernetes, infrastructure automation, observability, security, reliability and documentation.

- **Problem:** AI-TODO (one paragraph: what was wrong or missing before: no pipeline, no repeatable deploys, no monitoring...)
- **Track:**  B (AWS EC2 + Minikube): TODO
- **Team:** Gerald
- **Repository:** https://github.com/Gerald-Izuchukwu/Helix-Capstone.git

## 2. Architecture

![Architecture diagram](architecture-diagram.png)

TODO: short description of request flow (Browser → Nginx → Flask API → PostgreSQL / Redis).

## 3. Technology Choices and Justification

| Area | Tool | Why |
| --- | --- | --- |
| Source control | Git + GitHub | TODO |
| Containers | Docker, Docker Compose | TODO |
| CI/CD | GitHub Actions | TODO |
| Orchestration | Kubernetes (Minikube) | TODO |
| IaC / Config | Terraform, Ansible | TODO |
| Monitoring | Prometheus, Grafana | TODO |
| Logging | TODO (Loki?) | TODO |
| Security | TODO (Gitleaks, Trivy, ...) | TODO |
| GitOps | ArgoCD | TODO |

## 4. Local Deployment (Docker Compose), Phase 1

```bash
git clone https://github.com/Gerald-Izuchukwu/Helix-Capstone.git
cd Helix-Capstone
git checkout -b <your-project-branch>
docker compose up --build
docker compose ps
curl http://localhost:5000/health
curl http://localhost:5000/metrics
```

**What I inspected before running:** TODO
- Backend, 
- Frontend, 
- Dockerfiles, 
- Compose file

**Verification:** 
- Frontend URL: Verified, 
- `/health` response: Verified, 
- `/metrics` sample: Verified

### Problems encountered and how I fixed them

| # | Problem (exact error) | Cause | Fix |
| --- | --- | --- | --- |
| 1 | TODO | TODO | TODO |
| 2 | TODO | TODO | TODO |

## 5. Git and GitHub Workflow, Phase 2

**Branching strategy** (required in README):

- `main`: always deployable, protected, changes only via pull request
- `feature branches`: one branch per significant change

**Commit message convention:** `Type: short summary`

**Pull requests:** [PR 1](https://github.com/Gerald-Izuchukwu/Helix-Capstone/pull/1), [PR 2](https://github.com/Gerald-Izuchukwu/Helix-Capstone/pull/2)

**Merge conflict exercise:** AI-TODO (which file, how it was created, how it was resolved)

**Branch protection:** TODO (rules enabled, or note if the plan did not permit it)

**Evidence:** repository URL TODO, commit history TODO, PR screenshots in `docs/screenshots/phase2/`

## 6. Docker Engineering, Phase 3

**Improvements made to the supplied containerization:**

| Area | Before | After | Reason |
| --- | --- | --- | --- |
| Image tags | `latest` | `1.0.0` | TODO |
| User | root | non-root | TODO |
| Config | hard-coded | environment variables | TODO |
| Health checks | none | added curl | TODO |
| Image size | 0 MB | 113 MB | TODO (base image, multi-stage?) |
| `.dockerignore` | none | added | TODO |

**Image list:** TODO (`docker images` output)

**Compose test:** TODO (commands and results)

## 7. CI/CD, Phase 4

**Pipeline stages:** checkout → install deps → lint → test → build images → dependency/source scan → image scan → (publish)

| Stage | Tool | 
| --- | --- | 
| Lint | Flake8/Hadolint |
| Tests | pytest | 
| Dependency scan | Trivy | 
| Secret scan | Trivy | 
| Image scan | Trivy | 
| Publish (extension) | Docker Hub / ECR | 

**Proof that gates fail:** [Failed Run](https://github.com/Gerald-Izuchukwu/Helix-Capstone/actions/runs/39109006042/job/117620951307)

**Workflow file:** `.github/workflows/ci.yaml`

## 8. Kubernetes Architecture, Phase 5

ShopFlow runs on Minikube in a dedicated `shopflow` namespace. Manifests live in `k8s/` and are numbered so they apply in dependency order.
 
```bash
minikube start --cpus=4 --memory=6144 --driver=docker
minikube addons enable metrics-server
kubectl apply -f k8s/
kubectl get all,pvc,configmap,secret,hpa,pdb -n shopflow
minikube service frontend -n shopflow --url
```
 
### Manifests
 
| File | Resources | Notes |
| --- | --- | --- |
| `00-namespace.yaml` | Namespace `shopflow` | |
| `01-configmap.yaml` | ConfigMap `shopflow-config` | Non-sensitive settings only (hosts, ports, DB name, Redis URL) |
| `examples/02-secret.example.yaml` | Secret template | Placeholders only; the real Secret is created in the cluster |
| `03-postgres-pvc.yaml` | PVC `postgres-pvc` (1Gi) | Dynamically provisioned by Minikube's `standard` StorageClass |
| `04-postgres.yaml` | Deployment + Service `postgres` | `postgres:17-alpine`, `Recreate` strategy, probes using `pg_isready` |
| `05-redis.yaml` | Deployment + Service `redis` | `redis:7.4-alpine`, no volume (cache only) |
| `06-backend.yaml` | Deployment + Service `backend` | 2 replicas, image `gerald22/shopflow_be:1.1`, probes on `/health` |
| `07-frontend.yaml` | Deployment + Service `frontend` | 2 replicas, image `gerald22/shopflow_fe:1.0`, NodePort |
| `08-backend-hpa.yaml` | HorizontalPodAutoscaler | 2 to 5 replicas at 70% CPU |
| `09-backend-pdb.yaml` | PodDisruptionBudget | `minAvailable: 1` |
 
### Design decisions
 
- **Image workflow:** images are pulled from public Docker Hub repositories (`gerald22/shopflow_be`, `gerald22/shopflow_fe`) published by the Phase 4 pipeline, so nothing is built inside Minikube.
- **Only the frontend is exposed.** Postgres, Redis and the backend are `ClusterIP` Services reachable only inside the cluster; the frontend is `NodePort`.
- **Postgres has persistent storage; Redis does not.** Postgres is the system of record, so it uses a PVC. Redis is a disposable cache and rebuilds itself.
- **Postgres uses the `Recreate` strategy** so two pods never contend for the same `ReadWriteOnce` volume. `PGDATA` points to a subdirectory to avoid the `lost+found` startup error.
- **Pinned image tags** (no `latest`) for Postgres and Redis in the cluster.
- **Probes and resources:** every workload has readiness and liveness probes and CPU/memory requests and limits. The HPA depends on the backend CPU request.
- **Labels:** each workload carries an `app` label that its Service and PDB select on.
### Secrets handling
 
No real credentials are committed to Git. The Secret is created directly in the cluster:
 
```bash
kubectl create secret generic shopflow-secret \
  --namespace shopflow \
  --from-literal=POSTGRES_USER=<user> \
  --from-literal=POSTGRES_PASSWORD='<password>'
```
 
The backend's `DATABASE_URL` is assembled at runtime from the Secret values using Kubernetes variable expansion, so the password never appears in the ConfigMap. A `secret.example.yaml` with placeholders documents the expected keys, and real secret files are git-ignored.
 
### Autoscaling and disruption budget evidence
 
- A load generator against `/health` pushed backend CPU to 170% of the request, and the HPA scaled the backend from 2 to 4 and then 5 replicas (the configured maximum) within about 20 seconds of crossing the threshold. CPU later reached 381% at the maximum.
- `kubectl describe pdb backend-pdb` showed `Min available: 1` with 4 allowed disruptions while 5 replicas were running. With the normal 2 replicas, 1 disruption is allowed, so at least one backend pod always stays up during a node drain.
- Screenshots: `docs/screenshots/phase5/`
### Problems encountered and fixes
 
| Problem | Cause | Fix |
| --- | --- | --- |
| Backend in `CrashLoopBackOff` with `password authentication failed` | The backend reads one `DATABASE_URL` that still held the Docker Compose credentials, while Postgres was initialized from the Kubernetes Secret | Removed `DATABASE_URL` from the ConfigMap and built it from the Secret values in the Deployment |
| Credentials in plaintext in the ConfigMap | `DATABASE_URL` contained the password | Same fix; recorded as a security finding for Phase 11 |
| Redis would have failed to connect | `REDIS_URL` used port 6378, but Redis listens on 6379 | Corrected the port in the ConfigMap |
| Empty product list in the UI | The database was recreated and nothing seeds products automatically | Expected behavior; products are added through the API |

## 9. AWS EC2 Deployment (Terraform + Ansible), Phases 6 to 8

TODO

## 10. Monitoring and Logging, Phases 9 and 10

TODO

## 11. Security Controls and Findings, Phase 11

TODO (at least two findings: what, risk, how addressed)

## 12. GitOps Workflow (ArgoCD), Phase 12

TODO

## 13. Deployment Strategy and Rollback Procedure, Phase 13

TODO

## 14. SLIs, SLOs and Error Budget, Phase 14

TODO

## 15. Incident Response / Runbook

TODO

## 16. Failure-Testing Evidence, Phase 15

TODO (symptom → detection → investigation → root cause → mitigation → recovery → prevention, per scenario)

## 17. Cost Analysis, Phase 16

TODO

## 18. Lessons Learned

TODO
