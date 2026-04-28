# PingPongKong

[English](README.MD) | [한글](README.ko.md)

PingPongKong is a high-performance, Rust-powered connectivity validator designed to detect network drift in real time. It helps ensure that every required "Ping" in your infrastructure finds its expected "Pong."

## Ecosystem Structure

| Component | Environment | Description |
| :--- | :--- | :--- |
| **pingpongkong-k8s-agent** | Kubernetes | Rust DaemonSet that auto-discovers IPs through RBAC. |
| **pingpongkong-k8s-collector** | Kubernetes | Aggregates agent data, updates Kubernetes CRDs, and sends Discord alerts. |
| **pingpongkong-node-agent** | Legacy/Bare metal | Static binary for VMs and physical servers. |
| **pingpongkong-node-collector** | Legacy/Bare metal | Centralized aggregator for non-Kubernetes environments. |
| **state-samples** | Both | GitOps repositories containing connectivity matrices. |

## How It Works

1. **Declare state:** Define required connectivity, such as `worker -> s3:9000`, in your `state-samples` YAML.
2. **Watch drift:** Agents continuously perform TCP/UDP probes from the declared matrix.
3. **Aggregate truth:** Collectors scrape agents and provide a single cluster or node health view.
4. **Alert:** Failures trigger Discord notifications or Kubernetes events.

## Helm Commands

To pull this chart from the repository:

```bash
helm pull oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
```

Other commands:

```bash
helm show all oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
helm template <my-release> oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
helm install <my-release> oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
helm upgrade <my-release> oci://registry-1.docker.io/kimc1992/pingpongkong --version <new-version>
```

## Tech Stack

- **Language:** Rust, Tokio, kube-rs, Axum
- **Monitoring:** Prometheus exports and JSON logging
- **Delivery:** GitOps with Argo CD or manual workflows

## Collector Responsibilities

For a production-grade SRE tool, the collector should hold the GitLab token.

Giving the token to DaemonSet agents creates avoidable risk:

1. **Security blast radius:** A DaemonSet places the GitLab token on every worker node. If one node is compromised, the token can be stolen. Keeping it in the collector limits exposure to one secured deployment pod.
2. **GitLab API rate limits:** With hundreds of nodes, agents polling GitLab directly can quickly hit API limits. The collector fetches the configuration once and distributes it internally.
3. **Separation of concerns:** Agents should stay fast and focused. Their job is to run probes and report results, not handle GitLab API calls, HTTP proxies, TLS handshakes, or Git response parsing.

## Data Flow

### Phase 1: Sync

1. The collector deployment uses the GitLab token to fetch `matrix.yaml` and `discord.yaml`.
2. The collector parses the YAML and writes the current matrix into a Kubernetes ConfigMap, such as `pingpongkong-current-matrix`.

### Phase 2: Execution

1. Agents watch the ConfigMap through Kubernetes RBAC.
2. When the ConfigMap changes, agents reload their Tokio tasks and start TCP/UDP probes.
3. Agents expose results through Prometheus metrics, such as `:8080/metrics`, or a local JSON endpoint.

### Phase 3: Aggregation and Alerting

1. The collector continuously scrapes results from all agents.
2. When a failure appears, the collector checks the `discord.yaml` rules.
3. The collector applies rate limits, formats the message, and sends the Discord webhook.
