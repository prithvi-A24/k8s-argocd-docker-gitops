# Google Online Boutique --- Docker, Kubernetes, GitHub Actions, Artifact Registry, GitOps & Argo CD

A hands-on DevOps/GitOps implementation built around Google's Online
Boutique microservices application.

## 1. Project objective

The project builds the DevOps platform around an existing microservices
application without changing its business logic.

The final delivery flow is:

``` text
Developer
  |
  | git push
  v
Application GitHub repository
  |
  v
GitHub Actions CI
  |
  +--------------------+
  |                    |
  v                    v
 GHCR             Google Artifact Registry
  |
  +--------------------+
           |
           v
     GitOps repository
           |
           v
        Argo CD
           |
           v
       Kubernetes
        Minikube
           |
           +--------------------+
           |                    |
           v                    v
     Online Boutique       Monitoring
                            |
                 +----------+----------+
                 |          |          |
                 v          v          v
             Prometheus  Grafana  Alertmanager
                                      |
                                      v
                                    Gmail
```

The key design decision is that **GitHub Actions performs CI and updates
Git desired state; Argo CD performs Kubernetes CD**. GitHub Actions does
not directly run `kubectl apply` against the cluster.

------------------------------------------------------------------------

## 2. Application

The application is Google's **Online Boutique**.

Local application repository:

``` text
/mnt/c/AI Learning Devops/Devops open source/Google Boutique/microservices-demo
```

Application branch:

``` text
devops
```

Application GitHub repository:

``` text
https://github.com/prithvi-A24/k8s-argocd-docker-.git
```

The project intentionally uses an existing public microservices
application rather than creating application code.

------------------------------------------------------------------------

## 3. GitOps repository

GitOps repository:

``` text
https://github.com/prithvi-A24/k8s-argocd-docker-gitops.git
```

Local path:

``` text
/mnt/c/AI Learning Devops/Devops open source/Google Boutique/k8s-argocd-docker-gitops
```

Branch:

``` text
main
```

The separation is:

``` text
Application repository
    source + Dockerfiles + CI
            |
            v
GitOps repository
    Kubernetes desired state
            |
            v
         Argo CD
            |
            v
       Kubernetes
```

------------------------------------------------------------------------

## 4. GitOps repository structure

The important structure is:

``` text
k8s-argocd-docker-gitops/
├── base/
│   ├── kustomization.yaml
│   └── online-boutique.yaml
│
├── overlays/
│   └── minikube/
│       └── kustomization.yaml
│
├── monitoring/
│   ├── helm/
│   │   └── kube-prometheus-stack-values.yaml
│   ├── kubernetes/
│   │   ├── grafana/
│   │   │   └── online-boutique.json
│   │   └── kustomization.yaml
│   ├── kustomization.yaml
│   └── argocd-monitoring.yaml
│
└── docs/
    └── monitoring.md
```

The exact contents evolved as CI image updates, dashboard provisioning,
and monitoring configuration were added.

------------------------------------------------------------------------

## 5. Local environment

The development environment is:

``` text
Windows
  |
  v
WSL2 / Ubuntu
  |
  v
Docker Desktop
  |
  v
Minikube
  |
  v
Kubernetes
```

Docker Desktop initially had a WSL/Docker socket problem. The Docker
Desktop named-pipe endpoint produced:

``` text
protocol not available
```

The working socket was:

``` text
unix:///var/run/docker.sock
```

A `wsl-docker` Docker context was created. The working workaround was:

``` bash
docker -H unix:///var/run/docker.sock ...
```

Normal Docker commands were subsequently working.

------------------------------------------------------------------------

## 6. Minikube

Minikube was used as the local Kubernetes cluster.

Versions during the project included:

``` text
Minikube:       v1.39.0
Kubernetes:     v1.37.0
kubectl client: v1.36.1
Kustomize:      v5.8.1
```

Cluster creation:

``` bash
minikube start --driver=docker
```

------------------------------------------------------------------------

## 7. Initial Kubernetes deployment

The original Online Boutique Kubernetes release manifest was used to
establish a working application baseline:

``` bash
kubectl apply -f ./release/kubernetes-manifests.yaml
```

The current application deployment contains:

``` text
adservice
cartservice
checkoutservice
currencyservice
emailservice
frontend
loadgenerator
paymentservice
productcatalogservice
recommendationservice
redis-cart
shippingservice
```

The frontend is exposed through:

``` text
frontend-external
```

Access:

``` bash
minikube service frontend-external --url
```

------------------------------------------------------------------------

## 8. Docker image build and Minikube

A local frontend image was built:

``` bash
docker build -t online-boutique-frontend:dev ./src/frontend
```

The image was loaded into Minikube:

``` bash
minikube image load online-boutique-frontend:dev
```

The Deployment was manually updated:

``` bash
kubectl set image deployment/frontend   server=online-boutique-frontend:dev
```

This was used to understand Kubernetes rolling updates before moving to
the full GitOps flow.

------------------------------------------------------------------------

## 9. Rollouts and rollback

The frontend Deployment was updated and its rollout behavior was
observed.

