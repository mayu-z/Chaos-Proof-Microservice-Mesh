# Chaos-Proof Microservice Mesh

> A distributed system that detects its own failures, reroutes traffic automatically, scales on the right signal, and documents what happened — without human intervention.

---

## What This Is

A production-grade, multi-region platform built around a single idea: **the system should heal itself**.

Most distributed systems alert you when something breaks. This one fixes it. A custom Go controller continuously watches SLO burn rates across three simulated regions. When a region degrades, traffic drains away immediately, pods scale to absorb the load, and the system ramps traffic back only after validating recovery. Every incident is recorded and a structured post-mortem is posted to Slack automatically.

A chaos monkey runs on a schedule — killing pods, injecting network latency — to continuously validate that the recovery loop actually works.

---

## Architecture Overview

```
[ User Traffic ]
       │
       ▼
[ Global Envoy Proxy ]  ◄──────────────────────────────┐
       │                                               │
       │  weighted routing (patched by Go controller)  │
       ├──────────────┬──────────────┐                 │
       ▼              ▼              ▼                 │
  [us-east]      [us-west]     [eu-central]            │
  namespace      namespace      namespace              │
                                                       │
  Each region contains:                                │
  - Envoy proxy (circuit breaker)                      │
  - Users service                                      │
  - Timelines service  ◄── gRPC (sync)                 │
  - Posts service                                      │
  - Regional DB (PostgreSQL)                           │
                                                       │
  All regions feed into:                               │
[ Kafka cluster ]  (rate-limited consumers per region) │
                                                       │
[ Prometheus ]  ◄── scrapes p99, burn rate, Kafka lag  │
       │                                               │
       ▼                                               │
[ Go Controller ] ─────────────────────────────────────┘
       │
       ├──► patches Envoy weights (drain fast, recover slow)
       ├──► freezes HPA + ArgoCD sync during active incident
       ├──► writes to Incident DB
       └──► triggers Slack post-mortem

[ Grafana ]  ◄── p99, traffic weights, pod churn, MTTR, error budget
[ Chaos Monkey ]  ──► kills pods + injects tc netem latency on schedule
```

---

## Tech Stack

| Layer | Tool | Why |
|---|---|---|
| Infrastructure provisioning | Terraform | Reproducible cluster and namespace setup |
| Node configuration | Ansible | Installs exporters, configures Kafka brokers, hardens nodes |
| Application deployment | ArgoCD | GitOps sync — declared state always matches live state |
| Runtime automation | Bash scripts | Chaos injection, health checks, traffic ramp validation |
| Service mesh | Envoy | Circuit breaking, outlier detection, weighted routing |
| Controller | Go + controller-runtime | Custom CRD watching SLO burn rate, patching Envoy xDS |
| Autoscaling | Kubernetes HPA + custom metrics | Scales on RPS and queue depth, not CPU |
| Messaging | Kafka | Cross-region async fanout with rate-limited consumers |
| Metrics | Prometheus | p99 latency, error budget, Kafka consumer lag |
| Dashboards | Grafana | Full incident timeline, traffic weights, recovery MTTR |
| Alerting | Slack webhook | Structured post-mortem on every chaos recovery |
| Incident storage | PostgreSQL | Operational history — MTTR, p99 before/after, replication lag |
| Inter-service comms | gRPC | Sync calls between Users, Timelines, Posts |

---

## Project Structure

