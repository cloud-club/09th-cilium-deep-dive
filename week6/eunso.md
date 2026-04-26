# Week 6: Hubble 분석

## 1. Hubble 개요

### 1.1 Hubble이 필요한 이유

Kubernetes에서 애플리케이션은 Pod 단위로 계속 생성되고 사라진다. IP도 자주 바뀌고, Service를 거치면 실제 목적지 Pod가 어디인지 바로 보이지 않는다.

전통적인 방식으로 문제를 추적하려면 다음 도구를 번갈아 봐야 한다.

```bash
kubectl get pod -o wide
kubectl get svc,endpoints,endpointslice
tcpdump
iptables-save
conntrack -L
```

하지만 초보자 입장에서는 다음 질문에 답하기가 어렵다.

- A Pod가 B Pod와 통신했는가?
- 통신이 허용되었는가, 차단되었는가?
- 차단되었다면 정책 때문인가, DNS 때문인가, 라우팅 때문인가?
- 어떤 포트와 프로토콜을 사용했는가?
- Service 이름으로 요청했을 때 실제 어느 Pod로 갔는가?

Hubble은 Cilium이 eBPF datapath에서 관찰한 네트워크 이벤트를 사람이 읽기 쉬운 flow 형태로 보여준다. 즉, "패킷이 지나갔다"를 넘어서 "어떤 워크로드가 어떤 워크로드로, 어떤 포트와 정책 결과로 통신했는지"를 보여주는 도구다.

### 1.2 Hubble의 핵심 역할

| 영역 | 설명 |
|---|---|
| Flow Visibility | Pod, Service, IP, port, protocol, verdict를 포함한 네트워크 흐름 관찰 |
| Policy Debugging | CiliumNetworkPolicy에 의해 허용/차단된 트래픽 확인 |
| Service Map | 워크로드 간 의존 관계를 UI 그래프로 시각화 |
| Metrics | Prometheus/Grafana로 네트워크 동작과 보안 이벤트를 지표화 |
| Troubleshooting | "왜 연결이 안 되지?"를 flow 단위로 추적 |

### 1.3 Cilium과 Hubble의 관계

Hubble은 독립적인 CNI가 아니라 Cilium 위에서 동작하는 observability layer다.

중요한 점은 Hubble이 임의로 패킷을 복사해서 보는 것이 아니라, Cilium datapath가 이미 알고 있는 정보를 바탕으로 flow event를 만든다는 것이다. 그래서 Cilium identity, endpoint label, policy verdict 같은 Cilium 고유 정보를 함께 볼 수 있다.

---

## 2. Hubble 컴포넌트

### 2.1 Cilium Agent 내부 Hubble API

각 노드에는 `cilium-agent`가 DaemonSet으로 실행된다. 이 agent는 해당 노드에서 실행 중인 Pod들의 eBPF datapath를 관리한다.

Hubble API는 기본적으로 각 Cilium agent 안에서 제공된다. 이 경우 관찰 범위는 해당 노드의 Cilium agent가 본 트래픽으로 제한된다.

- Hubble이 켜져 있어도 기본 관찰 단위는 노드다.
- 클러스터 전체 흐름을 편하게 보려면 Hubble Relay가 필요하다.
- `hubble observe` 출력은 실시간 로그처럼 보이지만 실제로는 Cilium이 수집한 flow event stream이다.

### 2.2 Hubble Relay

Hubble Relay는 여러 노드의 Hubble API를 모아서 클러스터 단위 API를 제공한다.

Relay가 없으면 사용자는 노드별로 따로 확인해야 한다. Relay가 있으면 `hubble observe` 한 번으로 여러 노드에서 발생한 flow를 함께 볼 수 있다.

### 2.3 Hubble CLI

Hubble CLI는 flow를 터미널에서 확인하는 도구다.

```bash
# Hubble API 접근 확인
hubble status

# 실시간 flow 확인
hubble observe

# drop된 flow만 확인
hubble observe --verdict DROPPED

# 특정 Pod 관련 flow 확인
hubble observe --pod default/frontend

# HTTP flow 확인 (L7 visibility가 활성화된 경우)
hubble observe --protocol http
```

