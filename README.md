# Kubernetes Microservices Package Management with Helm and Helmfile

This repository documents my hands-on progression in deploying and managing the
Online Boutique microservices application on **Linode Kubernetes Engine (LKE)**.

The project evolved through three stages:

1. Kubernetes manifests deployed with `kubectl`
2. Reusable Helm charts with service-specific values
3. Declarative multi-release management with Helmfile

The goal was not simply to replace Kubernetes YAML with Helm. The project helped
me understand where each approach was useful, where it became cumbersome, and
why the deployment process evolved.

---

## Technology Stack

- Kubernetes
- Linode Kubernetes Engine (LKE)
- kubectl
- Helm
- Helmfile
- YAML
- Bash
- Docker container images
- Online Boutique microservices application

---

## Deployment Environment

The application was deployed and tested on a **Linode Kubernetes Engine (LKE)**
cluster.

The general management flow was:

```text
Local Workstation
       |
       | kubectl
       | Helm
       | Helmfile
       v
Linode Kubernetes Engine (LKE)
       |
       +-- Frontend
       +-- Email Service
       +-- Cart Service
       +-- Checkout Service
       +-- Currency Service
       +-- Payment Service
       +-- Product Catalog Service
       +-- Recommendation Service
       +-- Shipping Service
       +-- Ad Service
       +-- Redis Cart
```

Kubernetes manages the workloads, Helm provides reusable Kubernetes packaging,
and Helmfile manages the collection of Helm releases.

---

## Repository Structure

```text
.
├── config.yaml
├── helmfile.yaml
├── install.sh
├── uninstall.sh
│
├── charts/
│   ├── microservice/
│   │   ├── .helmignore
│   │   ├── Chart.yaml
│   │   ├── values.yaml
│   │   └── templates/
│   │       ├── deployment.yaml
│   │       └── service.yaml
│   │
│   └── redis/
│       ├── .helmignore
│       ├── Chart.yaml
│       ├── values.yaml
│       └── templates/
│           ├── deployment.yaml
│           └── service.yaml
│
└── values/
    ├── ad-service-values.yaml
    ├── cart-service-values.yaml
    ├── checkout-service-values.yaml
    ├── currency-service-values.yaml
    ├── email-service-values.yaml
    ├── frontend-values.yaml
    ├── payment-service-values.yaml
    ├── productcatalog-service-values.yaml
    ├── recommendation-service-values.yaml
    ├── redis-values.yaml
    └── shipping-service-values.yaml
```

---

## Project Evolution

```text
Stage 1
Kubernetes Manifests
        |
        | kubectl apply -f config.yaml
        v
Stage 2
Reusable Helm Charts
        |
        | templates + values
        | individual Helm releases
        v
install.sh / uninstall.sh
        |
        | easier execution
        | but every release still maintained manually
        v
Stage 3
Helmfile
        |
        | release -> chart -> values
        |
        | helmfile sync
        | helmfile list
        | helmfile destroy
        v
Declarative Multi-Release Management
```

---

## Stage 1 — Kubernetes Deployment

The Online Boutique application was initially deployed using standard
Kubernetes manifests defined in:

```text
config.yaml
```

The application could be deployed with:

```bash
kubectl apply -f config.yaml
```

From an execution perspective, this was simple because one command could apply
the complete application configuration to the LKE cluster.

This stage provided hands-on experience with:

- Deployments
- Services
- replicas
- labels and selectors
- namespaces
- environment variables
- container and service ports
- liveness probes
- readiness probes
- resource requests
- resource limits
- internal service communication
- external application exposure

The limitation was repetition.

Many microservices required almost the same Kubernetes Deployment and Service
definitions, while only a few properties changed, such as:

- application name
- image
- image version
- replica count
- ports
- environment variables
- service type

That repetition motivated the move to Helm.

---

## Stage 2 — Helm

Helm was introduced to reduce repeated Kubernetes configuration.

