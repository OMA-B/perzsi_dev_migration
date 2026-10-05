# Perzsi Development Server Migration

## Migration Audit & Preliminary Findings

**Current Server:** `srv008.perzsi.com`
**Current Provider:** DigitalOcean
**Target Provider:** DatabaseMart VPS
**Environment:** Development / Staging
**Status:** Infrastructure audit completed; migration preparation in progress.

### 1. Executive Summary

I began auditing the existing Perzsi development server in preparation for migrating it from DigitalOcean to the DatabaseMart VPS.

The initial assessment shows that the server is hosting **multiple independent Perzsi applications and supporting services**, rather than a single application. The environment also relies heavily on Docker, Docker Swarm, local Docker Registry, GitHub Actions self-hosted runners, Docker secrets, persistent volumes, custom networks, and several additional standalone services.

Because much of the existing infrastructure was originally configured by the previous developer, I am treating the migration as an **infrastructure reconstruction and validation exercise**, rather than simply copying application files.

The goal is to reproduce the existing development environment on DatabaseMart with minimal behavioral changes, verify all services, and only then decommission the DigitalOcean development server.

---

# 2. Current Server Infrastructure

The current server is:

* **Hostname:** `srv008.perzsi.com`
* **OS:** Ubuntu 20.04.6 LTS
* **CPU:** 2 vCPU
* **RAM:** 3.8 GB
* **Root disk:** 78 GB, approximately 55 GB used
* **Additional attached volume:** 100 GB, approximately 8.6 GB used
* **Docker:** 24.0.7
* **Docker Compose:** v2.18.1
* **Docker Swarm:** Enabled

The server is therefore not a simple Docker Compose host; it is operating as a **Docker Swarm environment**.



---

# 3. Applications Currently Running

The audit identified several active Perzsi stacks.

### Core Perzsi backend

The main backend consists of:

* Django API
* Authentication server
* Redis
* Celery/worker infrastructure
* Flower
* Nginx/reverse proxy
* Admin application load-balancing/proxy component

The backend is deployed as the `perzsibackend` Docker Swarm stack.

### Other Perzsi applications

The server also currently runs:

1. **Perzsi Admin App**
2. **Perzsi Analytics v2**
3. **Perzsi Dashboard**
4. **Perzsi Frontend v3**
5. **Perzsi Product Feed API**
6. **PTAE**
7. **PTIE**
8. Local Docker Registry

The currently deployed Swarm stacks include:

* `perzsibackend` — 6 services
* `perzsi-adminapp` — 1 service
* `perzsianalytics-v2` — 1 service
* `perzsidashboard` — 1 service
* `perzsifrontendv3` — 1 service



There are also additional standalone Docker Compose applications such as Product Feed, PTAE and PTIE that will need to be assessed separately rather than assuming that the five Swarm stacks represent the entire server.

---

# 4. Deployment Architecture

One of the most important findings is that deployment is tightly coupled to **GitHub Actions self-hosted runners**.

The development workflows generally follow this pattern:

```text
GitHub Repository
       ↓
Push to dev branch
       ↓
GitHub Actions
       ↓
Self-hosted runner on srv008
       ↓
Build Docker image
       ↓
Local Docker Registry
       ↓
Docker Swarm deployment
```

For example, the backend development workflow uses a self-hosted runner labelled `dev`, builds the application image, and deploys it through Docker Swarm.

The Admin App, Analytics, Dashboard and Frontend v3 follow similar self-hosted-runner deployment patterns.

This means **moving the applications without moving/recreating the runner infrastructure would break the existing CI/CD process.**

---

# 5. Local Docker Registry

The current server is also running its own Docker Registry:

```text
localhost:50001
```

The registry is used extensively by the applications.

For example:

```text
localhost:50001/perzsibackend:latest
localhost:50001/authserver:latest
localhost:50001/perzsi-adminapp:latest
localhost:50001/perzsi-analytic-v2:latest
localhost:50001/perzsi-dashboard:latest
localhost:50001/perzsi-frontendv3:latest
```

The registry itself is a running Docker container and is exposed on port `50001`. 

This is an important migration dependency because the current deployment process assumes that images can be built and retrieved from this registry.