Useful commands:

``` bash
kubectl rollout status deployment/frontend
kubectl rollout history deployment/frontend
```

Operational rollback:

``` bash
kubectl rollout undo deployment/frontend
```

The project distinguishes this from a GitOps rollback.

Normal GitOps rollback:

``` bash
git revert <commit>
git push origin main
```

Argo CD then reconciles the reverted desired state.

`kubectl rollout undo` is an operational/emergency mechanism because it
changes live Kubernetes state independently of Git.

------------------------------------------------------------------------

## 10. Dockerfiles and CI build targets

The application repository contains Dockerfiles for:

``` text
adservice
cartservice
checkoutservice
currencyservice
emailservice
frontend
loadgenerator
paymentservice
productcatalogservice
recommendationservice
shippingservice
shoppingassistantservice
```

That is 12 CI build targets.

Important distinction:

> The current Kubernetes release manifest does not deploy
> `shoppingassistantservice`; it deploys `redis-cart` instead.

Therefore CI builds 12 service images, while the current Kubernetes
application deployment contains the workloads listed in Section 7.

------------------------------------------------------------------------

## 11. GitHub Actions CI

The CI workflow performs:

``` text
Checkout
   |
Detect changed services
   |
Prepare dynamic matrix
   |
Build changed images
   |
+-------------------------+
|                         |
v                         v
Push to GHCR          Push to GAR
|                         |
+------------+------------+
             |
             v
Update GitOps repository
             |
             v
Commit + push
```

The workflow supports:

``` text
push
pull_request
workflow_dispatch
```

and runs against:

``` text
devops
```

------------------------------------------------------------------------

## 12. Selective CI

`dorny/paths-filter@v3` detects which service directories changed.

Examples:

``` text
src/frontend/**
        -> frontend

src/cartservice/**
        -> cartservice
```

The detected service names become a dynamic GitHub Actions matrix.

Therefore a frontend-only change can result in:

``` text
build frontend
push frontend
update frontend GitOps tag
```

instead of rebuilding all services.

Workflow-only changes can result in no application image build.

------------------------------------------------------------------------

## 13. Bootstrap CI

The workflow also supports manual:

``` text
workflow_dispatch
```

with:

``` text
bootstrap = true
```

Bootstrap mode builds all 12 CI services.

This was used to initially populate the registries.

Normal application pushes use selective builds.

------------------------------------------------------------------------

## 14. Docker build context issue

Most services use their service directory as the build context.

`cartservice` is different:

``` text
Dockerfile:
./src/cartservice/src/Dockerfile

Build context:
./src/cartservice/src
```

The workflow contains a service-specific mapping for this.

This resolved the cartservice build failure.

------------------------------------------------------------------------

## 15. GHCR

GitHub Container Registry is one of the image registries.

Images use immutable Git SHA tags:

``` text
ghcr.io/prithvi-a24/online-boutique-<service>:<GITHUB_SHA>
```

The repository owner is lowercased because container registry names must
be lowercase.

The workflow uses:

``` bash
${GITHUB_REPOSITORY_OWNER,,}
```

------------------------------------------------------------------------

## 16. Google Artifact Registry

Artifact Registry was configured in:

``` text
Project:    avpn-dev-001
Region:     us-central1
Repository: online-boutique
```

Registry URI:

``` text
us-central1-docker.pkg.dev/avpn-dev-001/online-boutique
```

Images are:

``` text
us-central1-docker.pkg.dev/avpn-dev-001/online-boutique/<service>:<GITHUB_SHA>
```

The bootstrap workflow successfully pushed all 12 CI service images.

------------------------------------------------------------------------

## 17. Google Workload Identity Federation

GitHub Actions authenticates to Google Cloud without a long-lived
service-account JSON key.

Configuration:

``` text
Workload Identity Pool:
github-actions-pool

Provider:
github-provider

Service account:
online-boutique-ci@avpn-dev-001.iam.gserviceaccount.com
```

The provider maps GitHub OIDC claims including:

``` text
google.subject
attribute.repository
attribute.repository_owner
attribute.ref
```

The provider is restricted to the application repository.

The service account has:

``` text
roles/artifactregistry.writer
```

on the Artifact Registry repository.

The authentication flow is:

``` text
GitHub OIDC
    |
    v
Google Workload Identity Federation
    |
    v
Dedicated service account
    |
    v
Artifact Registry
```

No service-account private key is stored in GitHub.

------------------------------------------------------------------------

## 18. GitOps repository authentication

GitHub Actions updates the separate GitOps repository using:

``` text
GITOPS_TOKEN
```

The workflow checks out:

``` text
prithvi-A24/k8s-argocd-docker-gitops
```

and updates:

``` text
overlays/minikube/kustomization.yaml
```

------------------------------------------------------------------------

## 19. Immutable image tags

The project does not use `latest` for application deployment.

Images are tagged with:

``` text
GITHUB_SHA
```

This creates traceability:

``` text
Git commit
    |
    v
Docker image
    |
    v
GitOps image tag
    |
    v
Kubernetes Deployment
```

This makes rollback and version identification much easier.