Instead of maintaining separate Deployment and Service YAML for every
microservice, common resource definitions were moved into reusable Helm
templates.

### Reusable Microservice Helm Chart

Most application services use:

```text
charts/microservice/
```

The chart contains:

```text
microservice/
├── Chart.yaml
├── values.yaml
└── templates/
    ├── deployment.yaml
    └── service.yaml
```

The chart is defined as an application chart:

```yaml
apiVersion: v2
name: microservice
description: A Helm chart for Kubernetes
type: application
version: 0.1.0
appVersion: "1.16.0"
```

The reusable Deployment template references Helm values instead of hard-coded
service details.

For example:

```yaml
metadata:
  name: {{ .Values.appName }}
```

Replica count is configurable:

```yaml
replicas: {{ .Values.appReplicas }}
```

The container image is parameterized:

```yaml
image: "{{ .Values.appImage }}:{{ .Values.appVersion }}"
```

The Service is also configurable:

```yaml
type: {{ .Values.serviceType }}
```

and:

```yaml
port: {{ .Values.servicePort }}
targetPort: {{ .Values.containerPort }}
```

The same chart can therefore be reused by multiple microservices.

---

### Default Values

The reusable chart contains defaults in:

```text
charts/microservice/values.yaml
```

For example:

```yaml
appName: servicename
appImage: gcr.io/google-samples/microservices-demo/servicename
appVersion: v0.0.0
appReplicas: 1
containerPort: 8080

containerEnvVars:
- name: ENV_VAR_ONE
  value: "valueone"
- name: ENV_VAR_TWO
  value: "valuetwo"

servicePort: 8080
serviceType: ClusterIP
```

The default `values.yaml` provides the baseline configuration expected by the
templates.

Service-specific values files then override those defaults.

---

### Service-Specific Values Files

Each microservice has a values file under:

```text
values/
```

Examples include:

```text
values/email-service-values.yaml
values/cart-service-values.yaml
values/payment-service-values.yaml
values/shipping-service-values.yaml
```

The project contains values files for:

- Ad Service
- Cart Service
- Checkout Service
- Currency Service
- Email Service
- Frontend
- Payment Service
- Product Catalog Service
- Recommendation Service
- Redis Cart
- Shipping Service

The relationship is:

```text
                  Reusable Chart
               charts/microservice
                       |
                       v
                Helm Templates
                       |
        +--------------+--------------+
        |              |              |
        v              v              v
   Email Values    Cart Values    Payment Values
        |              |              |
        v              v              v
   Email Service   Cart Service   Payment Service
```

This reduced the need to repeat entire Deployment and Service definitions.

---

### Environment Variables

Environment variables are important in a microservices application because each
service may require different runtime configuration.

The reusable Deployment template loops over environment variables supplied
through values:

```yaml
env:
{{- range .Values.containerEnvVars}}
- name: {{ .name }}
  value: {{ .value | quote }}
{{- end}}
```

This allows the same Deployment template to support services with different
runtime requirements.

---

### Render Before Deployment

Before installing a release, the chart can be rendered locally:

```bash
helm template \
  -f values/email-service-values.yaml \
  charts/microservice
```

`helm template` renders the Kubernetes manifests without installing the release.

This is useful for inspecting the generated YAML before deployment.

For chart linting:

```bash
helm lint charts/microservice
```

---

### Install an Individual Helm Release

A service can be installed with:

```bash
helm install \
  -f values/email-service-values.yaml \
  emailservice \
  charts/microservice
```

Helm follows:

```text
helm install [options] RELEASE_NAME CHART
```

Therefore:

```text
values/email-service-values.yaml
        |
        └── service-specific values

emailservice
        |
        └── Helm release name

charts/microservice
        |
        └── reusable Helm chart
```

Another service can use the same chart:

```bash
helm install \
  -f values/cart-service-values.yaml \
  cartservice \
  charts/microservice
```

This demonstrated the main benefit of Helm: **the Kubernetes resource structure
is reused while service-specific configuration is supplied through values**.