We therefore need to decide whether to:

* migrate the registry and its images, **or**
* rebuild the images from GitHub on the new server and recreate the registry, **or**
* redesign the deployment process to use an external registry.

For the immediate migration, preserving the existing architecture is likely the safest option.

---

# 6. Docker Swarm Networking

The current environment uses Docker Swarm overlay networks, particularly:

```text
perzsi-network
AdminNetwork
```

The audit confirms that both are Swarm-scoped overlay networks. 

These networks are referenced directly by the Docker Compose/Stack files.

For example, the backend expects:

```yaml
networks:
  perzsi-network:
    external: true
```

and the Admin App expects:

```yaml
networks:
  AdminNetwork:
    external: true
```

Therefore, these networks must be recreated correctly on the DatabaseMart server **before deploying the stacks**.

---

# 7. Docker Secrets

Another significant migration dependency is the use of **Docker Swarm secrets**.

The backend stack references external secrets such as:

* `main-env`
* `authserver`
* `redis-ca`
* `worker-dev`

Analytics also has its own external secret:

```text
analytic-env-v2
```

These secrets are not stored directly inside the Compose files.

This means simply copying the Git repositories to the new server will **not reproduce the running environment**.

The secrets must be identified, backed up/recreated securely, and injected into the new Swarm environment.

This is one of the areas I am treating carefully because losing or exposing these credentials could break the applications or create a security issue.

---

# 8. Persistent Docker Volumes

The server contains numerous Docker volumes.

Important examples include:

```text
perzsibackend_authkeys
perzsibackend_dashboard-attachment
perzsibackend_django-media-root
perzsibackend_django-static-root
perzsidashboard_dashboard-attachment
perzsi-product-feed_redis-data
perzsiadvertise_advertise-attachment
perzsiadvertisement_advertisement-attachment
perzsianalytics_analytic-attachment
perzsireels_reels-attachment
```



These volumes cannot simply be ignored because some may contain:

* uploaded files
* generated attachments
* authentication keys
* application media
* static files
* Redis data
* other application state

Each volume therefore needs to be classified as either:

**Required → migrate**

or

**Disposable/generated → recreate**

before the old server is shut down.

---

# 9. GitHub Actions Runner Situation

The server currently has multiple GitHub Actions runner services.

Several are active, including runners associated with:

* Backend
* Admin
* Analytics v2
* Dashboard
* Frontend v3

There are also several inactive/failed runners remaining from older projects/configurations. 

This is actually useful information because it means the migration should include a **runner cleanup/re-registration exercise**, rather than blindly copying the existing runner directories.

On the new server, the active required runners can be registered cleanly and assigned the correct labels, particularly:

```text
dev
worker
```

This should make the new environment easier to understand and maintain going forward.

---

# 10. Additional Services Requiring Review

The server is also running services outside the main Perzsi Swarm stacks.

For example:

```text
perzsi-product-feed
PTAE
PTIE
```

There are also host-level services such as:

* MySQL
* Redis
* Nginx
* Docker
* SSH
* cron
* GitHub Actions runners

The audit therefore needs to distinguish between:

### Application infrastructure

Things that should be intentionally migrated.

### Legacy infrastructure

Old runners, unused containers, abandoned projects, etc.

### Host-level dependencies

Services installed directly on Ubuntu rather than inside Docker.

This distinction is important because blindly migrating everything could reproduce unnecessary legacy configuration on the new VPS.

---

# 11. Current Risk Assessment

At this point, I would classify the migration as:

### 🟢 Low risk

* Git repositories
* Dockerfiles
* Docker Compose/Stack definitions
* GitHub Actions workflows

These are already represented in source control and can be recreated.

### 🟡 Medium risk

* Docker Swarm configuration
* GitHub Actions runners
* local Docker Registry
* external Docker networks
* environment configuration
* service dependencies

These need to be recreated carefully.

### 🔴 High attention

* Docker secrets
* persistent Docker volumes
* authentication keys
* uploaded files/media
* Redis/application state
* any databases running locally or accessed through local configuration
* undocumented host-level services

These require explicit backup and validation before the old server is retired.

---

# 12. Proposed Migration Strategy