------------------------------------------------------------------------

## 20. GitOps image update

After successful image publishing, GitHub Actions updates:

``` text
overlays/minikube/kustomization.yaml
```

with the changed service's new SHA.

Conceptually:

``` yaml
images:
  - name: ghcr.io/prithvi-a24/online-boutique-frontend
    newName: ghcr.io/prithvi-a24/online-boutique-frontend
    newTag: <git-sha>
```

The workflow commits changes with a message such as:

``` text
Update changed Online Boutique images to <GITHUB_SHA>
```

and pushes to:

``` text
main
```

------------------------------------------------------------------------

## 21. Git synchronization problem

Because both GitHub Actions and local development can modify the GitOps
repository, the local branch can become behind `origin/main`.

A non-fast-forward error occurred:

``` text
! [rejected] main -> main (fetch first)
```

This was a Git history synchronization problem, not an authentication
problem.

The correct resolution was:

``` bash
git fetch origin
git pull --rebase origin main
git push origin main
```

This preserved the remote CI commits and replayed the local commits on
top.

No force push was used.

------------------------------------------------------------------------

## 22. Kustomize

Kustomize provides the environment structure:

``` text
base/
    |
    +-- common Kubernetes configuration
    |
    v
overlays/minikube/
    |
    +-- Minikube-specific configuration
```

The base contains the Online Boutique Kubernetes manifest.

The Minikube overlay contains image mappings and environment-specific
configuration.

This structure can later support additional overlays such as staging or
production.

------------------------------------------------------------------------

## 23. Argo CD

Argo CD was installed in:

``` text
argocd
```

namespace.

The `online-boutique` Argo CD Application watches:

``` text
Repository:
https://github.com/prithvi-A24/k8s-argocd-docker-gitops.git

Revision:
main

Path:
overlays/minikube
```

The deployment flow is:

``` text
GitOps commit
     |
     v
Argo CD detects difference
     |
     v
Kustomize renders desired state
     |
     v
Kubernetes reconciliation
     |
     v
Rolling update
```

GitHub Actions does not directly deploy to Kubernetes.

------------------------------------------------------------------------

## 24. Argo CD access

Argo CD was accessed with:

``` bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Then:

``` text
https://localhost:8080
```

------------------------------------------------------------------------

## 25. Monitoring stack

Observability was implemented using:

``` text
kube-prometheus-stack
```

Helm chart:

``` text
91.8.0
```

Application version:

``` text
v0.94.1
```

Namespace:

``` text
monitoring
```

Components include:

``` text
Prometheus
Grafana
Alertmanager
Prometheus Operator
kube-state-metrics
node-exporter
```

------------------------------------------------------------------------

## 26. Monitoring through Argo CD

The monitoring stack itself was moved under Argo CD management.

The monitoring Application uses multiple sources:

``` text
Prometheus Community Helm repository
+
GitOps repository values
```

The GitOps values file is:

``` text
monitoring/helm/kube-prometheus-stack-values.yaml
```

The monitoring stack is therefore deployed declaratively through Argo
CD.

Argo CD itself is not currently self-managed.

------------------------------------------------------------------------

## 27. Prometheus

Prometheus was accessed using:

``` bash
kubectl port-forward   svc/monitoring-kube-prometheus-prometheus   -n monitoring   9090:9090
```

Prometheus successfully returned Kubernetes metrics.

Examples:

``` promql
sum(kube_pod_status_phase{
  namespace="default",
  phase="Running"
})
```

``` promql
sum(kube_pod_status_ready{
  namespace="default",
  condition="true"
})
```

------------------------------------------------------------------------

## 28. Grafana

Grafana was accessed with:

``` bash
kubectl port-forward   svc/monitoring-grafana   -n monitoring   3000:80
```

Username:

``` text
admin
```

The administrator password is stored in:

``` text
Secret: monitoring-grafana
Namespace: monitoring
```

It can be retrieved when needed with:

``` bash
kubectl get secret monitoring-grafana   -n monitoring   -o jsonpath="{.data.admin-password}" | base64 -d
echo
```

The Grafana Prometheus datasource is:

``` text
http://monitoring-kube-prometheus-prometheus.monitoring:9090/
```

------------------------------------------------------------------------

## 29. Online Boutique Grafana dashboard

The operations dashboard contains:

``` text
Ready Pods
Available Replicas
Desired Replicas
CPU Usage by Pod
Memory Usage by Pod
Pod Restarts
Deployment Availability
```

Representative PromQL:

### Ready Pods

``` promql
sum(
  kube_pod_status_ready{
    namespace="default",
    condition="true"
  }
)
```

### Available Replicas

``` promql
sum(
  kube_deployment_status_replicas_available{
    namespace="default"
  }
)
```

### Desired Replicas

``` promql
sum(
  kube_deployment_spec_replicas{
    namespace="default"
  }
)
```

### CPU

``` promql
sum by (pod) (
  rate(
    container_cpu_usage_seconds_total{
      namespace="default",
      container!="",
      container!="POD"
    }[5m]
  )
)
```

### Memory

``` promql
sum by (pod) (
  container_memory_working_set_bytes{
    namespace="default",
    container!="",
    container!="POD"
  }
)
```

### Pod restarts

``` promql
sum by (pod) (
  increase(
    kube_pod_container_status_restarts_total{
      namespace="default"
    }[1h]
  )
)
```

### Deployment availability

``` promql
sum by (deployment) (
  kube_deployment_status_replicas_available{
    namespace="default"
  }
)
```

------------------------------------------------------------------------

## 30. Grafana dashboard GitOps provisioning

The dashboard is stored in Git:

``` text
monitoring/kubernetes/grafana/online-boutique.json
```

Grafana's dashboard sidecar watches ConfigMaps labelled:

``` text
grafana_dashboard: "1"
```

The monitoring Kustomization manages the dashboard ConfigMap.

The dashboard was exported in Grafana V2 Resource format and placed
under GitOps management so it is not dependent on manual Grafana UI
configuration.

------------------------------------------------------------------------

## 31. Online Boutique Operations dashboard walkthrough

This section describes the **Online Boutique Operations** Grafana
dashboard: what it contains, how it is laid out, and how to load it.

The dashboard only reads from Prometheus. It does not create Kubernetes
resources, modify the application, or change the Prometheus
configuration.

Dashboard file:

``` text
online-boutique-operations.json
```

The full JSON is included in Appendix A at the end of this README.

Dashboard settings:

``` text
Title:        Online Boutique Operations
UID:          online-boutique-ops
Datasource:   Prometheus (selected through a datasource variable)
Namespace:    default
Time range:   Last 1 hour
Refresh:      30 seconds
Tooltip:      shared crosshair across panels
```

------------------------------------------------------------------------

### Layout

``` text
+---------------------+---------------------+---------------------+
| Online Boutique     | Available           | Desired             |
| Ready Pods          | Replicas            | Replicas            |
| (Stat)              | (Stat)              | (Stat)              |
+---------------------+---------------------+---------------------+
| CPU Usage by Pod                | Memory Usage by Pod           |
| (Time series)                   | (Time series)                 |
+---------------------------------+-------------------------------+
| Pod Restarts                    | Deployment Availability       |
| (Time series)                   | (Time series)                 |
+---------------------------------+-------------------------------+
```

Top row: three Stat panels for a quick health check.

Below: four Time series panels for trends and troubleshooting.

------------------------------------------------------------------------

### Panels

| Panel | Type | Unit | Meaning |
|---|---|---|---|
| Online Boutique Ready Pods | Stat | none | Pods whose Ready condition is true and can receive traffic |
| Available Replicas | Stat | none | Deployment replicas currently available |
| Desired Replicas | Stat | none | Replicas the deployments are configured to run |
| CPU Usage by Pod | Time series | CPU cores | 5-minute CPU rate per pod, excluding the pause container |
| Memory Usage by Pod | Time series | bytes (IEC) | Working set memory per pod |
| Pod Restarts | Time series | none | Container restarts per pod over a rolling 1-hour window |
| Deployment Availability | Time series | none | Available replicas per deployment |

Reading the Stat panels together:

``` text
Available Replicas == Desired Replicas   ->  healthy
Available Replicas <  Desired Replicas   ->  rollout in progress or outage
Ready Pods == 0                          ->  application is down
```

Stat panel thresholds:

``` text
Ready Pods         red at 0, green at 1 or more
Available Replicas red at 0, green at 1 or more
Desired Replicas   fixed blue (reference value)
```

Time series legends are shown as tables at the bottom of each panel:

``` text
CPU / Memory           mean, max, last
Pod Restarts           max, last
Deployment Availability min, last
```

------------------------------------------------------------------------

### PromQL

The queries are the same as in the previous dashboard section. Time
series panels additionally use a legend format:

``` text
CPU Usage by Pod          {{pod}}
Memory Usage by Pod       {{pod}}
Pod Restarts              {{pod}}
Deployment Availability   {{deployment}}
```

------------------------------------------------------------------------

### Import through the Grafana UI

Access Grafana:

``` bash
kubectl port-forward   svc/monitoring-grafana   -n monitoring   3000:80
```

Then open:

``` text
http://localhost:3000
```

Import steps:

``` text
1. Dashboards -> New -> Import
2. Upload dashboard JSON file
3. Select online-boutique-operations.json
4. Load
5. Import
6. Confirm the Datasource dropdown at the top shows Prometheus
```

The dashboard uses a datasource variable, so no datasource UID needs to
be edited before importing.

------------------------------------------------------------------------

### Import through GitOps

To keep the dashboard under GitOps management, place the JSON next to
the existing dashboard:

``` text
monitoring/kubernetes/grafana/online-boutique-operations.json
```

Then add it to the monitoring Kustomization as a ConfigMap labelled:

``` text
grafana_dashboard: "1"
```

Example `configMapGenerator` entry:

``` yaml
configMapGenerator:
  - name: online-boutique-operations-dashboard
    files:
      - grafana/online-boutique-operations.json
    options:
      labels:
        grafana_dashboard: "1"
      disableNameSuffixHash: true
