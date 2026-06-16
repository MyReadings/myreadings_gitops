# myreadings-gitops

GitOps repository for deploying the MyReadings application on OpenShift using Argo CD.

MyReadings is a Quarkus-based modular monolith being incrementally decomposed into microservices. The first extracted service is the user-service. This repo manages the full deployment lifecycle: infrastructure provisioning (PostgreSQL, Keycloak, RabbitMQ), post-deploy configuration via Ansible K8s Jobs, application workloads, and CI pipelines.

## Repository structure

```
.
├── bootstrap.yaml                  # App-of-apps entry point — apply this first
├── argocd/
│   ├── kustomization.yaml          # Orchestrates all ArgoCD resources
│   ├── projects/
│   │   └── myreadings-project.yaml # AppProject for application workloads + Tekton
│   ├── myreadings-app.yaml         # Argo CD Application for the app (overlays/dev)
│   └── myreadings-tekton-app.yaml  # Argo CD Application for Tekton CI resources
├── base/                           # Shared manifests (Kustomize base)
│   ├── kustomization.yaml
│   ├── namespace.yaml
│   ├── postgres/postgrescluster.yaml
│   ├── keycloak/keycloak.yaml
│   ├── rabbitmq/rabbitmqcluster.yaml
│   ├── config-jobs/                # Ansible K8s Jobs for post-deploy configuration
│   │   ├── rbac.yaml               #   ServiceAccount + Role for reading Secrets
│   │   ├── configure-keycloak.yaml #   Job: realm, client, roles
│   │   └── configure-rabbitmq.yaml #   Job: vhost, user, queue, binding
│   ├── keycloak/
│   │   ├── keycloak.yaml           #   Keycloak CR (RHBK operator, Ingress disabled)
│   │   └── route.yaml              #   Explicit Route (operator Ingress lacks host)
│   ├── monolith/
│   │   ├── configmap.yaml, deployment.yaml, service.yaml
│   ├── user-service/
│   │   ├── configmap.yaml, deployment.yaml, service.yaml
│   └── ui/
│       ├── deployment.yaml, service.yaml, route.yaml
├── overlays/
│   ├── dev/
│   │   ├── kustomization.yaml      # Namespace, patches, image overrides (updated by CI)
│   │   └── patches/
│   └── prod/
│       ├── kustomization.yaml      # Namespace + resource limits
│       └── patches/
├── ansible/                        # Ansible playbooks + roles baked into EE image
│   ├── Containerfile               # Custom EE image build
│   ├── requirements.yml
│   ├── configure-keycloak.yml
│   ├── configure-rabbitmq.yml
│   └── roles/
│       ├── ocp_keycloak_config/
│       └── ocp_rabbitmq_config/
└── tekton/                         # CI pipelines, tasks, RBAC
    ├── kustomization.yaml
    ├── rbac.yaml                   # ServiceAccount + image-builder RoleBinding
    ├── task-maven-test.yaml        # Maven test/lint (no Docker needed)
    ├── task-maven-test-dind.yaml   # Maven test with DinD sidecar (Testcontainers)
    ├── task-acs-image-scan.yaml    # ACS image scan (graceful skip if not configured)
    ├── pipeline-maven.yaml         # Pipeline for Java/Quarkus services
    ├── pipeline-node.yaml          # Pipeline for Node.js apps (UI)
    ├── pipelineruns/
    │   ├── run-app.yaml            # PipelineRun for the monolith (maven pipeline)
    │   ├── run-ui.yaml             # PipelineRun for the UI (node pipeline)
    │   └── run-user-service.yaml   # PipelineRun for user-service (maven pipeline)
    └── triggers/
        ├── event-listener.yaml     # Webhook endpoint
        ├── trigger-binding.yaml
        ├── tt-app.yaml, tt-user-service.yaml, tt-ui.yaml
        └── route.yaml              # OpenShift Route to EventListener
```

## Prerequisites

These operators must be installed on the cluster before deploying MyReadings:

| Operator | Source | Channel |
|----------|--------|---------|
| CrunchyData Postgres Operator | certified-operators | v5 |
| Red Hat Build of Keycloak (RHBK) | redhat-operators | stable-v26.4 |
| OpenShift Pipelines (Tekton) | redhat-operators | latest |
| RabbitMQ Cluster Operator | community-operators | stable |
| OpenShift GitOps (Argo CD) | redhat-operators | latest |

### Secrets (created manually, not in Git)

**GitHub token** — needed by the Tekton pipeline to push image tag updates to this gitops repo. Create a fine-grained PAT with `Contents: Read and Write` on `myreadings-gitops` only.

The `git-cli` ClusterTask expects `.git-credentials` and `.gitconfig` keys:

```bash
oc create secret generic github-token \
  --from-literal=".git-credentials=https://<github-username>:<fine-grained-PAT>@github.com" \
  --from-literal=".gitconfig=[credential \"https://github.com\"]
  helper = store" \
  -n myreadings-dev
```

**ACS API token** (optional) — needed for image security scanning. If not present, the scan step is skipped gracefully:

```bash
oc create secret generic acs-api-token \
  --from-literal=token=<acs-api-token> \
  -n myreadings-dev
```

## How it works

### Application deployment (Argo CD)

A single `bootstrap.yaml` creates an app-of-apps that sets up everything:

1. **AppProject** — defines allowed sources, destinations, and resource types
2. **myreadings-dev Application** — deploys `overlays/dev` to the local cluster namespace `myreadings-dev`
3. **myreadings-tekton Application** — deploys Tekton pipelines, tasks, and triggers to `myreadings-dev`

### Application sync-waves

Within the application deployment, resources are ordered by sync-waves:

| Wave | Resources | Purpose |
|------|-----------|---------|
| 0 (default) | Namespace, SA, RBAC, ConfigMaps, Services, Routes | Foundation |
| 1 | PostgresCluster, Keycloak, RabbitmqCluster | Infrastructure (waits for healthy) |
| 2 | K8s Jobs: configure-keycloak, configure-rabbitmq | Post-deploy config (must complete) |
| 3 | Deployments: myreadings-app, myreadings-user-service, myreadings-ui | Application (after config is done) |

### Post-deploy configuration (Ansible K8s Jobs)

Keycloak and RabbitMQ configuration runs as Kubernetes Jobs using the custom Ansible Execution Environment image (`quay.io/rh-ee-drossi/myreadings-ee:latest`). The playbooks and roles are baked into the image, mirroring the same pattern used locally with Docker Compose (`myreadings_deploy/ansible/`).

### CI (Tekton)

There are two pipelines, one per build archetype:

**`myreadings-maven-build`** — for Java/Quarkus services (monolith, user-service):

```
clone ─┬─ lint (checkstyle)  ─┬─ build+push ─┬─ integration tests ─┬─ update gitops
       └─ unit tests ─────────┘              └─ image scan (ACS)   ─┘
```

- **Lint** runs `checkstyle:check` against the project's `checkstyle.xml` (Sun conventions). Currently non-blocking (`failOnViolation=false`) due to pre-existing violations; switch to `true` to enforce
- **Unit tests** run in parallel with lint for fast feedback. Pure Mockito/JUnit tests only — no Docker required
- **Build** uses `buildah` with the app's multi-stage Dockerfile, pushes to the OpenShift internal registry tagged with the commit SHA
- **Integration tests** and **image scan** run in parallel after the build, both gating the promotion step
- **Update gitops** clones this repo, updates the image reference in the Kustomize overlay's `images` section, commits and pushes — which triggers Argo CD to deploy

**`myreadings-node-build`** — for Node.js apps (UI):

```
clone ─── build+push ─── image scan (ACS) ─── update gitops
```

Focused pipeline with no Maven steps. The UI is built via `Dockerfile.ocp` (multi-stage: `npm run build` then Node.js Express server). Ready for future `npm test` / `eslint` steps when needed.

Both pipelines tag images with the full commit SHA (no `:latest`).

#### Testcontainers / DinD limitation

Tests that use `@QuarkusTest` with Testcontainers (JPA repository tests, controller integration tests) are excluded from the pipeline because they need a Docker daemon. A `maven-test-dind` task with a Docker-in-Docker sidecar is included and ready, but OpenShift Pipelines globally enforces `pipelines-scc` via `TektonConfig`, which blocks privileged containers. To enable DinD tests, change the SCC in TektonConfig:

```yaml
# oc edit tektonconfig config
spec:
  platforms:
    openshift:
      scc:
        default: privileged   # was: pipelines-scc
```

Until then, Testcontainers-based tests should be run locally or in an environment with Docker available.

## Cluster-specific configuration

The `overlays/dev/` directory contains values that depend on the target cluster. When deploying to a new cluster, update these:

### OIDC token issuer

The Quarkus backend validates the `iss` claim in JWT tokens. The application's `application.properties` ships a local-dev default (`http://localhost:8080/...`) which is wrong on OCP. The overlay patches the monolith ConfigMap with the correct external Keycloak URL.

Update `overlays/dev/patches/monolith-oidc-issuer.yaml`:

```yaml
data:
  QUARKUS_OIDC_TOKEN_ISSUER: "https://myreadings-keycloak-myreadings-dev.apps.<your-cluster-domain>/realms/my-readings"
```