---

## Redis as a Separate Chart

Redis uses:

```text
charts/redis/
```

because its workload configuration differs from the application microservices.

The Redis chart includes:

- Redis image configuration
- port `6379`
- TCP liveness probe
- TCP readiness probe
- resource requests
- resource limits
- volume mounting
- `emptyDir` storage
- ClusterIP service exposure

Its default values include:

```yaml
appName: redis
appImage: redis
appVersion: alpine
appReplicas: 1
containerPort: 6379
volumeName: redis-data
containerMountPath: /data

servicePort: 6379
```

### Redis Health Checks

Redis uses TCP socket probes:

```yaml
livenessProbe:
  initialDelaySeconds: 5
  tcpSocket:
    port: {{ .Values.containerPort }}
  periodSeconds: 5
```

and:

```yaml
readinessProbe:
  initialDelaySeconds: 5
  tcpSocket:
    port: {{ .Values.containerPort }}
  periodSeconds: 5
```

The liveness probe checks whether the workload remains healthy.

The readiness probe checks whether the workload is ready to receive traffic.

TCP is appropriate here because Redis communicates over a TCP port.

### Redis Resource Management

The Redis chart defines:

```yaml
resources:
  requests:
    cpu: 70m
    memory: 200Mi
  limits:
    cpu: 125m
    memory: 300Mi
```

Requests help Kubernetes make scheduling decisions.

Limits constrain how much CPU or memory a container can consume.

A resource limit does not fix a memory leak, but it limits how much of the
cluster's resources a problematic container can consume.

Redis deserves particular attention to memory because it primarily operates as
an in-memory data store.

---

## Stage 2 Limitation — Individual Helm Commands

Although Helm reduced repeated YAML, the initial deployment workflow became
tedious because each service still had to be installed separately.

For example:

```bash
helm install -f values/email-service-values.yaml emailservice charts/microservice
helm install -f values/cart-service-values.yaml cartservice charts/microservice
helm install -f values/currency-service-values.yaml currencyservice charts/microservice
helm install -f values/payment-service-values.yaml paymentservice charts/microservice
helm install -f values/recommendation-service-values.yaml recommendationservice charts/microservice
```

The same process had to be repeated for the remaining services.

From an execution perspective, the original:

```bash
kubectl apply -f config.yaml
```

was actually simpler because one command could apply the whole application.

Helm solved the **configuration-reuse problem**, but managing many independent
Helm releases was still cumbersome.

---

### install.sh and uninstall.sh

The next approach was to collect the Helm commands into scripts.

`install.sh` contains commands such as:

```bash
helm install -f values/redis-values.yaml rediscart charts/redis
helm install -f values/email-service-values.yaml emailservice charts/microservice
helm install -f values/cart-service-values.yaml cartservice charts/microservice
```

The remaining services follow the same pattern.

`uninstall.sh` removes the individual releases:

```bash
helm uninstall rediscart
helm uninstall emailservice
helm uninstall cartservice
```

This reduced manual typing, but the scripts still had to explicitly contain
every release.

Adding, removing, or renaming a microservice meant updating the scripts.

The scripts automated the commands, but they did not provide a clean
declarative model for the desired collection of releases.

This limitation led to Helmfile.

---

## Stage 3 — Helmfile

Helmfile provides centralized, declarative management of the Helm releases.

Instead of maintaining a sequence of installation commands, the releases are
declared in:

```text
helmfile.yaml
```

Each release defines:

- release name
- chart
- values file

For example:

```yaml
- name: emailservice
  chart: charts/microservice
  values:
    - values/email-service-values.yaml

- name: cartservice
  chart: charts/microservice
  values:
    - values/cart-service-values.yaml
```

The relationship becomes:

```text
                        helmfile.yaml
                             |
              +--------------+--------------+
              |                             |
              v                             v
        emailservice                  cartservice
              |                             |
              v                             v
    charts/microservice           charts/microservice
              |                             |
              v                             v
email-service-values.yaml      cart-service-values.yaml
```

