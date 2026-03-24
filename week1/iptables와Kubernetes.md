## iptables란 무엇인가

패킷을 검사하고 어떻게 처리할지 결정하는 규칙 엔진으로, 네트워크 패킷이 커널을 통과할 때 필터링, NAT, 라우팅 결정 같은 작업을 수행한다.

### 기본 역할

패킷이 네트워크 인터페이스(NIC)를 통해 들어오면 커널에서 다음과 같은 판단을 한다.

- 이 패킷을 허용할 것인가
- 어디로 보낼 것인가
- 주소를 바꿀 것인가(NAT)
- 차단할 것인가

iptables는 이러한 판단을 rule 기반으로 수행한다.

rule 기반으로 source IP 확인, destination port 확인 등을 거쳐 ACCEPT, DROP 등의 행동을 결정한다.

---

### iptables의 패킷 처리 방식

iptables는 chain 기반 흐름으로 rule을 따라가며 패킷을 처리한다.

패킷이 NIC에서 들어오면 커널의 Netfilter hook에서 iptables가 실행된다.

대략적인 흐름은 다음과 같다.

```yaml
# 외부 -> 호스트
NIC
 ↓
PREROUTING # hook
 ↓
routing decision
 ↓
INPUT / FORWARD # hook
 ↓
POSTROUTING # hook -> forward의 경우에만(input x)

# 호스트 -> 외부
Application
 ↓
OUTPUT # hook
 ↓
routing decision
 ↓
POSTROUTING # hook
 ↓
NIC
```

각 Netfilter hook 지점마다 여러 table이 적용되고, 각 table에는 해당 hook에 연결된 chain이 존재하며,

```bash
PREROUTING (hook)
 ├─ raw table → PREROUTING chain
 ├─ mangle table → PREROUTING chain
 └─ nat table → PREROUTING chain
```

 chain 내부에는 여러 rule이 있다.

```yaml
table
 ├─ chain
 │   ├─ rule
 │   ├─ rule
 │   └─ rule
```

rule에서는 다음과 같은 동작이 수행된다.

```yaml
ACCEPT # 패킷 처리 바로 끝남
DROP # 패킷 처리 바로 끝남
JUMP # 특정 chain으로 이동 -> jump한 chain에서 결정이 나지 않으면 원래 chain으로 돌아옴
DNAT
SNAT
```

### tables

table을 좀 더 자세히 살펴보겠다.

```bash
PREROUTING (hook)
 ├─ raw table → PREROUTING chain
 ├─ mangle table → PREROUTING chain
 └─ nat table → PREROUTING chain
```

여기서 table은 패킷 처리 목적별로 rule을 나눈 그룹이다.

**주요 table 4개**

1. raw table
    
    가장 먼저 실행되며, conntrack(연결 상태 추적) 적용 여부를 결정한다. 
    
    → 이 패킷에 대해 상태 추적할 건지 말 건지 결정하며, 이후 NAT/filter에서 상태 기반 처리를 할 수 있다.
    
2. mangle table
    
    패킷 수정 용도로 TTL 변경 등 패킷 헤더를 조작한다.
    
3. nat table
    
    IP/Port를 변경한다.
    
    - PREROUTING 시 목적지를 바꾸고(DNAT),
    - POSTROUTING 시 출발지를 바꿈(SNAT)
4. filter table
    
    허용/차단 여부를 결정하는 기본 방화벽 역할을 한다.
    

**hook별 table 적용**

아래 표와 같이 각 hook마다 적용되는 table이 다르다.

| Hook | raw | mangle | nat | filter |
| --- | --- | --- | --- | --- |
| PREROUTING | O | O | O (DNAT) | X |
| INPUT | X | O | X | O |
| FORWARD | X | O | X | O |
| OUTPUT | O | O | O (DNAT 가능) | O |
| POSTROUTING | X | O | O (SNAT) | X |

---

## Kubernetes와 iptables

### Kubernetes에서의 iptables rule 생성

Kubernetes에서 iptables rule이 생성되는 건 다음 4가지가 대표적이다.

- Service routing - PREROUTING hook (nat table)
- NodePort/ LoadBalancer - PREROUTING hook (nat table)
- Pod egress NAT - POSTROUTING hook (nat table)
- NetworkPolicy - FORWARD/INPUT/OUTPUT hook (filter table)

이때 Service routing과 NodePort/LoadBalancer 같은 Service 관련 rule은 kube-proxy가 담당하고, Pod networking과 policy rule은 CNI plugin이 담당한다.

1. **Service routing**

예를 들어, 다음과 같은 Service가 있으면

```yaml
Service IP: 10.96.0.10
Endpoints:
  Pod1
  Pod2
```

iptables에 다음과 같은 rule을 생성한다.

```yaml
10.96.0.10 → Pod1
10.96.0.10 → Pod2
```

1. NodePort/ LoadBalancer

예를 들어, 다음과 같은 NodePort 유형의 Service가 있다면

```yaml
NodeIP:30007 → Service → Pod
```

이에 맞게 외부 트래픽을 해당하는 Service로 전달한다.

1. Pod egress NAT

Pod IP는 보통 cluster 내부 IP로 외부에서 접근이 불가능하다.

```yaml
Pod → Internet
↓
SNAT
↓
Node IP
```

따라서 외부로 나가는 패킷은 SNAT을 통해 Node IP로 변한된다.

1. NetworkPolicy 

예를 들어, 다음과 같은 NetworkPolicy가 있다면

```yaml
Pod A → Pod B 허용
Pod C → Pod B 차단
```

CNI plugin에 의해 해당하는 iptables rule로 구현된다.

이런 방식으로 Kubernetes에서는 kube-proxy가 Service를 위해 여러 chain을 생성하고, Service 개수가 많아지면 rule이 폭발적으로 증가하게 된다.

예를 들어 Service 요청이 들어오면 패킷은 다음과 같은 chain을 거친다.

```yaml
packet
 ↓
PREROUTING chain
 ↓
KUBE-SERVICES chain # 모든 서비스 entrypoint -> 서비스 매칭
 ↓
KUBE-SVC-XXXXX # 특정 Service -> 로드밸런싱(Pod 매칭)
 ↓
KUBE-SEP-XXXXX # 특정 Pod -> DNAT (실제 목적지 IP로 변경)
 ↓
endpoint pod
```

이처럼 패킷은 rule 검사 → chain 이동→ rule 검사 → chain 이동의 흐름을 반복하며 처리된다.

### iptables가 느려지는 이유

iptables의 rule 매칭은 기본적으로 순차 검사(O(n)) 방식이다.

즉, rule이 다음과 같이 있다면

```yaml
rule1
rule2
rule3
...
ruleN
```

패킷은 앞에서부터 rule을 검사한다.

따라서 rule 수가 증가할수록 검사 비용이 증가한다.

또한 Kubernetes에서는 Service와 Endpoint(Service backend pod)가 많아지면서 chain이 많이 생성되고 패킷이 여러 chain을 이동하며 rule을 검사하게 된다.

결과적으로 패킷 하나가 처리되는 동안 많은 rule + 많은 chain을 거치게 되며 지연이 증가할 수 있다.

이러한 문제를 해결하기 위해 IPVS나 eBPF 같은 순차 탐색(O(n))이 아닌 lookup으로 빠르게 rule을 적용할 수 있는 방식이 나오게 되었다.