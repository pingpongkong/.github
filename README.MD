# 🦍 PingPongKong

PingPongKong is a high-performance, Rust-powered connectivity validator designed to detect network drift in real-time. It ensures your infrastructure's "Pings" always find their "Pongs."

## 🏗 Ecosystem Structure

| Component | Environment | Description |
| :--- | :--- | :--- |
| **pingpongkong-k8s-agent** | Kubernetes | Rust DaemonSet. Auto-discovers IPs via RBAC. |
| **pingpongkong-k8s-collector** | Kubernetes | Aggregates Agent data. Updates K8s CRDs & Discord. |
| **pingpongkong-node-agent** | Legacy/BareMetal | Static binary for VMs/Physical servers. |
| **pingpongkong-node-collector** | Legacy/BareMetal | Centralized aggregator for non-K8s environments. |
| **state-samples** | Both | GitOps repositories containing the connectivity matrices. |

## 🚀 How it Works (Drift Detection)

1. **Declare State:** Define your required connectivity (e.g., Worker -> S3:9000) in your `state-sample` YAML.
2. **Watch Drift:** Agents constantly perform TCP/UDP handshakes based on the YAML.
3. **Aggregate Truth:** Collectors scrape Agents to provide a single "Cluster Health" or "Node Health" view.
4. **Alert:** Immediate notifications via Discord or Kubernetes Events if a handshake fails.

## 🛠 Tech Stack
- **Language:** Rust (Tokio, Kube-rs, Axum)
- **Monitoring:** Prometheus Export + JSON Logging
- **Delivery:** GitOps (ArgoCD / Manual)



For a production-grade SRE tool, the **Collector** should absolutely be the one holding the GitLab token. 

Here is why giving the token to the DaemonSet Agents is a bad idea, and why the Collector is the perfect "Brain" for this operation.

### 🦍 Why the Collector Must Hold the Token

1. **Security (Blast Radius):** If you give the token to the DaemonSet, the GitLab token exists on *every single worker node* in your cluster. If a hacker compromises just one random web pod and escapes to the node, they steal your Git token. If the Collector has it, the token only lives in one highly secured Deployment pod.
2. **GitLab API Rate Limiting:** If you have 500 nodes, and the DaemonSet checks Git for drift every 60 seconds, you are hitting the GitLab API 500 times a minute. GitLab will block you. The Collector only needs to hit the API **once**, and then it distributes the configuration.
3. **Separation of Concerns:** In the Rust ecosystem, we want the Agent to be as "dumb" and fast as possible. It shouldn't care about HTTP proxies, TLS handshakes with GitLab, or JSON parsing Git responses. Its only job is to Ping and Pong.

---

### 🌊 The Ideal "Kong" Data Flow

Here is exactly how the architecture should flow:

#### **Phase 1: Sync (The Collector)**
1. The **Collector** (Deployment) wakes up and uses the GitLab token to fetch the `matrix.yaml` and `discord.yaml`.
2. The Collector parses the YAML and pushes it into a Kubernetes **ConfigMap** (e.g., `pingpongkong-current-matrix`).

#### **Phase 2: Execution (The Agents)**
3. The **Agents** (DaemonSet) are watching that ConfigMap using native Kubernetes RBAC (no Git token required!).
4. As soon as the ConfigMap updates, the Agents automatically reload their internal Tokio tasks and start performing the TCP/UDP probes.
5. The Agents expose their results on a local Prometheus metrics port (`:8080/metrics`) or a local JSON endpoint.

#### **Phase 3: Aggregation & Alerting (The Collector)**
6. The Collector continuously scrapes the results from all the Agents.
7. If the Collector sees a failure, it checks its `discord.yaml` logic.
8. It applies the rate limit (e.g., max 5 per minute), formats a beautiful message, and fires the Discord Webhook.