### 2.4 Hubble UI

Hubble UI는 flow를 그래프 형태의 Service Map으로 보여준다.

UI에서 보기 좋은 것:

- namespace별 서비스 의존 관계
- 어떤 워크로드가 어떤 워크로드와 통신하는지
- drop된 flow가 어느 구간에서 발생하는지
- 최근 flow 이벤트 목록

---

## 3. Flow Visibility

### 3.1 Flow란 무엇인가?

Hubble에서 flow는 하나의 네트워크 이벤트를 사람이 이해할 수 있게 정리한 단위다.

패킷 관점:

```
src IP:Port  →  dst IP:Port  protocol
```

Hubble flow 관점:

```
namespace/pod-a:random-port
  → namespace/pod-b:service-port
  verdict: FORWARDED or DROPPED
  identity: Cilium security identity
  labels: app=frontend, role=backend
  reason: policy decision, trace event, DNS, HTTP ...
```

즉 Hubble은 단순히 IP와 port만 보여주는 것이 아니라 Kubernetes/Cilium 문맥을 붙여준다.

### 3.2 일반적인 flow 출력 해석

예시:

```text
default/frontend-7d9c8:52436 -> default/backend-6f8b9:80 FORWARDED (TCP Flags: SYN)
```

필드별 의미:

| 필드 | 의미 |
|---|---|
| `default/frontend-7d9c8` | 출발지 Pod |
| `52436` | 출발지 임시 포트 |
| `default/backend-6f8b9` | 목적지 Pod |
| `80` | 목적지 포트 |
| `FORWARDED` | Cilium datapath가 전달한 flow |
| `TCP Flags: SYN` | TCP 연결 시작 패킷 |

- frontend가 backend에 실제로 연결을 시도했다.
- 목적지 포트는 80이다.
- 적어도 이 flow는 drop되지 않았다.
- TCP SYN이므로 연결 시작 단계다.

### 3.3 Verdict

Hubble에서 가장 먼저 봐야 하는 값 중 하나가 `verdict`다.

| Verdict | 의미 | 초보자 해석 |
|---|---|---|
| `FORWARDED` | 패킷이 전달됨 | Cilium이 이 트래픽을 통과시켰다 |
| `DROPPED` | 패킷이 드롭됨 | 정책, 라우팅, CT, 기타 이유로 막혔다 |
| `ERROR` | 처리 중 오류 | 비정상 상황이므로 reason 확인 필요 |
| `AUDIT` | audit mode에서 관찰 | 실제 차단 전 정책 영향 분석용 |

Drop만 보고 싶을 때:

```bash
hubble observe --verdict DROPPED
```

정책 문제를 볼 때는 전체 flow보다 drop flow를 먼저 보는 것이 좋다. 전체 flow에는 정상 트래픽도 많아서 초보자에게는 노이즈가 크다.

### 3.4 Direction

Hubble flow에는 방향성이 있다.

```
frontend  ── egress ──▶  backend
frontend  ◀─ ingress ──  backend
```

Cilium policy도 ingress와 egress를 따로 평가한다.

- `ingress`: 목적지 Pod로 들어오는 트래픽
- `egress`: 출발지 Pod에서 나가는 트래픽

예를 들어 `frontend -> backend` 요청이 막혔다면 두 가지를 모두 의심해야 한다.

1. frontend의 egress policy가 backend로 나가는 것을 막았는가?
2. backend의 ingress policy가 frontend에서 들어오는 것을 막았는가?

frontend에 egress default-deny가 걸려 있으면 backend ingress가 허용되어도 통신은 안 된다.

### 3.5 L3/L4/L7 visibility 차이

| 계층 | Hubble에서 보이는 것 | 예시 |
|---|---|---|
| L3 | IP, Cilium identity, endpoint, namespace, labels | `frontend -> backend` |
| L4 | TCP/UDP, port, ICMP, connection flags | `TCP 80`, `UDP 53` |
| L7 | HTTP, DNS, Kafka 같은 애플리케이션 프로토콜 정보 | `GET /api`, DNS query |