```

Commit and push:

``` bash
git add monitoring/
git commit -m "Add Online Boutique Operations dashboard"
git push origin main
```

Argo CD syncs the ConfigMap and the Grafana dashboard sidecar loads it
automatically.

Dashboards provisioned this way are managed from Git. Change the JSON
in the repository rather than editing the dashboard in the Grafana UI.

------------------------------------------------------------------------

### Troubleshooting: No data

Panels depend on two metric sources:

``` text
kube_*        ->  kube-state-metrics
container_*   ->  cAdvisor (kubelet)
```

Check each metric in Grafana **Explore** or in Prometheus:

``` bash
kubectl port-forward   svc/monitoring-kube-prometheus-prometheus   -n monitoring   9090:9090
```

Example checks:

``` promql
kube_pod_status_ready{namespace="default"}
kube_deployment_status_replicas_available{namespace="default"}
container_cpu_usage_seconds_total{namespace="default"}
```

If a query returns nothing, check the scrape target under
**Status -> Targets** in Prometheus.

Note: `kubectl top` failing with `Metrics API not available` is a
separate Metrics Server limitation and does not affect this dashboard.
The dashboard reads from Prometheus, not `metrics.k8s.io`.

------------------------------------------------------------------------

## 32. Prometheus alert testing

A temporary test alert was created with:

``` promql
vector(1)
```

The alert was:

``` text
OnlineBoutiqueTestAlert
```

with a one-minute `for` duration.

This proved:

``` text
PrometheusRule
    |
    v
Prometheus
    |
    v
Alertmanager
```

was functioning.

------------------------------------------------------------------------

## 33. Online Boutique availability alert

A real application alert was created:

``` text
OnlineBoutiqueDeploymentUnavailable
```

Expression:

``` promql
kube_deployment_status_replicas_available{
  namespace="default",
  deployment="frontend"
} < 1
```

The alert waits one minute before firing.

It uses:

``` text
severity: critical
application: online-boutique
```

This represents a real application availability condition.

------------------------------------------------------------------------

## 34. Operational alert concepts

Operational rules were also created/designed for:

``` text
OnlineBoutiquePodNotReady
OnlineBoutiquePodRestarting
OnlineBoutiqueHighCPU
```

These cover:

``` text
pod readiness
container restarts
sustained CPU usage
```

------------------------------------------------------------------------

## 35. Alertmanager routing problem and resolution

Initially Prometheus was generating alerts and Alertmanager was
receiving them, but Alertmanager's generated configuration used:

``` text
receiver: null
```

The issue was therefore in the **Alertmanager routing/configuration
layer**, not in Prometheus rule evaluation.

A custom:

``` text
AlertmanagerConfig
```

named:

``` text
online-boutique-email
```

was created with:

``` text
alertmanagerConfig: online-boutique
```

The Helm values were updated with:

``` yaml
alertmanager:
  alertmanagerSpec:
    alertmanagerConfigSelector:
      matchLabels:
        alertmanagerConfig: online-boutique
```

------------------------------------------------------------------------

## 36. Gmail SMTP notification

SMTP was configured using:

``` text
smtp.gmail.com:587
```

TLS was required.

A Gmail App Password was created for SMTP authentication.

The actual App Password is deliberately not included in this README.

Credentials were stored in:

``` text
Secret:
alertmanager-smtp

Namespace:
monitoring
```

Keys:

``` text
username
password
```

Example creation command:

``` bash
kubectl create secret generic alertmanager-smtp   -n monitoring   --from-literal=username="EMAIL_ADDRESS"   --from-literal=password="APP_PASSWORD"
```

The secret must never be committed to Git.

------------------------------------------------------------------------

## 37. Email alert verification

The final notification flow is:

``` text
Prometheus
    |
    v
Alertmanager
    |
    v
AlertmanagerConfig
    |
    v
Gmail SMTP
    |
    v
Email
```

An actual Gmail notification was received successfully.

One received alert was:

``` text
NodeMemoryMajorPagesFaults
```

with:

``` text
severity = warning
job = node-exporter
namespace = monitoring
```

This proved the Alertmanager → SMTP → Gmail path.

That alert is a built-in monitoring-stack alert rather than an Online
Boutique-specific alert.

------------------------------------------------------------------------

## 38. Metrics Server limitation

The command:

``` bash
kubectl top nodes
```

returned:

``` text
error: Metrics API not available
```

The same happened with:

``` bash
kubectl top pods -A --sort-by=memory
```

This was intentionally left unresolved.

This does not mean Prometheus is broken.

The paths are different:

``` text
kubectl top
    |
    v
metrics.k8s.io
    |
    v
Metrics Server
```

versus:

``` text
Grafana
    |
    v
Prometheus
    |
    v