The chart provides the reusable resource structure.

The values file provides service-specific configuration.

Helmfile connects all the releases to their charts and values from one central
configuration.

---

### Releases Managed by Helmfile

The current Helmfile manages:

- Redis Cart
- Email Service
- Cart Service
- Currency Service
- Payment Service
- Recommendation Service
- Product Catalog Service
- Shipping Service
- Ad Service
- Checkout Service
- Frontend Service

Most services use:

```text
charts/microservice
```

Redis uses:

```text
charts/redis
```

---

### Redis Inline Overrides

The Redis release also demonstrates inline Helmfile overrides:

```yaml
- name: rediscart
  chart: charts/redis
  values:
    - values/redis-values.yaml
    - appReplicas: "1"
    - volumeName: "redis-cart-data"
```

This combines:

```text
values/redis-values.yaml
```

with release-specific overrides.

This provides additional configuration flexibility without creating another
copy of the Redis chart.

---

### Synchronize Releases

The declared releases can be synchronized with:

```bash
helmfile sync
```

Helmfile reads the release definitions and reconciles the managed Helm releases
with the declared configuration.

This eliminates the need to run a separate installation command for every
microservice.

---

### List Releases

The releases configured in the Helmfile can be inspected with:

```bash
helmfile list
```

---

### Destroy Releases

The releases managed by Helmfile can be removed with:

```bash
helmfile destroy
```

This avoids maintaining a long list of separate `helm uninstall` operations.

---

## Kubernetes and Helm Best Practices Learned

### Use Versioned Images

Images should use explicit version tags rather than relying on `latest`.

For example:

```yaml
appImage: gcr.io/google-samples/microservices-demo/emailservice
appVersion: v0.8.0
```

Versioned images make deployments more predictable and make rollback and
troubleshooting easier because the application version is explicit.

---

### Liveness Probes

Liveness probes help Kubernetes determine when an application has become
unhealthy and may need to be restarted.

The health-check method should match the workload.

---

### Readiness Probes

Readiness probes tell Kubernetes whether an application is ready to receive
traffic.

A container can be running while the application inside it is still starting.

Liveness and readiness answer different questions:

```text
Liveness:
Is the application healthy enough to continue running?

Readiness:
Is the application ready to receive traffic now?
```

---

### Use the Appropriate Probe Protocol

Health checks should match the service protocol.

Application services can use gRPC health probes when they expose a suitable
gRPC health endpoint.

Redis can use TCP socket checks because Redis communicates over TCP.

---

### Resource Requests

Resource requests tell Kubernetes how much CPU and memory a workload expects to
need.

The scheduler uses requests when deciding where pods can be placed.

---

### Resource Limits

Resource limits restrict how much CPU or memory a container can consume.

They help prevent one workload from consuming excessive resources at the
expense of other workloads.

Limits do not prevent a memory leak, but they can restrict how much memory an
affected container can consume.

Resource sizing should reflect actual workload behavior.

Collaboration between developers and DevOps engineers is important when
determining appropriate CPU and memory requirements.

---

### Multiple Replicas

Important services should run multiple replicas where appropriate.

If one pod fails, another replica can continue serving the application and
reduce downtime.

---

### Multiple Worker Nodes

Production clusters should avoid relying on a single worker node.

Multiple nodes reduce the risk of a single node failure making the complete
application unavailable.

Replicas and multiple worker nodes work together to improve resilience.

---

### Labels and Selectors

Labels provide consistent identification of Kubernetes resources.

The reusable Deployment template labels pods with:

```yaml
labels:
  app: {{ .Values.appName }}
```

The Service uses:

```yaml
selector:
  app: {{ .Values.appName }}
```

This connects the Service to the intended pods.

---

### Namespaces

Namespaces help group and organize related Kubernetes resources.

They can also support access-control, quota, and policy boundaries.