기본적으로 Hubble은 L3/L4 flow를 중심으로 보여준다. L7 정보를 보려면 Cilium L7 proxy와 L7 policy가 필요하다.

### 3.6 Service를 통한 통신은 어떻게 보이나?

Kubernetes에서는 Pod가 직접 Pod IP로 통신하기보다 Service DNS 이름으로 요청하는 경우가 많다.

```bash
curl http://backend.default.svc.cluster.local
```

논리적으로는 다음과 같다.

```
frontend Pod
  │
  │  dst = backend.default.svc.cluster.local:80
  ▼
Kubernetes Service
  │
  │  실제 endpoint 선택
  ▼
backend Pod 중 하나
```

Hubble은 Service 이름, Pod 이름, endpoint 정보를 함께 보여줄 수 있기 때문에 "Service로 요청했는데 실제 어느 Pod로 갔는지"를 추적하는 데 유용하다.

### 3.7 DNS flow

초보자에게 DNS flow는 매우 중요하다. 애플리케이션 로그에는 "connection refused"처럼 보이지만 실제 원인은 DNS 조회 실패일 수 있다.

```bash
hubble observe --protocol dns
```

DNS flow에서 확인할 것:

| 항목 | 확인 이유 |
|---|---|
| Query 이름 | 애플리케이션이 어떤 도메인을 조회했는가 |
| 응답 IP | DNS가 실제 IP를 돌려줬는가 |
| verdict | DNS 요청 자체가 정책에 막혔는가 |
| 대상 | CoreDNS/kube-dns로 갔는가 |

NetworkPolicy를 엄격하게 적용할 때는 DNS egress 허용을 잊기 쉽다.

---

## 4. L3 Policy

### 4.1 L3 policy란?

L3 policy는 "누가 누구와 통신할 수 있는가"를 정하는 정책이다.

여기서 "누구"는 보통 IP가 아니라 label 기반 endpoint다.

```
role=frontend  ── 허용 ──▶  role=backend
role=frontend  ── 차단 ──▶  role=database
```

Cilium의 강점은 Pod IP가 바뀌어도 label과 identity를 기준으로 정책을 적용한다는 점이다.

### 4.2 endpointSelector

`endpointSelector`는 이 정책이 적용될 대상 Pod를 고른다.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: backend-policy
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      role: backend
```

이 예시는 `role=backend` label을 가진 Pod에 정책을 적용한다.

### 4.3 Ingress L3 policy

Ingress는 선택된 Pod로 들어오는 방향이다.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-to-backend
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
```

읽는 순서:

1. `endpointSelector`: `role=backend` Pod를 보호한다.
2. `ingress`: backend로 들어오는 트래픽을 제어한다.
3. `fromEndpoints`: `role=frontend`에서 오는 트래픽만 허용한다.

```
role=frontend  ── allowed ──▶  role=backend
role=client    ── denied  ──▶  role=backend
role=admin     ── denied  ──▶  role=backend
```

### 4.4 Egress L3 policy

Egress는 선택된 Pod에서 나가는 방향이다.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-egress-to-backend
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toEndpoints:
    - matchLabels:
        role: backend
```

읽는 순서:

1. `endpointSelector`: `role=frontend` Pod를 선택한다.
2. `egress`: frontend에서 나가는 트래픽을 제어한다.
3. `toEndpoints`: `role=backend`로 가는 트래픽만 허용한다.

```
role=frontend  ── allowed ──▶  role=backend
role=frontend  ── denied  ──▶  role=database
role=frontend  ── denied  ──▶  world
```

### 4.5 Default Deny 이해

Cilium의 기본 정책 모드에서는 아무 정책도 선택하지 않은 endpoint는 ingress/egress가 기본 허용이다.

하지만 어떤 정책이 endpoint를 선택하고 `ingress` 또는 `egress` 섹션을 가지면, 해당 방향은 default-deny 상태가 된다.

```
정책 없음
  → ingress 허용
  → egress 허용

