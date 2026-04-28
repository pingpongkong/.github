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