```
chaos-proof-mesh/
├── terraform/                  # Infrastructure provisioning
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
├── ansible/                    # Node configuration
│   ├── inventory/
│   ├── playbooks/
│   │   ├── setup-nodes.yml
│   │   ├── configure-kafka.yml
│   │   └── install-exporters.yml
│   └── roles/
├── k8s/                        # Kubernetes manifests
│   ├── namespaces/
│   │   ├── us-east/
│   │   ├── us-west/
│   │   └── eu-central/
│   ├── envoy/
│   │   ├── global-proxy.yaml
│   │   └── virtual-services.yaml
│   └── argocd/
│       └── app-of-apps.yaml
├── controller/                 # Go controller
│   ├── main.go
│   ├── controller.go
│   ├── metrics.go
│   └── Dockerfile
├── services/                   # Social feed app
│   ├── users/
│   ├── posts/
│   └── timelines/
├── kafka/                      # Kafka setup + consumer configs
│   └── topics.yaml
├── scripts/                    # Bash scripts
│   ├── chaos/
│   │   ├── kill-pods.sh
│   │   └── inject-latency.sh
│   ├── recovery/
│   │   ├── health-check.sh
│   │   └── ramp-traffic.sh
│   └── setup/
│       └── bootstrap.sh
├── monitoring/                 # Prometheus + Grafana
│   ├── prometheus/
│   │   ├── prometheus.yml
│   │   └── alerts.yml
│   └── grafana/
│       └── dashboards/
└── docs/
    └── architecture.png
```

---

## How It Works

### 1. The Application

A social feed app split across three regions — each region runs three services:

- **Users** — handles user identity and authentication
- **Posts** — handles post creation and storage
- **Timelines** — builds and serves the feed by calling Posts over gRPC

When a user posts, the event publishes to Kafka. Each region consumes that topic and updates local timelines for followers in that region. Kafka consumers are rate-limited — so a recovering region does not get crushed by backlog when it comes back online.

Each region has its own PostgreSQL instance. The controller checks replication lag before declaring a region fully recovered — a healthy p99 with stale data is not a recovered region.

### 2. Infrastructure Setup

**Terraform** provisions the Kubernetes cluster and namespaces, simulating three regions as isolated namespaces with network policies that enforce realistic inter-region latency.

**Ansible** then runs across the provisioned nodes to:
- Install Prometheus node exporters and cAdvisor
- Configure Kafka broker settings per region
- Apply OS-level hardening and sysctl tuning
- Set up `tc netem` tooling on nodes for chaos injection

This separation is intentional — Terraform creates the infrastructure, Ansible configures what runs on it, ArgoCD deploys the applications.

### 3. The Control Loop

The Go controller is the brain of the system. It runs a continuous reconciliation loop:

```
every 2 minutes:
  read p99 latency from Prometheus per region
  calculate SLO burn rate (SLO: 99% of requests under 300ms)
  
  if burn rate > 2x threshold in a region:
    immediately drain traffic from that region (drop to 0%)
    freeze HPA scaling for that region (annotation on HPA object)
    pause ArgoCD sync for that namespace
    open incident record in PostgreSQL
    
  if region was draining and p99 now healthy:
    check replication lag — if lag > threshold, wait
    ramp traffic back: 10% → 30% → 60% → 100%
    run health-check.sh between each step
    unfreeze HPA and ArgoCD sync
    close incident record with MTTR
    post Slack summary
    
  if Prometheus returns stale or NaN metrics:
    do not assume healthy
    trigger alert
    hold current routing state
    if staleness persists > 5 minutes:
      fail-open: distribute traffic equally across all healthy regions
      (holding state risks trapping users in an already-degraded region)
```

The 2-minute evaluation window is intentional — it gives Envoy's local circuit breaker and the HPA time to resolve pod-level issues before escalating to a regional evacuation. A 2-pod failure should not cause a full regional drain.

### 4. Autoscaling Signal

HPA is wired to **RPS and Kafka consumer queue depth** via the custom metrics API — not CPU. CPU is a lagging indicator. By the time CPU spikes, users are already experiencing degradation.

p99 latency is reserved exclusively for the Go controller's regional traffic decisions. Keeping these signals separate prevents the HPA and controller from fighting each other in a feedback loop.

During an active traffic shift, the controller annotates the HPA object to freeze scaling for that region. This prevents the yo-yo effect where the controller drains traffic, HPA sees reduced load and scales down, controller restores traffic, and the region immediately degrades again on a smaller pod pool.

### 5. Envoy Configuration

Two layers of Envoy:

**Global Envoy Proxy** — sits at the entry point, holds the regional traffic weights. The Go controller patches these weights via the Envoy xDS API. This is the only thing the controller touches at the global layer.