ingress 정책이 endpoint를 선택
  → ingress는 명시적으로 허용된 것만 허용
  → egress는 기존 상태 유지

egress 정책이 endpoint를 선택
  → egress는 명시적으로 허용된 것만 허용
  → ingress는 기존 상태 유지
```

Cilium policy는 허용 목록 방식으로 생각해야 한다. 정책이 적용된 방향에서는 허용하지 않은 트래픽은 막힌다.

### 4.6 L3 policy와 Hubble 확인

정책 적용 후 정상 flow:

```text
default/frontend:51432 -> default/backend:80 FORWARDED (TCP Flags: SYN)
```

정책으로 차단된 flow:

```text
default/client:52011 -> default/backend:80 Policy denied DROPPED (TCP Flags: SYN)
```

확인 명령:

```bash
hubble observe --to-pod default/backend --verdict DROPPED
```

이 명령으로 backend로 들어오려다 차단된 트래픽을 볼 수 있다.

---

## 5. L4 Policy

### 5.1 L4 policy란?

L4 policy는 어떤 포트와 프로토콜로 통신할 수 있는가를 정한다.

L3가 사람 관계라면:

```
frontend는 backend와 통신 가능
```

L4는 문 종류까지 정한다.

```
frontend는 backend의 TCP/80만 접근 가능
frontend는 backend의 TCP/5432는 접근 불가
```

### 5.2 L3 + L4 ingress policy

아래 정책은 `role=frontend` Pod만 `role=backend` Pod의 TCP 80번 포트로 접근할 수 있게 한다.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: allow-frontend-to-backend-http
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      role: backend
  ingress:
  - fromEndpoints:
    - matchLabels:
        role: frontend
    toPorts:
    - ports:
      - port: "80"
        protocol: TCP
```

정책 해석:

| YAML 필드 | 의미 |
|---|---|
| `endpointSelector.role=backend` | backend Pod에 적용 |
| `ingress` | backend로 들어오는 트래픽 제어 |
| `fromEndpoints.role=frontend` | frontend에서 오는 트래픽만 |
| `toPorts.port=80` | backend의 80번 포트만 |
| `protocol=TCP` | TCP만 허용 |

허용/차단 결과:

```
frontend ── TCP/80   ──▶ backend  allowed
frontend ── TCP/8080 ──▶ backend  denied
client   ── TCP/80   ──▶ backend  denied
```

### 5.3 L4 egress policy

아래 정책은 `role=frontend` Pod가 외부 또는 다른 endpoint로 나갈 때 TCP 443만 사용하도록 제한한다.

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-egress-https-only
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

해석:

```
frontend ── TCP/443 ──▶ any destination  allowed
frontend ── TCP/80  ──▶ any destination  denied
frontend ── UDP/53  ──▶ DNS              denied
```

- 이 정책만 있으면 DNS도 막힐 수 있다.
- 실제 운영에서는 DNS 허용 정책을 같이 넣는 경우가 많다.
- "HTTPS만 허용"이라고 썼는데 DNS가 막혀서 도메인 접속이 실패하는 실수가 흔하다.

### 5.4 DNS를 함께 허용하는 egress 예시

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: frontend-egress-web-and-dns
  namespace: default
spec:
  endpointSelector:
    matchLabels:
      role: frontend
  egress:
  - toEndpoints:
    - matchLabels:
        "k8s:io.kubernetes.pod.namespace": kube-system
        "k8s:k8s-app": kube-dns
    toPorts:
    - ports:
      - port: "53"
        protocol: UDP
  - toPorts:
    - ports:
      - port: "443"
        protocol: TCP
```

frontend는
  1. kube-dns의 UDP/53으로 DNS 질의를 할 수 있고
  2. TCP/443으로 나갈 수 있다.

### 5.5 L4 policy와 Service port

Cilium L4 policy는 Service port mapping 이후의 포트를 기준으로 적용될 수 있다. 그래서 Service의 `port`와 Pod의 `targetPort`가 다를 때는 실제 어느 포트를 기준으로 정책이 적용되는지 주의해야 한다.

예시:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: backend
spec:
  ports:
  - port: 80
    targetPort: 8080
  selector:
    role: backend
```

