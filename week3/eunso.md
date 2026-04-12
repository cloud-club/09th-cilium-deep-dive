## 1. Calico 개요

Calico는 컨테이너, VM, 베어메탈 워크로드를 위한 오픈소스 네트워킹 및 네트워크 보안 솔루션이다. Kubernetes CNI(Container Network Interface) 플러그인으로 가장 널리 쓰이는 선택지 중 하나다.

### 핵심 설계 철학

- Pure L3 네트워킹: 가능하면 오버레이 없이 표준 IP 라우팅으로 패킷을 전달한다.
- BGP 기반 라우팅: 각 노드가 BGP speaker가 되어 Pod CIDR을 광고한다.
- iptables/eBPF 기반 정책: 커널 수준에서 NetworkPolicy를 강제한다.

### 주요 컴포넌트

| 컴포넌트 | 역할 |
|---|---|
| `calico-node` | 각 노드에서 실행되는 DaemonSet. Felix, BIRD, confd 포함 |
| `Felix` | 라우팅 테이블, iptables/eBPF 규칙을 실제로 프로그래밍하는 에이전트 |
| `BIRD` | BGP 데몬. 라우팅 정보를 다른 노드/라우터와 교환 |
| `confd` | etcd/Kubernetes API에서 설정을 읽어 BIRD 설정 파일을 렌더링 |
| `calico-kube-controllers` | Kubernetes 리소스 변경을 감지해 Calico 데이터스토어에 동기화 |
| `Typha` (선택) | Felix와 데이터스토어 사이의 팬아웃 프록시. 대규모 클러스터에서 API 서버 부하 감소 |
| `calicoctl` | CLI 관리 도구 |

---

## 2. 네트워크 기초: Overlay vs Underlay

