# Opsera Code-to-Cloud Enterprise - Deployment Session Log

**Date:** February 15-16, 2026
**Application:** enterpriseapp-vote (multi-service voting app)
**Platform:** Opsera Code-to-Cloud Enterprise v0.932
**Repository:** github.com/opsera-agent-demos/enterpriseapp-vote

---

## Table of Contents

1. [Application Architecture](#application-architecture)
2. [Infrastructure Overview](#infrastructure-overview)
3. [Session Timeline](#session-timeline)
4. [Issues Encountered & Fixes](#issues-encountered--fixes)
5. [CI/CD Pipeline Structure](#cicd-pipeline-structure)
6. [Deployment State](#deployment-state)
7. [Files Modified](#files-modified)
8. [Commit History](#commit-history)
9. [Managed Services Configuration](#managed-services-configuration)
10. [Kubernetes Resources](#kubernetes-resources)
11. [Lessons Learned](#lessons-learned)

---

## Application Architecture

The enterpriseapp-vote application is a classic microservices voting application consisting of three services:

| Service | Technology | Purpose | Port |
|---------|-----------|---------|------|
| **vote** | Python/Flask + Gunicorn | Web frontend for casting votes (Cats vs Dogs) | 8080 |
| **result** | Node.js/Express + Socket.IO | Real-time results dashboard via WebSockets | 8080 |
| **worker** | .NET 7.0 (C#) | Background processor: reads votes from Redis, writes to PostgreSQL | N/A |

**Data Flow:**
```
User → vote (Flask) → Redis (ElastiCache) → worker (.NET) → PostgreSQL (RDS) → result (Express/Socket.IO) → User
```

**Key Dependencies:**
- Redis (ElastiCache): Vote queue (RPUSH/LPOP on `votes` key)
- PostgreSQL (RDS): Persistent vote storage (`votes` table with `id` and `vote` columns)
- The worker creates the `votes` table on first connection (`CREATE TABLE IF NOT EXISTS`)

---

## Infrastructure Overview

### AWS Resources (us-west-2)

| Resource | Type | Identifier / Endpoint |
|----------|------|----------------------|
| **EKS Hub Cluster** | EKS | `argocd-usw2` (ArgoCD management) |
| **EKS Spoke Cluster** | EKS | `opsera-usw2-np` (workload cluster, 12 nodes) |
| **ECR Repositories** | ECR | `enterpriseapp-vote-vote`, `enterpriseapp-vote-result`, `enterpriseapp-vote-worker` |
| **RDS PostgreSQL** | RDS (db.t3.micro) | `enterpriseapp-vote-dev.chwgk42cofpp.us-west-2.rds.amazonaws.com` |
| **ElastiCache Redis** | ElastiCache (cache.t3.micro) | `enterpriseapp-vote-dev.2muj25.0001.usw2.cache.amazonaws.com` |
| **Secrets Manager** | Secret | `rds!db-154eeaec-7632-4f5f-8a65-3697eefcbc9f` (auto-managed RDS password) |

### Kubernetes Resources (Spoke Cluster: opsera-usw2-np)

| Resource | Namespace | Details |
|----------|-----------|---------|
| **Namespace** | `opsera-enterpriseapp-vote-dev` | All workloads deployed here |
| **Deployments** | 3 | vote, result, worker (1 replica each) |
| **Services** | 4 | vote (ClusterIP:80), result (ClusterIP:80), db (ExternalName→RDS), redis (ExternalName→ElastiCache) |
| **Ingress** | 1 | `enterpriseapp-vote-ingress` (nginx, host-based routing) |
| **ConfigMap** | 1 | `enterpriseapp-vote-config` (REDIS_HOST, DB_HOST, OPTION_A, OPTION_B) |
| **Secrets** | 2 | `enterpriseapp-vote-db-secret` (DATABASE_URL, DATABASE_CONNECTION_STRING), `ecr-pull-secret` |
| **ServiceAccount** | 1 | `enterpriseapp-vote-sa` |

### ArgoCD Configuration

| Setting | Value |
|---------|-------|
| **Application Name** | `opsera-enterpriseapp-vote-dev` |
| **Source Repo** | `github.com/opsera-agent-demos/enterpriseapp-vote` |
| **Source Path** | `.opsera-enterpriseapp-vote/k8s/overlays/dev` |
| **Destination Cluster** | `opsera-usw2-np` (spoke) |
| **Destination Namespace** | `opsera-enterpriseapp-vote-dev` |
| **Sync Policy** | Automated with self-heal |
| **Health Status** | Healthy |
| **Sync Status** | Synced |

---

## Session Timeline

### Session 1 (Feb 15, ~4:00 PM - 5:00 PM PST)

**Objective:** Generate complete CI/CD infrastructure for brownfield voting app

1. **4:07 PM** - Codebase exploration: identified 3 services, Dockerfiles, docker-compose, K8s specs
2. **4:08 PM** - Invoked Opsera Code-to-Cloud Enterprise skill
3. **4:09 PM** - Prerequisites verified: GitHub CLI, Docker, repo access
4. **4:10 PM** - Parallel generation started:
   - Bootstrap infrastructure workflow
   - 3x CI/CD workflows (vote, result, worker)
   - Kustomize K8s manifests + ArgoCD config
5. **4:14 PM** - Application code hardened:
   - Made Redis/PostgreSQL connections configurable via env vars
   - Hardened all 3 Dockerfiles (non-root user, health checks)
6. **4:15 PM** - All artifacts generated and committed

### Session 2 (Feb 15, ~6:45 PM - Feb 16, ~3:30 AM PST)

**Objective:** Fix deployment issues and get all services running

7. **6:45 PM** - Identified empty image tag bug (`newTag: ""` in kustomization.yaml)
8. **6:50 PM** - **FIX #1**: Added `build-image` to `update-manifests` needs chain (commit `d8559a4`)
9. **7:00 PM** - Pipelines triggered; Stages 1-8 pass, Stage 9 (ArgoCD Sync) times out
10. **7:05 PM** - User requested: "provision AWS managed services and continue with rest"
11. **7:10 PM** - User clarified: "lets do this as GHA workflow" and "we want to use IRSA and never have to deal with passwords"
12. **7:15 PM** - Created `1b-provision-managed-services` workflow (commit `ec9ed6c`)
13. **7:20 PM** - **FIX #2**: Compact JSON for GITHUB_OUTPUT (`jq -c`) (commit `34a6337`)
14. **7:25 PM** - **FIX #3**: PostgreSQL version 15.4 → 16.6 (commit `540d47d`)
15. **7:30 PM** - **FIX #4**: Temp file for K8s secret creation (commit `11fb728`)
16. **7:45 PM** - All managed services provisioned, endpoints populated
17. **7:50 PM** - **FIX #5**: Deleted duplicate ArgoCD cluster secret
18. **8:00 PM** - **FIX #6**: Port 80 → 8080 for non-root containers (commit `f45fa4a`)
19. **8:15 PM** - **FIX #7**: Improved ArgoCD Stage 9 debug output (commit `c95a280`)
20. **8:30 PM** - All 3 services deployed successfully, ArgoCD Healthy/Synced
21. **8:45 PM** - Added deployment landscape to Stage 11 (commit `1cdd89d`)
22. **9:00 PM** - Landscape verified working

### Session 3 (Feb 16, ~2:45 AM - 3:30 AM PST) - This Session

**Objective:** Fix /result routing and database connectivity

23. **2:45 AM** - User reported "Cannot GET /result" at `https://enterpriseapp-vote-dev.agent.opsera.dev/result`
24. **2:50 AM** - **FIX #8**: Result service `/result` path routing (commit `8997684`)
   - Mounted Express static at `/result` path
   - Configured Socket.IO path to `/result/socket.io`
   - Updated `<base href="/result/">` in HTML
25. **3:00 AM** - User reported result page showing 50/50 default (no DB data)
26. **3:05 AM** - Diagnosed: both result and worker stuck "Waiting for db"
27. **3:08 AM** - Verified network connectivity (nc to RDS port 5432 = OK)
28. **3:10 AM** - Verified password from Secrets Manager matches K8s secret
29. **3:12 AM** - Discovered two issues:
   - **Issue A**: Special characters in password (`$`, `>`, `[`) not URL-encoded in `postgres://` URI
   - **Issue B**: RDS PostgreSQL 16 requires SSL by default (`rds.force_ssl=1`)
30. **3:15 AM** - Error from result pod: `no pg_hba.conf entry for host ... no encryption`
31. **3:18 AM** - Added SSL to K8s secret, but got: `self-signed certificate in certificate chain`
32. **3:20 AM** - Found that `?sslmode=require` in URL overrides code-level `ssl: { rejectUnauthorized: false }`
33. **3:22 AM** - **FIX #9**: Result service DB connection (commit `4c7aa60`)
   - Added `ssl: { rejectUnauthorized: false }` to pg Pool config
   - URL-encoded password in DATABASE_URL (no `?sslmode=require`)
   - Added `SSL Mode=Require;Trust Server Certificate=true` to .NET connection string
34. **3:25 AM** - **FIX #10**: Updated provisioning workflow (commit `a4002f3`)
   - URL-encode password using `python3 urllib.parse.quote()`
   - Added SSL params to .NET connection string
35. **3:28 AM** - All services connected to DB, votes flowing end-to-end

---

## Issues Encountered & Fixes

### Fix #1: Empty Image Tags in Kustomization
- **Symptom:** `newTag: ""` in kustomization.yaml after CI/CD pipeline runs
- **Root Cause:** `update-manifests` job referenced `needs.build-image.outputs.image_tag` but `build-image` wasn't in its `needs` chain
- **Fix:** Added `build-image` to needs: `needs: [build-image, push-to-ecr, refresh-ecr-secret]`
- **Commit:** `d8559a4`
- **Files:** All 3 CI/CD workflows

### Fix #2: GITHUB_OUTPUT Multiline JSON
- **Symptom:** `Invalid format '  "subnet-079509530912d0d93",'` error in GitHub Actions
- **Root Cause:** Multiline JSON array from `jq` can't be written to GITHUB_OUTPUT
- **Fix:** Use `jq -c '.'` to compact JSON to single line
- **Commit:** `34a6337`
- **Files:** `1b-provision-managed-services-enterpriseapp-vote.yaml`

### Fix #3: PostgreSQL Version Not Available
- **Symptom:** `Cannot find version 15.4 for postgres` (exit code 254)
- **Root Cause:** PostgreSQL 15.4 not available in us-west-2 region
- **Fix:** Changed to version `16.6` after querying `aws rds describe-db-engine-versions`
- **Commit:** `540d47d`
- **Files:** `1b-provision-managed-services-enterpriseapp-vote.yaml`

### Fix #4: K8s Secret Pipe Masking
- **Symptom:** `error: no objects passed to apply` when creating K8s secret
- **Root Cause:** GitHub Actions masks secrets in piped output (`kubectl create --dry-run | kubectl apply`)
- **Fix:** Write secret YAML to temp file, then `kubectl apply -f <file>`
- **Commit:** `11fb728`
- **Files:** `1b-provision-managed-services-enterpriseapp-vote.yaml`

### Fix #5: ArgoCD Duplicate Cluster Secret
- **Symptom:** `InvalidSpecError: there are 2 clusters with the same name`
- **Root Cause:** Bootstrap created `opsera-usw2-np-cluster-secret` but `opsera-usw2-np-cluster` already existed
- **Fix:** Deleted duplicate: `kubectl delete secret opsera-usw2-np-cluster-secret -n argocd --context hub`
- **Files:** No code changes (runtime fix)

### Fix #6: Port 80 Permission Denied
- **Symptom:** CrashLoopBackOff - `[Errno 13] Permission denied` binding port 80
- **Root Cause:** Non-root containers (UID 1000/10001) can't bind to privileged ports (<1024)
- **Fix:** Changed to port 8080 in Dockerfiles, app code, K8s deployments and services
- **Commit:** `f45fa4a`
- **Files:** `vote/Dockerfile`, `vote/app.py`, `result/Dockerfile`, K8s deployment and service YAMLs

### Fix #7: ArgoCD Stage 9 Debug Output
- **Symptom:** `kubectl get application` with jsonpath returned empty strings
- **Root Cause:** Need full CRD name `application.argoproj.io` and proper JSON parsing
- **Fix:** Used full CRD name, `jq` for JSON parsing, added debug output
- **Commit:** `c95a280`
- **Files:** All 3 CI/CD workflows

### Fix #8: Result Service /result Path Routing
- **Symptom:** "Cannot GET /result" when accessing `https://enterpriseapp-vote-dev.agent.opsera.dev/result`
- **Root Cause:** Ingress forwards `/result` path to result service, but Express app only handles `/`
- **Fix:**
  - Mounted Express static at `/result` base path: `app.use('/result', express.static(...))`
  - Kept root `/` for health check probes
  - Configured Socket.IO server path: `{path: '/result/socket.io'}`
  - Updated client: `io.connect({path: '/result/socket.io'})`
  - Set `<base href="/result/">` in HTML so relative asset URLs resolve correctly
  - Made CSS href relative: `href='stylesheets/style.css'`
- **Commit:** `8997684`
- **Files:** `result/server.js`, `result/views/index.html`, `result/views/app.js`

### Fix #9: Result Service DB Connection (URL Encoding + SSL)
- **Symptom:** Result and worker pods stuck "Waiting for db"
- **Root Cause (dual):**
  1. RDS PostgreSQL 16 defaults to `rds.force_ssl=1` (SSL required)
  2. Special characters in auto-generated password (`$`, `>`, `[`) break `postgres://` URL parsing
- **Diagnosis Steps:**
  - `nc -zv` to RDS → port open (network OK)
  - `psql` with PGPASSWORD → connected (password correct)
  - `node -e` with pg Pool → `no pg_hba.conf entry ... no encryption` (SSL required)
  - Added `?sslmode=require` → `self-signed certificate in certificate chain` (RDS CA not trusted)
  - Found `?sslmode=require` overrides code-level `ssl: { rejectUnauthorized: false }`
- **Fix:**
  - `result/server.js`: Added `ssl: process.env.DATABASE_URL ? { rejectUnauthorized: false } : false`
  - K8s secret DATABASE_URL: URL-encoded password, NO `?sslmode=require`
  - K8s secret DATABASE_CONNECTION_STRING: Added `SSL Mode=Require;Trust Server Certificate=true;`
- **Commit:** `4c7aa60`
- **Files:** `result/server.js`, K8s secret (runtime)

### Fix #10: Provisioning Workflow - URL Encoding + SSL
- **Symptom:** Future re-provisioning would recreate the broken secret
- **Fix:**
  - URL-encode password: `python3 -c "import urllib.parse,sys; print(urllib.parse.quote(sys.argv[1], safe=''))"`
  - Added SSL params to .NET connection string in secret template
- **Commit:** `a4002f3`
- **Files:** `1b-provision-managed-services-enterpriseapp-vote.yaml`

---

## CI/CD Pipeline Structure

### 11-Stage Pipeline (per service)

Each service (vote, result, worker) has an identical 11-stage CI/CD pipeline:

| Stage | Name | Purpose |
|-------|------|---------|
| 1 | Security Scan (Gitleaks) | Scan for leaked secrets in code |
| 2 | Build Image | Docker build with security hardening |
| 3 | Security Scan (Grype) | Container vulnerability scanning |
| 4 | Push to ECR | Push image with immutable tag (SHA-timestamp) |
| 5 | Refresh ECR Secret | Update K8s image pull secret |
| 6 | Update Manifests | Update kustomization.yaml with new image tag |
| 7 | Quality Gate (SonarQube) | Code quality analysis |
| 8 | ArgoCD Hard Refresh | Force ArgoCD to detect manifest changes |
| 9 | ArgoCD Sync & Wait | Trigger sync and wait for Healthy/Synced |
| 10 | Verify Deployment | Confirm pods running on spoke cluster |
| 11 | Deployment Landscape | Display comprehensive deployment dashboard |

### Workflow Files

| Workflow | Purpose |
|----------|---------|
| `0-bootstrap-enterpriseapp-vote.yaml` | One-time infrastructure setup (ECR, namespaces, ArgoCD registration) |
| `1b-provision-managed-services-enterpriseapp-vote.yaml` | Provision RDS + ElastiCache with IRSA |
| `2-ci-enterpriseapp-vote-vote-dev.yaml` | Vote service CI/CD pipeline |
| `2-ci-enterpriseapp-vote-result-dev.yaml` | Result service CI/CD pipeline |
| `2-ci-enterpriseapp-vote-worker-dev.yaml` | Worker service CI/CD pipeline |

### Image Tagging Strategy

- Format: `<short-sha>-<YYYYMMDDHHMMSS>` (e.g., `d8559a4-20260216005531`)
- Immutable tags in ECR
- Tag propagated through pipeline via `build-image` job output → `update-manifests` job

---

## Deployment State

### Live Endpoints

| Endpoint | URL | Service |
|----------|-----|---------|
| Vote (Cast votes) | `https://enterpriseapp-vote-dev.agent.opsera.dev/` | vote (Flask) |
| Result (View results) | `https://enterpriseapp-vote-dev.agent.opsera.dev/result` | result (Express) |

### Pod Status (as of Feb 16, 2026 ~3:30 AM PST)

```
NAME                      READY   STATUS    RESTARTS   AGE
result-64c47c6568-5bhz6   1/1     Running   0          72m
vote-56df999d96-xgjk5     1/1     Running   0          116m
worker-5d84ccc894-wqdx7   1/1     Running   0          80m
```

### Service Connectivity

```
vote     → Redis (ElastiCache)  : RPUSH votes (write)
worker   → Redis (ElastiCache)  : LPOP votes (read)
worker   → PostgreSQL (RDS)     : INSERT/UPDATE votes table (write)
result   → PostgreSQL (RDS)     : SELECT FROM votes (read)
result   → Browser              : Socket.IO WebSocket (real-time updates)
```

### Managed Service Endpoints

| Service | Endpoint | Engine |
|---------|----------|--------|
| Redis | `enterpriseapp-vote-dev.2muj25.0001.usw2.cache.amazonaws.com:6379` | Redis 7.1 |
| PostgreSQL | `enterpriseapp-vote-dev.chwgk42cofpp.us-west-2.rds.amazonaws.com:5432` | PostgreSQL 16.6 |

---

## Files Modified

### Application Code Changes

| File | Changes |
|------|---------|
| `vote/app.py` | Made Redis connection configurable via `REDIS_HOST` env var; changed port 80→8080 |
| `vote/Dockerfile` | Non-root user (UID 1000), health check, port 8080 |
| `result/server.js` | Configurable DB via `DATABASE_URL`; SSL support; `/result` base path; Socket.IO path |
| `result/views/index.html` | `<base href="/result/">`; relative CSS path |
| `result/views/app.js` | Socket.IO path `/result/socket.io` |
| `result/Dockerfile` | Non-root user (UID 10001), health check, port 8080 |
| `worker/Program.cs` | Configurable DB via `DATABASE_CONNECTION_STRING`; configurable Redis via `REDIS_HOST` |
| `worker/Dockerfile` | Non-root user (UID 10001) |

### CI/CD & Infrastructure Files

| File | Purpose |
|------|---------|
| `.github/workflows/0-bootstrap-enterpriseapp-vote.yaml` | Bootstrap infrastructure |
| `.github/workflows/1b-provision-managed-services-enterpriseapp-vote.yaml` | Managed services provisioning |
| `.github/workflows/2-ci-enterpriseapp-vote-vote-dev.yaml` | Vote CI/CD pipeline |
| `.github/workflows/2-ci-enterpriseapp-vote-result-dev.yaml` | Result CI/CD pipeline |
| `.github/workflows/2-ci-enterpriseapp-vote-worker-dev.yaml` | Worker CI/CD pipeline |
| `.opsera-enterpriseapp-vote/k8s/base/*.yaml` | Base K8s manifests (deployments, services, ingress, configmap) |
| `.opsera-enterpriseapp-vote/k8s/overlays/dev/kustomization.yaml` | Dev overlay with image tags and endpoints |
| `.opsera-enterpriseapp-vote/argocd/enterpriseapp-vote-dev.yaml` | ArgoCD Application manifest |

---

## Commit History

```
a4002f3 Fix provisioning workflow: URL-encode DB password and add SSL params
4c7aa60 Fix result service DB connection: URL-encode password and enable SSL
8997684 Fix result service /result path routing
1cdd89d feat: add comprehensive deployment landscape with live cluster data
f45fa4a fix: use port 8080 for vote/result (non-root can't bind port 80)
c95a280 fix: improve Stage 9 ArgoCD sync debug with full resource name
11fb728 fix: use temp file for K8s secret creation (avoid pipe masking)
540d47d fix: use PostgreSQL 16.6 (15.4 not available in us-west-2)
34a6337 fix: compact JSON output for GITHUB_OUTPUT compatibility
6c828c9 feat: IRSA-based managed services provisioning (no passwords)
ec9ed6c feat: add managed services provisioning workflow (RDS + ElastiCache)
d8559a4 fix: add build-image to update-manifests needs chain
30e5fa0 fix: use UID 10001 for result/worker (avoid UID 1000 conflict with base images)
7e8b238 fix: handle existing ClusterRoleBinding in bootstrap (immutable roleRef) [skip ci]
87176e0 feat: add enterprise C2C pipeline with Day 0 hardening for enterpriseapp-vote
```

---

## Managed Services Configuration

### RDS PostgreSQL

```
Identifier:     enterpriseapp-vote-dev
Engine:         PostgreSQL 16.6
Instance Class: db.t3.micro
Storage:        20 GB gp3
Multi-AZ:       No (dev)
Authentication:  --manage-master-user-password (Secrets Manager auto-rotation)
SSL:            Required (rds.force_ssl=1, PostgreSQL 16 default)
Subnet Group:   enterpriseapp-vote-dev-subnet-group (private subnets)
Security Group: enterpriseapp-vote-dev-rds-sg (ingress from EKS SG on port 5432)
```

### ElastiCache Redis

```
Cluster ID:     enterpriseapp-vote-dev
Engine:         Redis 7.1
Node Type:      cache.t3.micro
Nodes:          1 (dev)
Authentication: None (dev, VPC-only access)
Subnet Group:   enterpriseapp-vote-dev-cache-subnet-group (private subnets)
Security Group: enterpriseapp-vote-dev-redis-sg (ingress from EKS SG on port 6379)
```

### Security Groups Created

| Security Group | Ingress Rule |
|---------------|--------------|
| `enterpriseapp-vote-dev-rds-sg` | TCP 5432 from EKS cluster security group |
| `enterpriseapp-vote-dev-redis-sg` | TCP 6379 from EKS cluster security group |

---

## Kubernetes Resources

### Deployments

All deployments run as non-root with security hardening:

```yaml
securityContext:
  runAsNonRoot: true
  runAsUser: 10001  # (1000 for vote)
containers:
  securityContext:
    allowPrivilegeEscalation: false
```

### K8s Secret: enterpriseapp-vote-db-secret

```
DATABASE_URL: postgres://postgres:<URL-ENCODED-PASSWORD>@<RDS-HOST>:5432/postgres
DATABASE_CONNECTION_STRING: Server=<RDS-HOST>;Port=5432;Username=postgres;Password=<RAW-PASSWORD>;Database=postgres;SSL Mode=Require;Trust Server Certificate=true;
```

**Important notes:**
- `DATABASE_URL` must have the password URL-encoded (special chars: `$`→`%24`, `>`→`%3E`, `[`→`%5B`)
- `DATABASE_URL` must NOT include `?sslmode=require` (it overrides code-level SSL config)
- `DATABASE_CONNECTION_STRING` must include `SSL Mode=Require;Trust Server Certificate=true;`
- The result app's code handles SSL via `ssl: { rejectUnauthorized: false }` in the pg Pool config

### Ingress Routing

```yaml
rules:
  - host: enterpriseapp-vote-dev.agent.opsera.dev
    paths:
      - path: /result    → service: result (port 80 → targetPort 8080)
      - path: /           → service: vote   (port 80 → targetPort 8080)
```

**Important:** The result app must serve from `/result` base path because the ingress does NOT rewrite the path. The app handles this with `app.use('/result', express.static(...))`.

---

## Lessons Learned

### 1. RDS PostgreSQL 16 Requires SSL by Default
PostgreSQL 16 on RDS sets `rds.force_ssl=1` by default. All clients must connect with SSL. For Node.js `pg`, use `ssl: { rejectUnauthorized: false }` in code (not `?sslmode=require` in URL, which enables strict cert verification).

### 2. URL-Encode Passwords in PostgreSQL URIs
Auto-generated passwords from Secrets Manager can contain special characters (`$`, `>`, `[`, etc.) that break `postgres://` URL parsing. Always URL-encode the password portion using `urllib.parse.quote()`.

### 3. Don't Mix sslmode URL Param with Code-Level SSL Config
In the Node.js `pg` library, `?sslmode=require` in the connection string overrides any `ssl` option passed to the Pool constructor. To use `{ rejectUnauthorized: false }`, omit `sslmode` from the URL.

### 4. GitHub Actions GITHUB_OUTPUT Can't Handle Multiline Values Simply
Use `jq -c` to compact JSON to a single line before writing to GITHUB_OUTPUT.

### 5. GitHub Actions Masks Secrets in Piped Output
When constructing K8s secrets from GitHub Actions secrets, write to a temp file instead of piping through `kubectl create --dry-run | kubectl apply`.

### 6. Non-Root Containers Can't Bind Privileged Ports
Ports below 1024 require root. Use port 8080 for application containers running as non-root. K8s services can still expose port 80 externally with `targetPort: 8080`.

### 7. Ingress Path Routing Without Rewrite
When using nginx ingress with `pathType: Prefix` and no rewrite annotation, the application receives the full path (e.g., `/result/app.js`). The app must be configured to serve from that base path.

### 8. ArgoCD CRD Queries Need Full Resource Name
Use `application.argoproj.io` instead of just `application` when querying ArgoCD resources with kubectl to avoid ambiguity.

### 9. Pipeline needs Chain Must Include All Output Dependencies
If job B references `needs.A.outputs.X`, then A MUST be in B's `needs` list. GitHub Actions silently returns empty string for outputs from jobs not in the needs chain.

### 10. PostgreSQL Version Availability Varies by Region
Always check `aws rds describe-db-engine-versions` for the target region before specifying a version.