사용자는 `backend:80`으로 접근하지만 실제 Pod는 `8080`에서 듣고 있다.

정책을 작성할 때는 Cilium 문서와 실제 flow를 함께 확인하는 것이 안전하다.

```bash
hubble observe --to-pod default/backend --port 8080
```

Service port와 Pod targetPort가 다르면 Hubble 출력에서 실제 목적지 Pod와 port가 어떻게 보이는지 먼저 확인한 뒤 정책을 좁혀가는 것이 좋다.

---

## 6. Hubble로 정책 디버깅하기

### 6.1 문제 상황 예시

가정:

```
frontend Pod가 backend Service로 요청해야 한다.
하지만 curl이 timeout 된다.
```

초보자는 먼저 애플리케이션 문제라고 생각하기 쉽지만, 네트워크에서는 여러 단계가 있다.

```
frontend
  │
  ├─ DNS 조회 성공?
  │
  ├─ Service IP 확인?
  │
  ├─ 실제 backend Pod 선택?
  │
  ├─ frontend egress 허용?
  │
  ├─ backend ingress 허용?
  │
  └─ backend 애플리케이션이 포트 listen?
```

### 6.2 1단계: drop flow 확인

```bash
hubble observe --verdict DROPPED
```

만약 다음과 같이 보이면 정책 문제 가능성이 높다.

```text
default/frontend:39822 -> default/backend:80 Policy denied DROPPED (TCP Flags: SYN)
```

확인할 것:

- 출발지 Pod가 예상한 Pod인가?
- 목적지 Pod/Service가 맞는가?
- port가 정책에서 허용한 port인가?
- drop reason이 `Policy denied`인가?

### 6.3 2단계: 목적지 기준으로 필터링

```bash
hubble observe --to-pod default/backend
```

backend로 들어오는 모든 flow를 본다.

정상 요청과 차단 요청이 같이 보이면, label 차이 또는 port 차이를 의심할 수 있다.

```text
default/frontend:41100 -> default/backend:80 FORWARDED
default/client:41110   -> default/backend:80 Policy denied DROPPED
```

### 6.4 3단계: 출발지 기준으로 필터링

```bash
hubble observe --from-pod default/frontend
```

frontend에서 나가는 flow를 본다.

여기서 DNS 요청이 drop되는지 확인한다.

```text
default/frontend:51222 -> kube-system/coredns:53 Policy denied DROPPED (UDP)
```

이 경우 backend 정책이 아니라 frontend의 egress 정책이 DNS를 막고 있는 것이다.

### 6.5 4단계: 정책 확인

```bash
kubectl get cnp -A
kubectl describe cnp -n default allow-frontend-to-backend-http
kubectl get pod -n default --show-labels
```

확인 포인트:

| 확인 항목 | 이유 |
|---|---|
| Pod label | `fromEndpoints`, `toEndpoints`, `endpointSelector`가 label과 일치해야 함 |
| namespace | 같은 label이라도 namespace가 다르면 정책 범위가 달라질 수 있음 |
| ingress/egress 방향 | 반대 방향에 정책을 썼을 수 있음 |
| port/protocol | TCP/UDP 또는 port 번호가 다를 수 있음 |
| DNS 허용 | egress default-deny에서 자주 빠짐 |

### 6.6 정책 디버깅 사고방식

문제가 생겼을 때 다음 순서로 생각하면 편하다.

```
1. 누가?        from-pod / source identity
2. 누구에게?    to-pod / destination identity
3. 어떤 문으로?  port / protocol
4. 결과는?      FORWARDED / DROPPED
5. 이유는?      Policy denied / DNS / TCP flags / service translation
```

Hubble은 이 다섯 가지를 한 줄에 가깝게 보여주기 때문에 정책 디버깅에 적합하다.

---

## 7. Observability

### 7.1 Observability란?

