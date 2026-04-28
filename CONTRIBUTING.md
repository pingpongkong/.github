# Contributing to PingPongKong

We love Rust! To maintain high performance and low memory footprints for our H100 and Bare Metal nodes, please follow these guidelines:

## Development Workflow
1. **Common Logic:** If adding a new probe type (e.g., UDP health check), add it to the `common` crate so both K8s and Node agents benefit.
2. **Memory Safety:** Avoid heavy allocations in the `Prober` loop. We aim for < 10MB RAM usage per Agent.
3. **Async:** Use `tokio` for all networking tasks.

## Testing
- Ensure `cargo test` passes for the handshake logic.
- Use the `state-sample` YAMLs to verify configuration parsing.