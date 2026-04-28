# PingPongKong

[English](README.MD) | [한글](README.ko.md)

PingPongKong은 Rust 기반의 고성능 연결성 검증 도구입니다. 실시간으로 네트워크 드리프트를 감지하고, 인프라에서 필요한 모든 "Ping"이 기대한 "Pong"에 도달하는지 확인합니다.

## 생태계 구조

| 컴포넌트 | 환경 | 설명 |
| :--- | :--- | :--- |
| **pingpongkong-k8s-agent** | Kubernetes | RBAC를 통해 IP를 자동으로 발견하는 Rust DaemonSet입니다. |
| **pingpongkong-k8s-collector** | Kubernetes | 에이전트 데이터를 집계하고 Kubernetes CRD를 업데이트하며 Discord 알림을 보냅니다. |
| **pingpongkong-node-agent** | Legacy/Bare metal | VM과 물리 서버용 정적 바이너리입니다. |
| **pingpongkong-node-collector** | Legacy/Bare metal | Kubernetes 외부 환경을 위한 중앙 집계기입니다. |
| **state-samples** | Both | 연결성 매트릭스를 저장하는 GitOps 저장소입니다. |

## 동작 방식

1. **상태 선언:** `state-samples` YAML에 `worker -> s3:9000` 같은 필수 연결성을 정의합니다.
2. **드리프트 감시:** 에이전트가 선언된 매트릭스를 기준으로 TCP/UDP 프로브를 계속 수행합니다.
3. **결과 집계:** 컬렉터가 에이전트를 스크랩하여 클러스터 또는 노드 상태를 하나의 뷰로 제공합니다.
4. **알림:** 실패가 발생하면 Discord 알림 또는 Kubernetes 이벤트를 보냅니다.

## Helm 명령어

저장소에서 차트를 가져오려면:

```bash
helm pull oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
```

기타 명령어:

```bash
helm show all oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
helm template <my-release> oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
helm install <my-release> oci://registry-1.docker.io/kimc1992/pingpongkong --version 0.0.1
helm upgrade <my-release> oci://registry-1.docker.io/kimc1992/pingpongkong --version <new-version>
```

## 기술 스택

- **언어:** Rust, Tokio, kube-rs, Axum
- **모니터링:** Prometheus export와 JSON 로깅
- **배포:** Argo CD 또는 수동 워크플로우 기반 GitOps

## 컬렉터 책임

프로덕션 수준의 SRE 도구에서는 컬렉터가 GitLab 토큰을 보유하는 구조가 적합합니다.

DaemonSet 에이전트에 토큰을 주면 불필요한 위험이 생깁니다.

1. **보안 영향 범위:** DaemonSet은 GitLab 토큰을 모든 워커 노드에 배치합니다. 한 노드만 침해되어도 토큰이 유출될 수 있습니다. 컬렉터에만 토큰을 두면 노출 범위를 하나의 보안 강화된 Deployment Pod로 제한할 수 있습니다.
2. **GitLab API 제한:** 수백 개의 노드가 GitLab을 직접 폴링하면 API 제한에 쉽게 걸릴 수 있습니다. 컬렉터는 설정을 한 번 가져온 뒤 내부적으로 배포합니다.
3. **관심사 분리:** 에이전트는 빠르고 단순해야 합니다. 에이전트의 역할은 프로브를 실행하고 결과를 보고하는 것이며, GitLab API 호출, HTTP 프록시, TLS 핸드셰이크, Git 응답 파싱을 담당할 필요가 없습니다.

## 데이터 흐름

### Phase 1: Sync

1. 컬렉터 Deployment가 GitLab 토큰을 사용해 `matrix.yaml`과 `discord.yaml`을 가져옵니다.
2. 컬렉터가 YAML을 파싱하고 현재 매트릭스를 `pingpongkong-current-matrix` 같은 Kubernetes ConfigMap에 기록합니다.

### Phase 2: Execution

1. 에이전트가 Kubernetes RBAC를 통해 ConfigMap을 감시합니다.
2. ConfigMap이 변경되면 에이전트가 Tokio 태스크를 다시 로드하고 TCP/UDP 프로브를 시작합니다.
3. 에이전트는 `:8080/metrics` 같은 Prometheus 메트릭 또는 로컬 JSON 엔드포인트로 결과를 노출합니다.

### Phase 3: Aggregation and Alerting

1. 컬렉터가 모든 에이전트의 결과를 계속 스크랩합니다.
2. 실패가 발견되면 컬렉터가 `discord.yaml` 규칙을 확인합니다.
3. 컬렉터가 rate limit을 적용하고 메시지를 포맷한 뒤 Discord webhook을 전송합니다.