Observability는 시스템 내부에서 무슨 일이 일어나는지 외부 출력만 보고 추론할 수 있는 능력이다.

네트워크 observability에서는 다음 세 가지 질문에 답할 수 있어야 한다.

1. 현재 어떤 통신이 일어나고 있는가?
2. 어떤 통신이 실패하고 있으며, 왜 실패하는가?
3. 시간에 따라 트래픽과 drop이 어떻게 변하는가?

Hubble은 Cilium 환경에서 이 질문에 답하기 위한 도구다.

### 7.2 Hubble CLI 기반 관찰

실시간 관찰:

```bash
hubble observe --follow
```

drop만 관찰:

```bash
hubble observe --verdict DROPPED --follow
```

특정 namespace:

```bash
hubble observe --namespace default
```

특정 port:

```bash
hubble observe --port 80
```

특정 protocol:

```bash
hubble observe --protocol dns
hubble observe --protocol http
```

```bash
# 1. 먼저 drop만 본다.
hubble observe --verdict DROPPED

# 2. 문제 Pod 기준으로 좁힌다.
hubble observe --from-pod default/frontend

# 3. 목적지 기준으로 좁힌다.
hubble observe --to-pod default/backend

# 4. 필요한 경우 port/protocol까지 좁힌다.
hubble observe --to-pod default/backend --port 80
```

### 7.3 Hubble UI 기반 관찰

Hubble UI는 관계를 보는 데 강하다.

CLI가 다음 질문에 좋다면:

```
"이 요청이 왜 drop됐지?"
```

UI는 다음 질문에 좋다.

```
"우리 namespace 안에서 어떤 서비스들이 서로 연결되어 있지?"
```

Hubble UI에서 확인할 것:

| 화면 요소 | 의미 |
|---|---|
| 노드 | Pod, Service, workload |
| 선 | 통신 관계 |
| 색상/상태 | 정상/차단/오류 흐름 |
| flow list | 최근 개별 flow event |
| namespace selector | 관찰 범위 |

접속 예시:

```bash
cilium hubble ui
```

이 명령은 로컬 포트포워딩을 열고 브라우저에서 UI를 볼 수 있게 한다.

### 7.4 Metrics 기반 관찰

CLI와 UI는 "지금 무슨 일이 일어나는가"를 보기에 좋다. Metrics는 "시간에 따라 어떤 경향이 있는가"를 보기에 좋다.

예시:

| 지표 관점 | 보고 싶은 것 |
|---|---|
| flow 수 | 트래픽 양이 갑자기 늘었는가 |
| drop 수 | 정책 차단이나 오류가 증가했는가 |
| DNS query | DNS 요청이 비정상적으로 많아졌는가 |
| HTTP status | 5xx 응답이 증가했는가 |
| TCP flags | 연결 실패/SYN 재시도가 많아졌는가 |

Cilium metrics와 Hubble metrics는 분리해서 생각하면 좋다.

| 구분 | 초점 |
|---|---|
| Cilium Metrics | Cilium agent/operator/envoy 자체 상태 |
| Hubble Metrics | Cilium이 관리하는 Pod들의 네트워크 동작과 보안 이벤트 |

Prometheus와 Grafana를 붙이면 다음처럼 볼 수 있다.

```
Hubble flow events
     │
     ▼
Prometheus scrape
     │
     ▼
Grafana dashboard
     │
     ├─ namespace별 flow rate
     ├─ verdict별 drop rate
     ├─ protocol별 traffic
     └─ DNS/HTTP 관찰
```

### 7.5 Flow log와 metrics의 차이

| 구분 | Flow log | Metrics |
|---|---|---|
| 형태 | 개별 이벤트 | 집계된 숫자 |
| 예시 | `frontend -> backend DROPPED` | `drop/sec = 12` |
| 장점 | 원인 분석에 좋음 | 추세/알림에 좋음 |
| 단점 | 양이 많으면 보기 어려움 | 개별 요청 내용은 부족 |

운영에서는 둘 다 필요하다.

- 장애 순간 원인 분석: Hubble flow
- 평소 이상 징후 탐지: Hubble metrics