![Overlay vs Underlay](https://media.licdn.com/dms/image/v2/D4D12AQGe6dMDiTOT3A/article-inline_image-shrink_1500_2232/article-inline_image-shrink_1500_2232/0/1709361489085?e=1776902400&v=beta&t=gnJJz1ezDm-Hn99x6YV-6Z_OSyXPh_7zydQw5Hnvw2c)

### Underlay 네트워킹

물리적(혹은 가상) 네트워크 인프라 자체를 그대로 활용하는 방식이다.

**특징**
- 패킷에 추가 헤더가 없으므로 오버헤드 최소
- 물리 네트워크가 Pod CIDR을 알아야 함 (BGP로 광고하거나 정적 라우팅)
- 낮은 레이턴시, 높은 처리량
- 네트워크 가시성이 높음 (패킷 캡처가 쉬움)

**Calico에서 Underlay 사용 조건**
- 노드 간 L2 연결이 있거나 (같은 서브넷)
- 물리 라우터에 BGP peering이 가능하거나
- 노드 간 라우팅 정보를 공유할 수단이 있을 때

### Overlay 네트워킹

기존 IP 패킷을 다른 IP 패킷 안에 캡슐화해서 전달하는 방식이다.

**Calico가 지원하는 오버레이 방식**

| 방식 | 설명 | 특징 |
|---|---|---|
| **VXLAN** | UDP/8472로 캡슐화. L3 네트워크 위에서 동작 | BGP 불필요. MTU 오버헤드 50bytes |
| **IP-in-IP** | IP 헤더를 IP 헤더로 감쌈. 프로토콜 4번 | 오버헤드 20bytes. L3 라우팅 필요 |

**언제 Overlay를 써야 하나?**
- 물리 네트워크 변경 권한이 없을 때 (퍼블릭 클라우드, 공유 인프라)
- 다른 서브넷에 있는 노드들이 Pod IP를 직접 라우팅할 수 없을 때
- 네트워크 격리가 필요한 멀티테넌트 환경

### MTU 고려사항

| 모드 | 권장 MTU |
|---|---|
| Underlay (순수 라우팅) | 1500 (표준) |
| IP-in-IP | 1480 (IP 헤더 20bytes 제외) |
| VXLAN | 1450 (VXLAN 헤더 50bytes 제외) |
| WireGuard 암호화 | 1440 |

MTU 불일치는 패킷 단편화나 드롭을 유발하므로, `FelixConfiguration`의 `mtu` 또는 `vxlanMTU` 필드를 명시적으로 설정하는 것이 좋다.

---

## 3. BGP (Border Gateway Protocol)

### BGP란?

BGP는 인터넷을 구성하는 AS(Autonomous System) 간에 라우팅 정보를 교환하기 위한 프로토콜이다 (RFC 4271). Calico는 이를 클러스터 내부 라우팅에 활용한다.

### 핵심 개념

**AS (Autonomous System)**
- 단일 관리 도메인 하의 네트워크 집합
- AS Number (ASN)으로 식별: 16비트(1-65535) 또는 32비트
- 사설 ASN 범위: 64512-65534 (16비트), 4200000000-4294967294 (32비트)

**iBGP vs eBGP**

| 구분 | iBGP (Internal BGP) | eBGP (External BGP) |
|---|---|---|
| 위치 | 같은 AS 내부 | 서로 다른 AS 간 |
| TTL | 255 (기본값) | 1 (직접 연결된 피어만) |
| Next-hop | 변경 안 함 | 피어 자신으로 변경 |
| Loop prevention | Full-mesh 또는 Route Reflector | AS_PATH attribute |
| Calico 사용 | 노드간 기본 방식 | ToR 스위치 peering |

**BGP Attributes (경로 선택에 사용)**

```
LOCAL_PREF  → AS 내부 경로 선호도 (높을수록 선호)
AS_PATH     → 경유한 AS 목록 (짧을수록 선호)
MED         → Multi-Exit Discriminator, 외부 AS에 경로 힌트 제공
NEXT_HOP    → 패킷을 보낼 다음 홉 주소
COMMUNITY   → 정책 태깅용 메타데이터
```

### Calico에서 BGP 동작 방식

#### 기본 모드: Node-to-Node Mesh

모든 노드가 서로 iBGP peering을 맺는다.

```
Node A (ASN 64512)
  ↕ BGP
Node B (ASN 64512)
  ↕ BGP
Node C (ASN 64512)
  ↕ BGP
... (n*(n-1)/2 connections)
```

- 노드 수가 적을 때(< 100) 적합
- 노드 추가 시 자동으로 peering 설정
- 노드 수 증가에 따라 O(n²) 연결 수

#### 확장 모드: Route Reflector

대규모 클러스터에서는 Route Reflector(RR)를 사용한다.

![RR](https://www.tigera.io/app/uploads/2019/03/route-reflection.png)

- 각 클라이언트 노드는 RR에만 peering
- RR이 학습한 경로를 모든 클라이언트에 반영(reflect)
- 연결 수: O(n) — 대규모에 적합

#### External BGP Peering (ToR 스위치 연동)

물리 네트워크의 ToR(Top-of-Rack) 스위치와 eBGP peering을 맺어 Pod CIDR을 실제 네트워크에 광고할 수 있다.

```
     인터넷/데이터센터 코어
            │
    ┌───────┴───────┐
    │  ToR Switch   │  ← eBGP (ASN 65000)
    └───────┬───────┘
            │
    ┌───────┴───────┐
    │  Kubernetes   │  ← eBGP (ASN 64512)
    │  Nodes        │
    └───────────────┘
```

```yaml
apiVersion: projectcalico.org/v3
kind: BGPPeer
metadata:
  name: tor-switch
spec:
  peerIP: 192.168.0.1    # ToR 스위치 IP
  asNumber: 65000        # ToR의 ASN
```

이 설정으로 Pod IP가 외부 네트워크에서 직접 라우팅 가능해진다.

---

## 4. Calico의 아키텍처

### 컨트롤 플레인과 데이터 플레인

![Calico Architecture](https://docs.tigera.io/assets/images/architecture-calico-deae813300e472483f84d6bfb49650ab.svg)

### Felix 상세

Felix는 Calico의 핵심 에이전트로, 데이터스토어에서 정책을 읽어 커널에 직접 적용한다.

**Felix의 주요 작업**
1. **인터페이스 관리**: veth pair 생성, 노드의 라우팅 테이블 업데이트
2. **ACL 프로그래밍**: NetworkPolicy를 iptables/eBPF 규칙으로 변환
3. **라우팅 정보 배포**: BIRD에게 로컬 워크로드 경로 전달
4. **헬스체크**: 데이터플레인 상태 모니터링 및 보고

**Felix의 데이터 흐름**

```
Kubernetes API / etcd
        ↓
  Felix (Watch & Reconcile)
        ↓ 변환
  iptables chains / eBPF maps
        ↓ 적용
  커널 네트워킹 스택
```

---

## 5. Calico 데이터 플레인

### iptables 기반 데이터플레인

전통적인 Calico의 방식으로, Linux iptables를 통해 정책을 구현한다.

**Chain 구조**

```
INPUT/OUTPUT/FORWARD (기본 체인)
        ↓
cali-INPUT / cali-OUTPUT / cali-FORWARD  (Calico 진입점)
        ↓
cali-wl-to-host / cali-from-wl-dispatch  (워크로드 방향별)
        ↓
cali-tw-<interface-id>  (인터페이스별 정책)
        ↓
cali-pi-<policy-name>   (개별 정책 규칙)
```

**iptables의 한계**
- 규칙이 많아질수록 선형 탐색 비용 증가
- 1만 개 이상 정책에서 성능 저하 뚜렷
- 규칙 업데이트 시 전체 테이블 재로드 필요 (원자성 문제)

### eBPF 기반 데이터플레인

Calico v3.13부터 eBPF 데이터플레인을 지원한다.

**eBPF의 장점**
- 패킷 처리를 XDP/TC 훅에서 수행 → iptables 우회
- 해시맵 기반 정책 조회 → O(1) 룩업
- DSR(Direct Server Return) 지원으로 kube-proxy 대체 가능
- 커넥션 추적(conntrack)을 eBPF 맵으로 관리

### Windows HNS 데이터플레인

Windows 노드에서는 Host Networking Service(HNS)를 통해 정책을 구현한다. 기능 제약이 있으므로 혼합 OS 클러스터에서는 주의가 필요하다.

---

## 6. Network Policy 적용

### Kubernetes NetworkPolicy vs Calico NetworkPolicy

| 기능 | Kubernetes NetworkPolicy | Calico NetworkPolicy | Calico GlobalNetworkPolicy |
|---|---|---|---|
| 네임스페이스 범위 | 네임스페이스 내 | 네임스페이스 내 | 클러스터 전체 |
| 우선순위 | 없음 (모두 동등) | 있음 (order 필드) | 있음 |
| 서비스 계정 기반 선택 | 불가 | 가능 | 가능 |
| ICMP 타입/코드 매칭 | 불가 | 가능 | 가능 |
| HTTP 메서드/경로 매칭 | 불가 | 가능 (L7) | 가능 (L7) |
| 노드 대상 정책 | 불가 | 불가 | 가능 (HostEndpoint) |
| 로그 액션 | 불가 | 가능 | 가능 |
| Pass 액션 | 불가 | 가능 | 가능 |

### 정책 평가 순서

Calico의 정책은 `order` 값에 따라 순서대로 평가된다.

```
낮은 order 값 = 먼저 평가 (높은 우선순위)

GlobalNetworkPolicy (order: 100)  ← 먼저 평가
GlobalNetworkPolicy (order: 200)
NetworkPolicy (order 없음)         ← 다음 평가
GlobalNetworkPolicy (order 없음)   ← 마지막
```

**액션 종류**
- `Allow`: 트래픽 허용
- `Deny`: 트래픽 차단 (연결 거부)
- `Pass`: 이 정책에서 판단을 포기하고 다음 정책으로 넘김
- `Log`: 로그만 남기고 다음 규칙으로 계속

### 기본 거부(Default Deny) 정책

Kubernetes NetworkPolicy는 선택된 Pod에 정책이 하나라도 있으면 나머지는 묵시적으로 거부된다. 명시적 기본 거부는 다음과 같이 설정한다.

```yaml
# 네임스페이스 전체 기본 거부 (Kubernetes)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}    # 모든 Pod 선택
  policyTypes:
  - Ingress
  - Egress
```

```yaml
# 클러스터 전체 기본 거부 (Calico GlobalNetworkPolicy)
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny
spec:
  order: 1000        # 높은 order = 나중에 평가 (기본값으로 작동)
  selector: all()
  types:
  - Ingress
  - Egress
```

### L7 정책 (HTTP)

Calico Enterprise / Calico Cloud에서 지원. 오픈소스에서는 Istio 연동으로 구현 가능.

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-get-only
spec:
  selector: app == 'api-server'
  ingress:
  - action: Allow
    http:
      methods: ["GET"]
      paths:
      - exact: "/api/v1/status"
      - prefix: "/api/v1/public/"
```

---

## 7. IP Address Management (IPAM)

### Calico IPAM의 특징

Calico는 자체 IPAM 구현(`calico-ipam`)을 제공하며, host-local IPAM보다 유연하다.

**IP 할당 방식: Block 기반**

```
전체 Pod CIDR: 10.244.0.0/16
        │
        ├── Node A: 10.244.0.0/26 (블록, 64개 IP)
        ├── Node B: 10.244.0.64/26
        ├── Node C: 10.244.0.128/26
        └── ...
```

- 기본 블록 크기: `/26` (64 IPs)
- 노드에 블록이 부족하면 추가 블록 요청
- 블록은 특정 노드에 '예약'되지만, 다른 노드가 필요하면 빌릴 수 있음(IP borrowing)

**IP Borrowing (IP 차용)**

한 노드의 블록이 꽉 차면 다른 노드에 할당된 블록에서 IP를 빌릴 수 있다.
이 경우 해당 Pod의 트래픽은 블록 소유 노드를 거쳐 라우팅될 수 있으므로 성능에 영향을 줄 수 있다.

---

## 8. Calico vs Cilium 비교

| 항목 | Calico | Cilium |
|---|---|---|
| **기반 기술** | iptables (기본) / eBPF (선택) | eBPF (기본) |
| **BGP 지원** | 네이티브 (BIRD 내장) | 없음 (별도 구성 필요) |
| **L7 정책** | 엔터프라이즈 / Istio 연동 | 오픈소스에서 기본 지원 |
| **서비스 메시** | 없음 (별도) | Cilium Service Mesh 내장 |
| **관찰가능성** | 기본적 | Hubble (풍부한 L7 관찰) |
| **Network Policy** | Kubernetes + Calico 확장 | Kubernetes + CiliumNetworkPolicy |
| **성숙도** | 매우 높음 (오래됨) | 높음 (빠르게 성장) |
| **복잡도** | 비교적 낮음 | 비교적 높음 |
| **BGP 기반 온프레미스** | 강점 | 약점 |
| **투명한 암호화** | WireGuard | WireGuard / IPsec |
| **멀티클러스터** | Calico Federation | Cluster Mesh |

**선택 기준**
- BGP가 필요하고 기존 네트워크 인프라와 통합해야 한다면: **Calico**
- eBPF를 최대한 활용하고 서비스 메시, L7 관찰이 필요하다면: **Cilium**
- 단순하고 안정적인 CNI가 필요하다면: **Calico (iptables 모드)**

## 추가 학습 참고자료

- [Calico 공식 문서](https://docs.tigera.io/calico/latest/about/)
- [Calico BGP 구성 가이드](https://docs.tigera.io/calico/latest/networking/configuring/bgp)
- [Calico eBPF 데이터플레인](https://docs.tigera.io/calico/latest/operations/ebpf/)
- [Project Calico GitHub](https://github.com/projectcalico/calico)