exporters / Kubernetes metrics
```

The current Minikube environment therefore has a known Metrics Server
limitation.

------------------------------------------------------------------------

## 39. Important troubleshooting categories

### Docker / WSL

Symptom:

``` text
protocol not available
```

Layer:

``` text
Docker Desktop / WSL socket
```

Working socket:

``` text
unix:///var/run/docker.sock
```

### Container registry naming

Symptom:

``` text
repository name must be lowercase
```

Layer:

``` text
Docker registry naming
```

Solution:

``` bash
${GITHUB_REPOSITORY_OWNER,,}
```

### Docker build context

Symptom:

``` text
cartservice build failure
```

Layer:

``` text
Docker build context
```

Solution:

``` text
./src/cartservice/src
```

### Git push rejected

Symptom:

``` text
fetch first
non-fast-forward
```

Layer:

``` text
Git history synchronization
```

Solution:

``` bash
git fetch origin
git pull --rebase origin main
git push origin main
```

### Alertmanager receiver null

Layer:

``` text
Alertmanager routing
```

Prometheus was generating alerts correctly, but Alertmanager was routing
to the null receiver.

### kubectl top unavailable

Layer:

``` text
Kubernetes Metrics API / Metrics Server
```

This remains a known limitation.

------------------------------------------------------------------------

## 40. Useful commands

### Kubernetes

``` bash
kubectl get nodes
kubectl get pods -A
kubectl get deployments -A
kubectl get services -A
```

### Application

``` bash
kubectl get pods
kubectl get deployments
kubectl get services
```

### Frontend

``` bash
minikube service frontend-external --url
```

### Rollouts

``` bash
kubectl rollout status deployment/frontend
kubectl rollout history deployment/frontend
kubectl rollout undo deployment/frontend
```

### Argo CD

``` bash
kubectl get applications -n argocd
kubectl get pods -n argocd
```

### Monitoring

``` bash
kubectl get pods -n monitoring
kubectl get prometheus -n monitoring
kubectl get alertmanager -n monitoring
kubectl get prometheusrules -n monitoring
kubectl get servicemonitors -n monitoring
```

### Prometheus

``` bash
kubectl port-forward   svc/monitoring-kube-prometheus-prometheus   -n monitoring   9090:9090
```

### Grafana

``` bash
kubectl port-forward   svc/monitoring-grafana   -n monitoring   3000:80
```

### Argo CD

``` bash
kubectl port-forward   svc/argocd-server   -n argocd   8080:443
```

------------------------------------------------------------------------

## 41. Git commands

Application repository:

``` bash
git status
git add .
git commit -m "message"
git push origin devops
```

GitOps repository:

``` bash
git status
git fetch origin
git pull --rebase origin main
git push origin main
```

History:

``` bash
git log --oneline --decorate --graph
```

Local commits not on remote:

``` bash
git log --oneline origin/main..HEAD
```

Remote commits not on local:

``` bash
git log --oneline HEAD..origin/main
```

------------------------------------------------------------------------

## 42. Current project status

Completed:

``` text
[x] Existing Online Boutique application
[x] Docker image builds
[x] Minikube Kubernetes cluster
[x] Kubernetes application deployment
[x] Frontend access
[x] Rolling update testing
[x] Kubernetes rollback testing
[x] GitHub Actions CI
[x] Selective service detection
[x] Dynamic service matrix
[x] GHCR publishing
[x] Artifact Registry publishing
[x] Workload Identity Federation
[x] GitOps repository
[x] Immutable SHA image tags
[x] Automated GitOps image updates
[x] Kustomize base/overlay
[x] Argo CD
[x] Argo CD application deployment
[x] Prometheus
[x] Grafana
[x] GitOps-managed Grafana dashboard
[x] Online Boutique Operations dashboard (documented, JSON in Appendix A)
[x] Alertmanager
[x] Prometheus alert testing
[x] Online Boutique availability alert
[x] Alertmanager routing
[x] Gmail email notification
[x] Monitoring stack managed through Argo CD
[x] CI/GitOps troubleshooting
[x] Rollback exercises

[ ] Kubernetes Metrics Server / kubectl top
```

The Metrics Server item was intentionally left for later.

------------------------------------------------------------------------

## 43. Final architecture

``` text
                         SOURCE CONTROL
                              |
                +-------------+-------------+
                |                           |
                v                           v
        Application Repo                GitOps Repo
                |                           |
                v                           |
        GitHub Actions CI                   |
                |                           |
        +-------+-------+                   |
        |               |                   |
        v               v                   |
      GHCR             GAR                  |
        |               |                   |
        +-------+-------+                   |
                |                           |
                +------ GitOps update ------+
                            |
                            v
                         Argo CD
                            |
                            v
                       Kubernetes
                        /                              /                               v           v
             Online Boutique   Monitoring
                              /    |                                  v     v      v
                        Prometheus Grafana Alertmanager
                                             |
                                             v
                                           Gmail
```

------------------------------------------------------------------------

## 44. Final CI/CD lifecycle

``` text
Git commit
    |
    v
GitHub Actions
    |
    +--> detect changed services
    |
    +--> build Docker image
    |
    +--> push GHCR
    |
    +--> push Artifact Registry
    |
    +--> update GitOps SHA
    |
    v
GitOps main
    |
    v
Argo CD
    |
    v
Kubernetes
    |
    v
Rolling deployment
    |
    v
Online Boutique
    |
    v