### 7.6 L7 visibility와 주의점

Hubble은 L7 정보도 보여줄 수 있지만, 기본 L3/L4 flow와는 다르게 생각해야 한다.

L7 visibility를 위해서는 보통 Cilium L7 policy가 필요하다. HTTP나 DNS 같은 애플리케이션 계층 정보를 보려면 트래픽이 Cilium proxy 경로로 들어와야 하기 때문이다.

예시:

```text
default/frontend -> default/backend http-request FORWARDED (GET /api/users)
default/frontend <- default/backend http-response FORWARDED (HTTP/1.1 200)
```

주의할 점:

- L7 policy는 visibility만 켜는 것이 아니라 허용/차단에도 영향을 줄 수 있다.
- HTTP URL, header, DNS query 같은 정보는 민감할 수 있다.
- 운영 환경에서는 redaction 설정과 접근 권한을 함께 고려해야 한다.

---

## 8. Hubble 활성화와 기본 확인

### 8.1 Cilium CLI로 활성화

```bash
cilium hubble enable
```

Hubble UI까지 같이 켜려면:

```bash
cilium hubble enable --ui
```

환경에 따라 Helm으로 설정할 수도 있다.

```bash
helm upgrade cilium cilium/cilium \
  --namespace kube-system \
  --reuse-values \
  --set hubble.relay.enabled=true \
  --set hubble.ui.enabled=true
```

### 8.2 상태 확인

```bash
cilium status
hubble status
```

확인할 것:

| 항목 | 정상 예시 |
|---|---|
| Cilium | `OK` |
| Hubble Relay | `OK` |
| Connected Nodes | 전체 노드 수와 일치 |
| Current/Max Flows | flow buffer가 꽉 차지 않는지 확인 |
| Flows/s | 현재 flow 발생량 |

### 8.3 접근 방식

로컬에서 Hubble Relay에 접근하려면 port-forward가 필요할 수 있다.

```bash
cilium hubble port-forward
```

또는 명령마다 port-forward 옵션을 사용할 수 있다.

```bash
hubble status -P
hubble observe -P
```

---

## 9. 요약

Hubble은 Cilium 환경에서 네트워크를 보이게 만드는 도구다.

- Hubble은 Cilium eBPF datapath에서 나온 flow event를 보여준다.
- Flow visibility는 IP/port뿐 아니라 Pod, namespace, label, identity, verdict까지 포함한다.
- L3 policy는 "누가 누구와 통신 가능한가"를 제어한다.
- L4 policy는 "어떤 port/protocol로 통신 가능한가"를 제어한다.
- 정책이 endpoint를 선택하면 해당 방향은 default-deny가 될 수 있다.
- Hubble CLI는 개별 문제 분석에 좋고, Hubble UI는 서비스 관계 파악에 좋다.
- Hubble metrics는 시간에 따른 트래픽과 drop 추세를 보는 데 좋다.

---

## 참고

- [Cilium Docs: Network Observability with Hubble](https://docs.cilium.io/en/stable/observability/hubble/)
- [Cilium Docs: Setting up Hubble Observability](https://docs.cilium.io/en/stable/observability/hubble/setup/)
- [Cilium Docs: Inspecting Network Flows with the CLI](https://docs.cilium.io/en/stable/observability/hubble/hubble-cli/)
- [Cilium Docs: Service Map & Hubble UI](https://docs.cilium.io/en/stable/observability/hubble/hubble-ui/)
- [Cilium Docs: Layer 3 Policies](https://docs.cilium.io/en/stable/security/policy/layer3/)
- [Cilium Docs: Layer 4 Policies](https://docs.cilium.io/en/stable/security/policy/layer4/)
- [Cilium Docs: Policy Enforcement Modes](https://docs.cilium.io/en/stable/security/policy/intro/)
- [Cilium Docs: Layer 7 Protocol Visibility](https://docs.cilium.io/en/stable/observability/visibility/)
- [Cilium Docs: Monitoring & Metrics](https://docs.cilium.io/en/stable/observability/metrics/)