**Regional Envoy Proxies** — sit inside each namespace, handle pod-level circuit breaking:
- Outlier detection: ejects unhealthy pods from the local pool automatically
- Max connections and retry limits: prevents cascade from one slow pod
- These operate independently of the Go controller

This two-layer design means a 2-pod failure in us-east is handled locally by regional Envoy without ever triggering a regional evacuation. The Go controller only fires when the region as a whole is degraded past the SLO burn rate threshold.

### 6. Traffic Drain and Recovery

**Drain is fast. Recovery is slow. These are asymmetric by design.**

If a region is burning through its error budget at 2x rate, waiting for a gradual drain means sending real users into a degraded region while the timer counts down. Traffic drops to 0% immediately.

Recovery is the opposite. The Go controller executes the ramp natively in Go — it does not shell out to a bash script. The logic looks like this in practice:

```go
// controller/controller.go
steps := []int{10, 30, 60, 100}
for _, weight := range steps {
    patchEnvoyWeight(region, weight)
    time.Sleep(60 * time.Second)
    if !runHealthCheck(region) {
        holdIncidentOpen(region)
        return
    }
}
closeIncident(region)
postSlackSummary(region)
```

`scripts/recovery/ramp-traffic.sh` exists as a standalone manual override tool — for local testing, runbook-driven intervention, or situations where an engineer needs to manually step traffic back without triggering the full controller reconciliation loop. It is not called by the controller itself.

If any health check fails during the ramp, the controller stops, holds the incident open, and waits for the next reconciliation cycle to re-evaluate. The region does not get traffic until it proves it can handle it.

### 7. Chaos Monkey

A Kubernetes CronJob runs on a randomized schedule. It executes two bash scripts:

**`kill-pods.sh`** — randomly selects pods across regions and deletes them, exercising pod-level recovery via Kubernetes restart policies and Envoy outlier detection.

**`inject-latency.sh`** — uses `tc netem` to inject packet delay and loss on inter-namespace network interfaces, simulating real inter-region degradation:

```bash
# scripts/chaos/inject-latency.sh
tc qdisc add dev eth0 root netem delay 200ms 50ms loss 5%
sleep $CHAOS_DURATION
tc qdisc del dev eth0 root
```

The chaos monkey does not notify the controller directly. The controller sees degraded p99 and acts — it does not matter whether degradation came from chaos, a bad deploy, or a real outage. The response is identical.

### 8. Observability

**Prometheus** scrapes:
- p99 latency per service per region
- Error rate and request volume
- Kafka consumer lag per topic per region
- PostgreSQL replication lag — exposed via `postgres_exporter` as `pg_wal_lsn_diff`, a gauge metric measuring the byte difference between primary and replica WAL positions. Ansible installs and configures `postgres_exporter` on each regional node as part of the setup playbook. The Go controller queries this metric directly from Prometheus before clearing a region as recovered.
- HPA freeze state (custom metric exported by controller)
- Envoy circuit breaker state

**Grafana dashboards** show:
- p99 per region with SLO burn rate overlay
- Traffic weight per region over time
- Pod churn (restarts, evictions)
- Kafka consumer lag per region
- Error budget remaining
- Full incident timeline — when chaos fired, when controller acted, when recovery completed, MTTR

**Slack post-mortem** is posted automatically on every recovery:

```
Incident #1234 — Resolved
Region:       us-east
Detected:     2025-06-16 12:01:22 UTC
Resolved:     2025-06-16 12:01:49 UTC
MTTR:         27 seconds

SLO burn rate at trigger:  4.2x
p99 at peak degradation:   847ms
p99 after recovery:        61ms

Traffic drain:   12:01:22 → 0% (immediate)
Traffic ramp:    12:01:31 → 10%, 12:01:49 → 100%

Replication lag at recovery: 120ms (within threshold)
```

### 9. Incident Database

Every incident is written to PostgreSQL:

```sql
CREATE TABLE incidents (
  id              SERIAL PRIMARY KEY,
  region          TEXT NOT NULL,
  detected_at     TIMESTAMPTZ NOT NULL,
  resolved_at     TIMESTAMPTZ,
  mttr_seconds    INTEGER,
  p99_at_peak     FLOAT,
  p99_after       FLOAT,
  burn_rate       FLOAT,
  replication_lag INTEGER,
  notes           TEXT
);
```