Prometheus
    |
    +--> Grafana
    |
    +--> Alertmanager
             |
             v
           Gmail
```

------------------------------------------------------------------------

## 45. Final takeaway

This project was not about writing a new application. It was about
building the DevOps platform around an existing, real multi-service
application.

The major capabilities implemented were:

``` text
Containerization
      +
CI
      +
GHCR
      +
Artifact Registry
      +
Workload Identity Federation
      +
GitOps
      +
Kustomize
      +
Argo CD
      +
Kubernetes
      +
Prometheus
      +
Grafana
      +
Alertmanager
      +
Gmail notifications
```

The resulting workflow provides a clear relationship between:

``` text
Git commit
    ↓
Docker image
    ↓
GitOps desired state
    ↓
Argo CD
    ↓
Kubernetes
    ↓
Application
    ↓
Observability
    ↓
Alerting
```

The project can now be extended toward additional environments, stronger
secret management, production Kubernetes, and more advanced
observability without changing the fundamental CI/CD architecture.

------------------------------------------------------------------------

## Appendix A. Online Boutique Operations dashboard JSON

File: `online-boutique-operations.json`

Import it through **Dashboards -> New -> Import**, or place it under
`monitoring/kubernetes/grafana/` for GitOps provisioning (see Section 31).

``` json
{
  "id": null,
  "uid": "online-boutique-ops",
  "title": "Online Boutique Operations",
  "description": "Operational health of the Google Online Boutique application in the default namespace.",
  "tags": ["kubernetes", "online-boutique", "operations"],
  "timezone": "browser",
  "schemaVersion": 39,
  "version": 1,
  "editable": true,
  "graphTooltip": 1,
  "refresh": "30s",
  "time": { "from": "now-1h", "to": "now" },
  "templating": {
    "list": [
      {
        "name": "datasource",
        "label": "Datasource",
        "type": "datasource",
        "query": "prometheus",
        "current": {},
        "hide": 0,
        "refresh": 1
      }
    ]
  },
  "panels": [
    {
      "id": 1,
      "type": "stat",
      "title": "Online Boutique Ready Pods",
      "description": "Number of pods in the default namespace whose Ready condition is true, meaning they are passing readiness checks and can receive traffic.",
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "gridPos": { "h": 5, "w": 8, "x": 0, "y": 0 },
      "targets": [
        {
          "refId": "A",
          "datasource": { "type": "prometheus", "uid": "${datasource}" },
          "expr": "sum(\n  kube_pod_status_ready{\n    namespace=\"default\",\n    condition=\"true\"\n  }\n)",
          "instant": false,
          "range": true
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 0,
          "color": { "mode": "thresholds" },
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "green", "value": 1 }
            ]
          }
        },
        "overrides": []
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "colorMode": "background",
        "graphMode": "area",
        "justifyMode": "center",
        "textMode": "auto",
        "orientation": "auto"
      }
    },
    {
      "id": 2,
      "type": "stat",
      "title": "Available Replicas",
      "description": "Total number of deployment replicas currently available to serve traffic across all deployments in the default namespace.",
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "gridPos": { "h": 5, "w": 8, "x": 8, "y": 0 },
      "targets": [
        {
          "refId": "A",
          "datasource": { "type": "prometheus", "uid": "${datasource}" },
          "expr": "sum(\n  kube_deployment_status_replicas_available{\n    namespace=\"default\"\n  }\n)",
          "instant": false,
          "range": true
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 0,
          "color": { "mode": "thresholds" },
          "thresholds": {
            "mode": "absolute",
            "steps": [
              { "color": "red", "value": null },
              { "color": "green", "value": 1 }
            ]
          }
        },
        "overrides": []
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "colorMode": "background",
        "graphMode": "area",
        "justifyMode": "center",
        "textMode": "auto",
        "orientation": "auto"
      }
    },
    {
      "id": 3,
      "type": "stat",
      "title": "Desired Replicas",
      "description": "Total number of replicas the deployments in the default namespace are configured to run. Compare with Available Replicas to spot rollouts or capacity gaps.",
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "gridPos": { "h": 5, "w": 8, "x": 16, "y": 0 },
      "targets": [
        {
          "refId": "A",
          "datasource": { "type": "prometheus", "uid": "${datasource}" },
          "expr": "sum(\n  kube_deployment_spec_replicas{\n    namespace=\"default\"\n  }\n)",
          "instant": false,
          "range": true
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "none",
          "decimals": 0,
          "color": { "mode": "fixed", "fixedColor": "blue" },
          "thresholds": {
            "mode": "absolute",
            "steps": [{ "color": "blue", "value": null }]
          }
        },
        "overrides": []
      },
      "options": {
        "reduceOptions": { "calcs": ["lastNotNull"], "fields": "", "values": false },
        "colorMode": "background",
        "graphMode": "area",
        "justifyMode": "center",
        "textMode": "auto",
        "orientation": "auto"
      }
    },
    {
      "id": 4,
      "type": "timeseries",
      "title": "CPU Usage by Pod",
      "description": "CPU consumed by each pod in the default namespace, measured in CPU cores (5-minute rate of container CPU seconds, excluding the pause container).",
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "gridPos": { "h": 9, "w": 12, "x": 0, "y": 5 },
      "targets": [
        {
          "refId": "A",
          "datasource": { "type": "prometheus", "uid": "${datasource}" },
          "expr": "sum by (pod) (\n  rate(\n    container_cpu_usage_seconds_total{\n      namespace=\"default\",\n      container!=\"\",\n      container!=\"POD\"\n    }[5m]\n  )\n)",
          "legendFormat": "{{pod}}",
          "range": true
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "suffix: cores",
          "decimals": 3,
          "color": { "mode": "palette-classic" },
          "custom": {
            "drawStyle": "line",
            "lineInterpolation": "smooth",
            "lineWidth": 1,
            "fillOpacity": 10,
            "showPoints": "never",
            "spanNulls": false
          }
        },
        "overrides": []
      },
      "options": {
        "legend": {
          "showLegend": true,
          "displayMode": "table",
          "placement": "bottom",
          "calcs": ["mean", "max", "lastNotNull"]
        },
        "tooltip": { "mode": "multi", "sort": "desc" }
      }
    },
    {
      "id": 5,
      "type": "timeseries",
      "title": "Memory Usage by Pod",
      "description": "Working set memory used by each pod in the default namespace, in bytes (IEC). This is the figure the kubelet uses for OOM decisions.",
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "gridPos": { "h": 9, "w": 12, "x": 12, "y": 5 },
      "targets": [
        {
          "refId": "A",
          "datasource": { "type": "prometheus", "uid": "${datasource}" },
          "expr": "sum by (pod) (\n  container_memory_working_set_bytes{\n    namespace=\"default\",\n    container!=\"\",\n    container!=\"POD\"\n  }\n)",
          "legendFormat": "{{pod}}",
          "range": true
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "bytes",
          "color": { "mode": "palette-classic" },
          "custom": {
            "drawStyle": "line",
            "lineInterpolation": "smooth",
            "lineWidth": 1,
            "fillOpacity": 10,
            "showPoints": "never",
            "spanNulls": false
          }
        },
        "overrides": []
      },
      "options": {
        "legend": {
          "showLegend": true,
          "displayMode": "table",
          "placement": "bottom",
          "calcs": ["mean", "max", "lastNotNull"]
        },
        "tooltip": { "mode": "multi", "sort": "desc" }
      }
    },
    {
      "id": 6,
      "type": "timeseries",
      "title": "Pod Restarts",
      "description": "Container restarts per pod over a rolling 1-hour window. Any value above zero suggests crashes, failed probes or OOM kills.",
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "gridPos": { "h": 9, "w": 12, "x": 0, "y": 14 },
      "targets": [
        {
          "refId": "A",
          "datasource": { "type": "prometheus", "uid": "${datasource}" },
          "expr": "sum by (pod) (\n  increase(\n    kube_pod_container_status_restarts_total{\n      namespace=\"default\"\n    }[1h]\n  )\n)",
          "legendFormat": "{{pod}}",
          "range": true
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short",
          "decimals": 0,
          "min": 0,
          "color": { "mode": "palette-classic" },
          "custom": {
            "drawStyle": "line",
            "lineInterpolation": "stepAfter",
            "lineWidth": 1,
            "fillOpacity": 10,
            "showPoints": "never",
            "spanNulls": false
          }
        },
        "overrides": []
      },
      "options": {
        "legend": {
          "showLegend": true,
          "displayMode": "table",
          "placement": "bottom",
          "calcs": ["max", "lastNotNull"]
        },
        "tooltip": { "mode": "multi", "sort": "desc" }
      }
    },
    {
      "id": 7,
      "type": "timeseries",
      "title": "Deployment Availability",
      "description": "Number of available replicas for each deployment in the default namespace. A drop indicates pods becoming unavailable or a rollout in progress.",
      "datasource": { "type": "prometheus", "uid": "${datasource}" },
      "gridPos": { "h": 9, "w": 12, "x": 12, "y": 14 },
      "targets": [
        {
          "refId": "A",
          "datasource": { "type": "prometheus", "uid": "${datasource}" },
          "expr": "sum by (deployment) (\n  kube_deployment_status_replicas_available{\n    namespace=\"default\"\n  }\n)",
          "legendFormat": "{{deployment}}",
          "range": true
        }
      ],
      "fieldConfig": {
        "defaults": {
          "unit": "short",
          "decimals": 0,
          "min": 0,
          "color": { "mode": "palette-classic" },
          "custom": {
            "drawStyle": "line",
            "lineInterpolation": "stepAfter",
            "lineWidth": 1,
            "fillOpacity": 10,
            "showPoints": "never",
            "spanNulls": false
          }
        },
        "overrides": []
      },
      "options": {
        "legend": {
          "showLegend": true,
          "displayMode": "table",
          "placement": "bottom",
          "calcs": ["min", "lastNotNull"]
        },
        "tooltip": { "mode": "multi", "sort": "desc" }
      }
    }
  ]
}
```
