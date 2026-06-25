# CILIUM HUBBLE

## 1. Hubble이란?

Hubble은 Cilium 내장 네트워크 Observability platform. Cilium이 커널 레벨에서 eBPF를 통해 패킷 처리 시 발생하는 모든 이벤트를 수집하고, 이를 Kubernetes 메타데이터(Pod 명, Namespace, Label)과 결합하여 사람이 읽을 수 있는 Flow 로그를 제공.

기존 CNI에서 N/W 문제를 디버깅하기위해 노드에 접속하여 `tcpdump`를 뜨거나, iptables 로그를 파싱해야함. Hubble은 해당 과정을 대체하고, 단순한 패킷 캡처를 넘어 L3/L4/L7 레벨의 네트워크 가시성을 제공 함.

### 1.1 핵심 개념: Flow

Hubble의 기본 데이터 단위는 **Flow**. 하나의 Flow는 단일 네트워크 이벤트를 나타내며, 다음 정보를 포함한다.

| 필드 | 설명 | 예시 |
|------|------|------|
| Timestamp | 이벤트 발생 시각 | `Apr 25 10:03:41.234` |
| Source | 출발지 (Identity + Pod 정보) | `production/frontend-7b8d4` |
| Destination | 목적지 (Identity + Pod 정보) | `production/backend-3c9a1` |
| Verdict | 패킷 판정 결과 | `FORWARDED`, `DROPPED`, `ERROR` |
| Drop Reason | 드롭 사유 (verdict=DROPPED일 때) | `POLICY_DENIED`, `CT_MAP_INSERTION_FAILED` |
| Type | 이벤트 타입 | `L3/L4`, `L7`, `PolicyVerdict` |
| L4 Info | L4 프로토콜 정보 | `TCP/80`, `UDP/53` |
| L7 Info | L7 프로토콜 정보 (활성화 시) | `HTTP GET /api/users 200` |
| Identity | Cilium Security Identity | `src=12345, dst=67890` |

### 1.2 Hubble이 기존 도구와 다른 점

**tcpdump와의 차이:**
tcpdump는 raw 패킷을 보여준다. IP 주소와 포트 번호만으로 어떤 Pod인지, 어떤 서비스인지 파악해야 한다. Hubble은 eBPF 데이터패스에서 직접 이벤트를 가져오기 때문에, 패킷에 Kubernetes 컨텍스트가 이미 부착되어 있다. Pod 이름, 네임스페이스, Label, 정책 판정 결과까지 한 줄에 표시된다.

**kube-proxy 로그와의 차이:**
kube-proxy는 Service → Pod으로의 DNAT/SNAT 정보만 제공하며, 개별 요청 수준의 가시성은 없다. Hubble은 모든 패킷 판정을 기록하므로, 정책에 의한 차단, DNS 해석 과정, HTTP 요청/응답까지 추적할 수 있다.

---

## 2. Hubble 아키텍처

Hubble은 세 가지 컴포넌트로 구성.

### 2.1 Hubble Server (노드당 1개)

Cilium Agent 프로세스 안에 임베디드되어 동작한다. 별도의 Pod가 아니라 Cilium Agent의 일부이다.

**동작 방식:**
1. eBPF 프로그램이 커널에서 패킷을 처리할 때마다 이벤트를 perf ring buffer에 기록
2. Hubble Server가 이 ring buffer를 읽어 raw 이벤트를 파싱
3. Cilium의 Identity 캐시를 참조하여 IP → Identity → Pod/Label 매핑 수행
4. 완성된 Flow 이벤트를 in-memory ring buffer에 저장 (기본 4096개)
5. gRPC API를 통해 클라이언트(CLI, Relay)에 Flow 스트림 제공

**중요 특성:**
- 각 노드의 Hubble Server는 해당 노드를 통과하는 트래픽만 볼 수 있다
- in-memory ring buffer이므로 오래된 이벤트는 덮어써진다
- Hubble Server만으로는 단일 노드의 Flow만 조회 가능

### 2.2 Hubble Relay (클러스터당 1개 Deployment)

여러 노드의 Hubble Server를 집계하는 중간 계층이다.

**동작 방식:**
1. 클러스터 내 모든 Cilium Agent(= Hubble Server)에 gRPC 연결 수립
2. 각 노드의 Flow 스트림을 하나로 합쳐 클러스터 전체 뷰 제공
3. Hubble CLI와 Hubble UI가 Relay를 통해 전체 클러스터의 Flow를 조회

**왜 필요한가:**
Relay 없이는 `hubble observe` 명령이 실행 중인 노드의 트래픽만 보여준다. Pod A(node-1)에서 Pod B(node-2)로의 통신을 추적하려면 두 노드에 각각 접속해야 한다. Relay가 있으면 한 곳에서 클러스터 전체의 네트워크 흐름을 볼 수 있다.

### 2.3 Hubble UI (선택적 배포)

웹 기반 인터페이스로, 두 가지 핵심 기능을 제공한다.