This gives the system operational history. Over time you can query average MTTR per region, which chaos scenarios cause the longest recovery, and whether the system is getting faster or slower at healing itself.

---

## SLO Definition

```
SLO: 99% of requests complete in under 300ms
Error budget: 1% of requests per rolling 30-day window

Controller threshold: fires when burn rate exceeds 2x
(burning through 30-day budget in 15 days)
```

The controller reacts to burn rate, not raw p99. A single slow request does not trigger a regional evacuation. A sustained pattern of budget burn does.

---

## Known Tradeoffs

**Single Kafka cluster** — in production this would be per-region Kafka brokers with MirrorMaker 2 for cross-region replication. A single cluster is a SPOF. For this project it is a deliberate simplification with rate-limited consumers to demonstrate the recovery pattern.

**Simulated regions as namespaces** — real multi-region would use separate clusters. Namespaces with network policies and `tc netem` latency injection give the same failure modes at near-zero cost.

**Database consistency** — writes go to regional PostgreSQL. During a regional drain, users are rerouted to another region. The controller waits for replication lag to normalize before restoring traffic, but there is a window of eventual consistency. A production system would use CockroachDB or Aurora Global for stronger guarantees.

---

## Design Decisions

**Why a custom Go controller instead of Istio?**
Istio does not freeze HPA scaling based on custom SLO burn metrics. The coordination between traffic shifting and autoscaling is the core of this project and requires custom logic that a standard service mesh does not expose.

**Why p99 for routing and RPS for scaling?**
p99 is a user-experience signal. RPS is a load signal. Mixing them causes the HPA and controller to react to the same metric and fight each other. Separating the signals keeps the two control loops independent.

**Why drain fast and recover slow?**
A degraded region is hurting users now. Speed of drain directly reduces user impact. A recovering region needs to prove stability before receiving full load — caches need to warm, connections need to re-establish. Symmetrical ramp up and down gets you a second outage.

**Why Ansible alongside Terraform?**
Terraform is declarative infrastructure. Ansible is imperative configuration. Terraform creates a node. Ansible makes it production-ready — exporters installed, kernel parameters tuned, Kafka brokers configured consistently. Mixing these concerns into Terraform makes the IaC brittle and hard to test.

---

## Running Locally

```bash
# Bootstrap the local environment
./scripts/setup/bootstrap.sh

# Provision infrastructure
cd terraform && terraform apply

# Configure nodes
cd ansible && ansible-playbook playbooks/setup-nodes.yml

# Deploy applications via ArgoCD
kubectl apply -f k8s/argocd/app-of-apps.yaml

# Trigger chaos manually
./scripts/chaos/kill-pods.sh us-east
./scripts/chaos/inject-latency.sh us-west 200ms

# Watch the controller react
kubectl logs -f deployment/go-controller -n control-plane

# View dashboards
open http://localhost:3000  # Grafana
```

---

## Grafana Dashboards

- **Regional health** — p99, error rate, traffic weight per region
- **Control loop** — controller decisions, HPA freeze state, Envoy weight history
- **Chaos timeline** — chaos events overlaid with p99 and recovery time
- **Error budget** — burn rate, budget remaining, SLO compliance over 30 days
- **Kafka** — consumer lag per region, fanout throughput, backlog size

---

## What This Demonstrates

- Custom Kubernetes controller in Go using controller-runtime
- p99 and SLO burn rate as autoscaling and routing signals
- Two-layer Envoy architecture: global routing + local circuit breaking
- Asymmetric traffic management: fast drain, slow recovery with health validation
- Coordinated control loops: HPA and ArgoCD frozen during active incidents
- Infrastructure as Code with Terraform + Ansible separation of concerns
- GitOps deployment with ArgoCD
- Chaos engineering as continuous validation, not a one-time test
- Structured incident management with automated post-mortem generation
- Kafka consumer rate limiting for thundering herd prevention