The URL follows the OCP Route pattern `https://{route-name}-{namespace}.apps.{cluster-domain}/realms/{realm}`. This is the only value that needs changing per cluster. The UI server derives all other URLs dynamically at runtime from the incoming request's `Host` header.

### UI architecture (Node server, no nginx)

On OCP the UI runs a Node.js Express server (`Dockerfile.ocp` + `server.js`) instead of nginx:

- **API proxy**: `/api/*` requests are proxied to backend services via internal K8s DNS — no backend Route needed, no CORS. Uses `http-proxy-middleware` v3 with `pathFilter` to preserve the full request path
- **Dynamic config**: `GET /config.js` is generated at runtime by deriving the Keycloak Route URL from the request's `Host` header and the pod's `NAMESPACE` env var (Kubernetes Downward API)
- **Health check**: `GET /healthz` returns `{ "status": "UP" }`, used by readiness and liveness probes
- **SPA fallback**: all other requests serve the Vue app's `index.html`

For local development (`docker-compose`), the original `Dockerfile` with nginx is used unchanged.

### Health checks

- **Backend (Quarkus)**: Uses the `quarkus-smallrye-health` extension. Kubernetes probes hit `/q/health/ready` (readiness), `/q/health/live` (liveness), and `/q/health/started` (startup). The startup probe has a generous failure threshold to accommodate Quarkus boot time
- **UI (Node)**: A lightweight `/healthz` endpoint returns a JSON status response. Probes use this instead of `/` to avoid serving the full SPA HTML on every check

## Getting started

### 1. Install required operators

Install the operators listed in Prerequisites on your cluster (via OperatorHub or `oc apply` of Subscription manifests).

### 2. Apply the bootstrap

```bash
oc apply -f bootstrap.yaml
```

### 3. Create required secrets

See the Secrets section in Prerequisites above.

### 4. Build container images

Trigger a Tekton PipelineRun for each component:

```bash
oc create -f tekton/pipelineruns/run-app.yaml -n myreadings-dev            # Maven pipeline (monolith)
oc create -f tekton/pipelineruns/run-user-service.yaml -n myreadings-dev    # Maven pipeline (user-service)
oc create -f tekton/pipelineruns/run-ui.yaml -n myreadings-dev              # Node pipeline (UI)
```

Each pipeline builds, pushes the image to the internal registry, runs its quality gates, and commits the new image tag to this gitops repo — triggering Argo CD to deploy.

## Container images

| Component | Base image ref | Built by | Pipeline |
|-----------|---------------|----------|----------|
| Monolith | `quay.io/too-common-name/myreadings-app` | Tekton | `myreadings-maven-build` |
| User-service | `quay.io/too-common-name/myreadings-user-service` | Tekton | `myreadings-maven-build` |
| UI | `quay.io/too-common-name/myreadings-ui` | Tekton | `myreadings-node-build` |
| Keycloak (custom) | `quay.io/rh-ee-drossi/myreadings-keycloak:latest` | Manual | — |
| Ansible EE | `quay.io/rh-ee-drossi/myreadings-ee:latest` | Manual | — |

Tekton pushes images to the OpenShift internal registry with commit SHA tags. The Kustomize overlay maps the base image refs to the internal registry images.

## Known limitations and tech debt

- **Testcontainers in CI**: JPA and controller integration tests are excluded from the Maven pipeline due to the `pipelines-scc` restriction. The `maven-test-dind` task is ready for when the cluster allows privileged containers.
- **Checkstyle violations**: The project has pre-existing checkstyle violations. The lint step currently reports but does not fail the build. Flip `failOnViolation` to `true` once violations are cleaned up.
- **No frontend tests in CI**: The Node pipeline does not run `npm test` or linting yet. These steps can be added as custom Tekton tasks when the project adds frontend tests.
- **Dockerfiles use fully qualified image names**: Required by buildah on OpenShift, which enforces short-name resolution and cannot prompt for a registry without a TTY (e.g. `docker.io/library/maven:3.9-eclipse-temurin-21`).
- **OIDC issuer is per-cluster**: The `QUARKUS_OIDC_TOKEN_ISSUER` env var must match the external Keycloak Route URL. When moving to a new cluster, update `overlays/dev/patches/monolith-oidc-issuer.yaml` with the new apps domain.
- **RHBK operator Ingress has no host**: The RHBK operator creates a Kubernetes Ingress without a `host` field, so OpenShift doesn't auto-generate a usable Route. An explicit Route is included in `base/keycloak/route.yaml`. The operator's Ingress is disabled via `spec.ingress.enabled: false` in the Keycloak CR.
- **`quarkus.oidc.token.issuer` in source**: The `application.properties` ships a local-dev issuer (`http://localhost:8080/...`) that is wrong on OCP. It should be moved to the `%dev` profile in a future refactor so the env var override is no longer needed.