---

### Service Exposure

Internal microservices generally do not need direct public exposure and can use
ClusterIP Services.

NodePort can be useful for testing or particular architectures, but a managed
cloud LoadBalancer or ingress-based design is generally more appropriate for
controlled external application access.

Only services that actually require external access should be exposed.

---

### Container Security

Container images should be checked for known vulnerabilities.

Containers should avoid running as root unless elevated privileges are
genuinely required.

Reducing privileges limits the potential impact of a vulnerable or compromised
application.

Useful practices include:

- vulnerability scanning
- explicit image versions
- non-root execution where possible
- minimal privileges
- controlled secret handling
- appropriate resource limits

---

### Kubernetes Cluster Upgrades

Kubernetes clusters should be kept on supported and secure versions.

Upgrades should be performed gradually using the upgrade path supported by the
Kubernetes platform.

In a multi-node cluster, worker nodes should be upgraded progressively so that
workloads can continue running on available nodes while maintenance is being
performed.

---

## Relationship Between the Three Stages

Not every later file contains every setting demonstrated in the earlier
Kubernetes manifest.

The initial `config.yaml` stage focused broadly on Kubernetes workload
configuration and operational concepts.

The reusable `charts/microservice` chart focused on:

- reusable templates
- values-driven configuration
- image parameterization
- replicas
- ports
- environment variables
- Service configuration

The Redis chart explicitly includes:

- TCP liveness probe
- TCP readiness probe
- resource requests
- resource limits
- storage configuration

The repository therefore demonstrates the **progression of the learning
exercise**, rather than implying that every setting from the original
Kubernetes manifest was copied unchanged into every later chart.

A production-ready reusable microservice chart could be extended further to
parameterize features such as:

- liveness probes
- readiness probes
- resource requests
- resource limits
- security contexts
- autoscaling
- affinity rules
- disruption budgets

---

## Useful Commands

### Deploy the Original Kubernetes Manifest

```bash
kubectl apply -f config.yaml
```

### Render a Helm Chart

```bash
helm template \
  -f values/email-service-values.yaml \
  charts/microservice
```

### Lint the Chart

```bash
helm lint charts/microservice
```

### Install an Individual Helm Release

```bash
helm install \
  -f values/email-service-values.yaml \
  emailservice \
  charts/microservice
```

### List Helm Releases

```bash
helm list
```

### Synchronize Helmfile Releases

```bash
helmfile sync
```

### List Helmfile Releases

```bash
helmfile list
```

### Remove Helmfile Releases

```bash
helmfile destroy
```

---

## What I Learned

This project helped me understand how Kubernetes package and release management
can evolve as the number of services increases.

The main lessons were:

1. Standard Kubernetes manifests are straightforward to deploy, but repeated
   configuration becomes difficult to maintain as the application grows.

2. Helm reduces duplication by separating reusable resource templates from
   values that differ between services.

3. Default values can live in the chart's `values.yaml`, while individual
   values files customize the same chart for different microservices.

4. Runtime environment variables can also be parameterized through Helm values.

5. `helm template` allows generated Kubernetes manifests to be inspected before
   installation.

6. Installing every microservice with a separate `helm install` command becomes
   tedious when many releases are involved.

7. `install.sh` and `uninstall.sh` reduce manual typing, but every release still
   has to be explicitly maintained in the scripts.

8. The original `kubectl apply -f config.yaml` was operationally simpler than
   those first two Helm deployment approaches.

9. Helmfile provides the more elegant orchestration layer by declaring release
   names, charts, and values centrally.

10. Kubernetes best practices such as versioned images, health checks, resource
    management, replicas, multiple worker nodes, labels, namespaces, controlled
    service exposure, container security, and controlled cluster upgrades
    remain important regardless of the packaging tool being used.

The central lesson from the project is:

**Helm improved configuration reuse, while Helmfile improved the orchestration and lifecycle management of multiple Helm releases.**