I do **not** recommend shutting down the DigitalOcean server and trying to rebuild everything from memory.

Instead, the migration should happen in phases.

### Phase 1 — Complete audit

Identify:

* all applications
* all containers
* all Swarm services
* all Compose projects
* all volumes
* all networks
* all secrets
* all databases
* all runners
* all ports
* all DNS dependencies
* all external integrations

**Current phase.**

---

### Phase 2 — Prepare DatabaseMart VPS

Install/configure:

* Ubuntu
* Docker
* Docker Compose
* Docker Swarm
* required system packages
* firewall
* SSH access
* required storage

Then recreate:

```text
perzsi-network
AdminNetwork
```

and the required Docker secrets.

---

### Phase 3 — Recreate CI/CD

Install the required GitHub Actions runners on DatabaseMart.

Register them with the appropriate repositories and labels.

Then verify that a test deployment can successfully execute.

---

### Phase 4 — Recreate Registry

Set up the local Docker Registry or determine whether we want to move to a more appropriate registry architecture.

Then verify:

```text
build → push → pull → deploy
```

works correctly.

---

### Phase 5 — Migrate applications

Move applications one by one:

1. Backend
2. Worker/Celery infrastructure
3. Auth server
4. Admin App
5. Frontend v3
6. Dashboard
7. Analytics
8. Product Feed
9. PTAE
10. PTIE
11. Other confirmed active services

Each application should be tested independently before moving to the next.

---

### Phase 6 — Migrate persistent data

Migrate only the volumes/data that have been confirmed necessary.

Particular attention will be given to:

* authentication keys
* attachments
* media
* static assets where required
* Redis/application state
* database data

---

### Phase 7 — Functional validation

Before touching the DigitalOcean server:

* CI/CD deployment test
* API test
* authentication test
* frontend test
* admin test
* dashboard test
* analytics test
* scraper-related functionality
* worker/Celery test
* Redis connectivity
* file upload/download test
* external integrations
* DNS/reverse proxy verification

---

### Phase 8 — Cutover

Only after successful validation:

```text
DigitalOcean
    ↓
DatabaseMart
```

Update any necessary DNS/configuration references.

Keep the DigitalOcean server available temporarily as a rollback option.

---

### Phase 9 — Decommission

Once the new development environment has remained stable:

* disable old GitHub runners
* stop old services
* retain required backups
* remove obsolete infrastructure
* finally terminate the DigitalOcean development droplet

---

# 13. Current Progress

### Completed

* ✔️ Audited server OS and hardware resources
* ✔️ Audited Docker installation
* ✔️ Identified running containers
* ✔️ Identified Docker Swarm stacks
* ✔️ Identified Docker networks
* ✔️ Identified persistent Docker volumes
* ✔️ Identified GitHub Actions runners
* ✔️ Inspected application repositories
* ✔️ Inspected Docker Compose/Stack configurations
* ✔️ Inspected development CI/CD workflows
* ✔️ Identified Docker Registry dependency
* ✔️ Identified Docker Secret dependencies
* ✔️ Identified multiple applications outside the main Perzsi stack

### In progress

* ⏳ Mapping persistent data to individual applications
* ⏳ Auditing Docker secrets and configuration dependencies
* ⏳ Auditing databases and external services
* ⏳ Determining which containers/projects are still actively required
* ⏳ Preparing the DatabaseMart environment
* ⏳ Planning the migration/cutover sequence

---

# 14. Important Conclusion

The initial audit shows that the migration is **very feasible**, but it should not be treated as a simple server-to-server file copy.

The current development server has evolved into a fairly interconnected Docker/Swarm-based environment with CI/CD, a private registry, multiple applications, persistent volumes, secrets, and several legacy services.

The good news is that **the majority of the application deployment logic is already represented in GitHub repositories**, which significantly reduces the risk of rebuilding the environment.

The main challenge is reconstructing the **infrastructure state around those repositories** — particularly secrets, volumes, networks, runners, registry and any host-level services.

Therefore, my recommendation is to **build the DatabaseMart environment in parallel while keeping `srv008` untouched as the fallback**, migrate and validate services incrementally, and only decommission the DigitalOcean server after the new development environment has been verified.

---