**서비스 맵:** 네임스페이스 내의 서비스 간 통신을 토폴로지 맵으로 자동 생성한다. 각 연결선의 색상/두께로 트래픽 상태(정상/에러/차단)를 시각화한다. 

**Flow 탐색:** 특정 서비스 간의 Flow를 필터링하여 시간순으로 조회할 수 있다. 개별 Flow를 클릭하면 상세 정보(L7 요청/응답, 정책 판정 사유 등)를 확인할 수 있다.

---

## 3. Prometheus/Grafana와의 비교 및 연동

### 3.1 근본적 차이: 메트릭 vs Flow

Prometheus와 Hubble은 네트워크를 바라보는 시각이 근본적으로 다르다.

| 관점 | Prometheus | Hubble |
|------|-----------|--------|
| 데이터 모델 | 시계열 숫자 (counter, gauge, histogram) | 개별 네트워크 이벤트 (Flow) |
| 수집 방식 | Pull (주기적 scrape, 보통 15~30초) | Push (eBPF perf ring buffer, 실시간) |
| 질문 유형 | "지난 1시간 에러율은?" | "10:03:42에 이 요청이 왜 실패했는지?" |
| 데이터 크기 | 작음 (집계된 숫자) | 큼 (개별 이벤트) |
| 보존 기간 | 길다 (주~월 단위) | 짧다 (in-memory ring buffer) |
| 용도 | 추세 파악, 알러팅, 대시보드 | 실시간 디버깅, 포렌식, 정책 검증 |

### 3.2 같은 상황, 다른 시각

**시나리오:** frontend 서비스에서 backend 서비스로의 요청 중 일부가 실패하고 있다.

**Prometheus가 보여주는 것:**
```
http_requests_total{source="frontend", destination="backend", status="5xx"} = 47
http_request_duration_seconds_p99{source="frontend", destination="backend"} = 0.34
```
→ "에러가 47건 발생했고, p99 레이턴시가 340ms이다."

**Hubble이 보여주는 것:**
```
10:03:41 FORWARDED frontend-7b8d4 → backend-3c9a1 HTTP GET /api/users 200 12ms
10:03:42 DROPPED   frontend-7b8d4 → backend-db-0   TCP SYN port 5432 POLICY_DENIED
10:03:42 FORWARDED frontend-7b8d4 → backend-3c9a1 HTTP GET /api/users 200 8ms
10:03:43 DROPPED   frontend-7b8d4 → backend-db-0   TCP SYN port 5432 POLICY_DENIED
```
→ "frontend가 backend-db에 직접 접근을 시도하고 있는데, Network Policy에 의해 차단되고 있다."

Prometheus는 "무엇이" 일어나고 있는지 알려주고, Hubble은 "왜" 일어나고 있는지 알려준다. 둘은 경쟁 관계가 아니라 보완 관계이다.

### 3.3 Hubble → Prometheus 메트릭 연동

Hubble은 Flow 데이터를 집계하여 Prometheus 메트릭으로 노출할 수 있다. 즉, eBPF가 수집한 원시 이벤트를 Prometheus가 이해하는 시계열 숫자로 변환한다.

**활성화 방법 (Helm):**

```yaml
# values.yaml
hubble:
  enabled: true
  metrics:
    enabled:
      - dns
      - drop
      - tcp
      - flow
      - icmp
      - http
    serviceMonitor:
      enabled: true  # Prometheus Operator 사용 시
```

**주요 메트릭:**

| 메트릭 이름 | 설명 |
|------------|------|
| `hubble_flows_processed_total` | 처리된 총 Flow 수 |
| `hubble_drop_total` | 드롭 사유별 카운터 (label: reason, protocol) |
| `hubble_tcp_flags_total` | TCP 플래그별 카운터 (SYN, FIN, RST 등) |
| `hubble_dns_queries_total` | DNS 쿼리 수 (label: query, rcode) |
| `hubble_dns_responses_total` | DNS 응답 수 (label: query, rcode) |
| `hubble_http_requests_total` | HTTP 요청 수 (label: method, status, protocol) |
| `hubble_http_request_duration_seconds` | HTTP 요청 레이턴시 히스토그램 |

### 3.4 Grafana 대시보드 구성

Hubble 메트릭이 Prometheus에 수집되면, 기존 Grafana에서 바로 시각화할 수 있다. Cilium 공식 Grafana 대시보드를 임포트하거나, 커스텀 패널을 구성할 수 있다.

**활용 예시:**

```promql
# 네임스페이스별 드롭률
sum(rate(hubble_drop_total[5m])) by (source_namespace, destination_namespace)

# DNS 에러 비율
sum(rate(hubble_dns_responses_total{rcode!="No Error"}[5m]))
  / sum(rate(hubble_dns_responses_total[5m]))

# HTTP 에러율 (5xx)
sum(rate(hubble_http_requests_total{status=~"5.."}[5m]))
  / sum(rate(hubble_http_requests_total[5m]))
```
