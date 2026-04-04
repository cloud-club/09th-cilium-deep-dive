# Week 2: kube-proxy 딥다이브

## 1. kube-proxy 개요

### 1.1 역할

kube-proxy는 각 노드에서 실행되는 DaemonSet으로, Kubernetes Service 오브젝트를 실제 네트워크 규칙으로 변환하는 역할을 한다.

```
[API Server]
    │  Service/Endpoint 변경 감지 (watch)
    ▼
[kube-proxy]
    │  노드별 네트워크 규칙 생성/갱신
    ▼
[iptables rules / IPVS virtual server]
    │  패킷이 VIP에 도달 시 실제 Pod IP로 DNAT
    ▼
[Pod]
```

핵심 책임
- Service의 ClusterIP (VIP) → 실제 Pod IP:Port로 DNAT
- NodePort 트래픽 처리
- 로드밸런싱 (다수의 Pod 엔드포인트 분산)
- 헬스체크 실패한 엔드포인트 제거

### 1.2 감시 대상 리소스

kube-proxy는 API Server를 watch하여 다음 리소스 변경에 반응한다.

| 리소스 | 용도 |
|--------|------|
| `Service` | ClusterIP, NodePort, 포트 정보 |
| `EndpointSlice` (v1.21+) | Service에 연결된 실제 Pod IP/Port 목록 |
| `Node` | 노드 IP 정보 (NodePort 바인딩) |

v1.21 이전에는 `Endpoints` 오브젝트를 사용했으나, 대규모 클러스터에서의 성능 문제로 `EndpointSlice`로 이전. EndpointSlice는 최대 100개의 엔드포인트를 하나의 오브젝트로 슬라이스하여 API Server 부하를 분산시킨다.

### 1.3 동작 모드

| 모드 | 도입 | 특징 |
|------|------|------|
| `userspace` | v1.0 | 초기 구현, kube-proxy 프로세스가 직접 패킷 처리. 성능 최악 (deprecated) |
| `iptables` | v1.2 | Netfilter 규칙으로 커널 내 처리. 현재 기본값 |
| `ipvs` | v1.11 (stable) | Linux IPVS 기반. 대규모 환경에 유리 |
| `nftables` | v1.31 (alpha) | iptables의 후속 기술 nftables 기반. 실험적 |

---

## 2. iptables 모드

### 2.1 동작 원리

iptables 모드에서 kube-proxy는 **nat 테이블**에 체인을 삽입하여 Service VIP로 향하는 패킷을 실제 Pod IP로 DNAT한다. 커널 Netfilter가 패킷을 처리하므로 userspace를 거치지 않는다.

```
패킷: src=client, dst=ClusterIP:port
  │
  ▼
[PREROUTING hook] → nat 테이블
  │  KUBE-SERVICES chain 진입
  ▼
  KUBE-SVC-XXXXXXXX (해당 Service chain)
  │  확률적 분기 (statistic module, --probability)
  ├─ 33% → KUBE-SEP-AAA (Pod 1의 DNAT rule)
  ├─ 50% → KUBE-SEP-BBB (Pod 2의 DNAT rule)
  └─ 100% → KUBE-SEP-CCC (Pod 3의 DNAT rule)
  │
  ▼
DNAT: dst = 10.244.1.5:8080 (실제 Pod IP:Port)
  │
  ▼
conntrack에 NAT 정보 기록 → 이후 응답 패킷 자동 역변환
```

### 2.2 체인 구조

kube-proxy가 생성하는 iptables 체인 계층:

```
nat 테이블의 기존 PREROUTING chain
  └─ -j KUBE-SERVICES  ← kube-proxy가 삽입

KUBE-SERVICES
  ├─ -d 10.96.0.1/32 -p tcp --dport 443  -j KUBE-SVC-NPX46M4PTMTKRN6Y  (kubernetes svc)
  ├─ -d 10.96.10.20/32 -p tcp --dport 80  -j KUBE-SVC-XXXXXXXXXXX       (my-service)
  └─ -j KUBE-NODEPORTS  (NodePort 처리)

KUBE-SVC-XXXXXXXXXXX  (my-service, 엔드포인트 3개)
  ├─ -m statistic --mode random --probability 0.33333  -j KUBE-SEP-AAA
  ├─ -m statistic --mode random --probability 0.50000  -j KUBE-SEP-BBB
  └─ -j KUBE-SEP-CCC

KUBE-SEP-AAA  (Pod 1: 10.244.1.5:8080)
  ├─ -s 10.244.1.5/32 -j KUBE-MARK-MASQ  ← Pod 자신이 출발지일 때 masquerade 마킹
  └─ -p tcp -j DNAT --to-destination 10.244.1.5:8080
```

핵심 체인 요약

| 체인 | 역할 |
|------|------|
| `KUBE-SERVICES` | Service IP 매칭 진입점 |
| `KUBE-SVC-*` | 서비스별 로드밸런싱 (확률적 분기) |
| `KUBE-SEP-*` | 개별 엔드포인트 DNAT |
| `KUBE-NODEPORTS` | NodePort 처리 진입점 |
| `KUBE-MARK-MASQ` | MASQUERADE 필요 패킷에 마킹 (0x4000) |
| `KUBE-POSTROUTING` | 마킹된 패킷 MASQUERADE 적용 |

### 2.3 로드밸런싱 구현 방식

확률 기반 랜덤 분기로 균등 분산을 구현한다.

엔드포인트가 N개일 때, i번째 엔드포인트(0-indexed)의 확률:
```
P(i) = 1 / (N - i)
```

예시 (3개 엔드포인트):
```
1번째 규칙: probability = 1/3 ≈ 0.333   → Pod 1
2번째 규칙: probability = 1/2 = 0.500   → Pod 2 (1번 통과한 패킷의 절반)
3번째 규칙: probability = 1/1 = 1.000   → Pod 3 (나머지 전부)

결과: 각 Pod에 1/3 확률로 균등 분배
```

이 방식은 **stateless** 로드밸런싱이다. 동일 클라이언트의 연속 요청이 다른 Pod로 갈 수 있다. <br>
sticky session이 필요하면 Service `sessionAffinity: ClientIP` 설정으로 conntrack 기반 고정을 사용한다.

### 2.4 NodePort 구현

```
외부 트래픽: dst=NodeIP:30080

[PREROUTING]
  │
  ▼
KUBE-SERVICES → KUBE-NODEPORTS
  │
  ▼
  -p tcp --dport 30080 -j KUBE-SVC-XXXXXXXXXXX
  │                        (이후 동일: KUBE-SEP-* → DNAT)
  ▼
KUBE-MARK-MASQ로 마킹 (다른 노드의 Pod로 전달될 경우 SNAT 필요)
  │
  ▼
[POSTROUTING]
  KUBE-POSTROUTING: 마킹된 패킷 -j MASQUERADE
```

`externalTrafficPolicy: Local` 설정 시, 해당 노드에 Pod가 없으면 패킷을 DROP한다.
로컬 Pod로만 전달하기 때문에 원본 클라이언트 IP가 보존된다.

### 2.5 iptables 모드의 성능 한계

**O(n) 선형 규칙 순회:**

```
Service 수 = S, 엔드포인트 수 = E 라 하면
생성되는 iptables rule 수 ≈ O(S × E)

Service 1000개, 평균 10개 엔드포인트 → 약 10,000개 규칙
패킷마다 최악의 경우 10,000개 규칙을 순회
```

전체 규칙 갱신 문제

iptables는 원자적 업데이트를 위해 `iptables-restore`로 전체 테이블을 재작성한다.
엔드포인트 하나가 변경되어도 전체 nat 테이블을 덤프 → 수정 → 복원하는 과정이 필요하다.
대규모 클러스터에서 이 오버헤드가 수 초에 달할 수 있다.

conntrack 테이블 고갈

모든 연결이 conntrack 엔트리를 소비한다.
기본값 `nf_conntrack_max = 131072` 는 고트래픽 서비스에서 쉽게 고갈될 수 있다.

---

## 3. IPVS 모드

### 3.1 Linux IPVS(IP Virtual Server) 란?

IPVS는 Linux 커널에 내장된 L4 로드밸런서로, LVS(Linux Virtual Server) 프로젝트의 일부다.
Netfilter의 `LOCAL_IN` hook에 등록되어 가상 서버(Virtual Server)로 들어오는 패킷을 실제 서버(Real Server)로 전달한다.

```
IPVS 구조:
Virtual Server (VS): 10.96.10.20:80  ← Service ClusterIP
  ├─ Real Server 1: 10.244.1.5:8080  ← Pod 1
  ├─ Real Server 2: 10.244.2.3:8080  ← Pod 2
  └─ Real Server 3: 10.244.3.7:8080  ← Pod 3
```

### 3.2 IPVS 모드에서 kube-proxy 동작

```
[kube-proxy (IPVS 모드)]
  │
  ├─ ipset으로 Service IP 집합 관리
  ├─ dummy 인터페이스(kube-ipvs0)에 ClusterIP 바인딩
  │    → 커널이 해당 IP를 로컬로 인식하게 함
  ├─ ipvsadm으로 Virtual Server / Real Server 등록
  └─ 최소한의 iptables 규칙 (MASQUERADE, REJECT 등)
```

dummy 인터페이스가 필요한 이유
IPVS는 `LOCAL_IN` hook에서 동작하므로, 커널이 ClusterIP를 로컬 주소로 인식해야 한다.<br>
`kube-ipvs0` dummy 인터페이스에 모든 ClusterIP를 바인딩하면 라우팅 결정에서 `local` 로 분류된다.

```bash
# kube-ipvs0 인터페이스 예시
ip addr show kube-ipvs0
# 10.96.0.1/32 (kubernetes svc)
# 10.96.10.20/32 (my-service)
# 10.96.10.21/32 (another-service)
# ...
```

### 3.3 IPVS 스케줄링 알고리즘

iptables 모드의 랜덤 확률 분기와 달리, IPVS는 다양한 스케줄링 알고리즘을 지원한다.

| 알고리즘 | 약어 | 설명 |
|----------|------|------|
| Round Robin | `rr` | 순서대로 순환 (기본값) |
| Least Connection | `lc` | 현재 연결 수가 가장 적은 서버 |
| Destination Hash | `dh` | 목적지 IP 기반 해시 (캐시 친화적) |
| Source Hash | `sh` | 출발지 IP 기반 해시 (sticky session 유사) |
| Weighted Round Robin | `wrr` | 가중치 기반 순환 |
| Shortest Expected Delay | `sed` | 예상 응답시간 최소 서버 |

kube-proxy에서 설정:
```yaml
# kube-proxy ConfigMap
apiVersion: kubeproxy.config.k8s.io/v1alpha1
kind: KubeProxyConfiguration
mode: "ipvs"
ipvs:
  scheduler: "rr"   # 기본값: rr
```

### 3.4 패킷 흐름 (IPVS 모드)

```
패킷: src=client, dst=10.96.10.20:80 (ClusterIP)
  │
  ▼
[PREROUTING hook] → iptables nat (MASQUERADE 마킹만)
  │
  ▼
[routing decision]
  │  kube-ipvs0가 10.96.10.20을 로컬로 소유
  │  → LOCAL_IN 경로로 라우팅
  ▼
[LOCAL_IN hook] → IPVS 처리
  │  Virtual Server 10.96.10.20:80 매칭
  │  스케줄러가 Real Server 선택: 10.244.2.3:8080
  │  DNAT 적용 + IPVS 세션 테이블에 기록
  ▼
패킷 재전달 (ip_forward 경로로)
  │
  ▼
[FORWARD hook] → 목적지 Pod로 전달
```

### 3.5 IPVS 세션 테이블

IPVS는 자체 세션 테이블을 유지하여 연결 고정(persistence)을 구현한다. <br>
conntrack과 별개로 동작하며 해시 기반 O(1) 룩업이 가능하다.

```bash
# IPVS 가상 서버 목록 확인
ipvsadm -Ln

# 출력 예시:
# IP Virtual Server version 1.2.1 (size=4096)
# Prot LocalAddress:Port Scheduler Flags
#   -> RemoteAddress:Port    Forward Weight ActiveConn InActConn
# TCP  10.96.10.20:80 rr
#   -> 10.244.1.5:8080       Masq    1      5          0
#   -> 10.244.2.3:8080       Masq    1      3          0
#   -> 10.244.3.7:8080       Masq    1      4          0
```

---

## 4. iptables vs IPVS 비교

### 4.1 성능 비교

| 항목 | iptables | IPVS |
|------|----------|------|
| 룰 매칭 복잡도 | O(n) - 선형 순회 | O(1) - 해시 테이블 |
| 규칙 업데이트 | 전체 테이블 재작성 | 증분 업데이트 (개별 VS/RS) |
| 대규모 클러스터 | Service 수 증가 시 지수적 지연 | Service 수에 무관하게 일정 |
| 메모리 사용 | 규칙 수에 비례 | 세션 테이블 별도 유지 |
| 커널 자원 | conntrack 의존 | 자체 세션 테이블 + conntrack |

### 4.2 기능 비교

| 기능 | iptables | IPVS |
|------|----------|------|
| 로드밸런싱 알고리즘 | 랜덤 확률만 지원 | rr, lc, dh, sh, wrr 등 다수 |
| 세션 고정 | conntrack + sessionAffinity | 자체 persistence + sessionAffinity |
| 헬스체크 연동 | kube-proxy가 규칙 재작성 | kube-proxy가 Real Server 가중치 조정 |
| DSCP/TOS 처리 | mangle table 별도 설정 | 제한적 |
| 디버깅 도구 | iptables-save, conntrack | ipvsadm, conntrack |

### 4.3 IPVS 모드의 주의사항

IPVS 모드에서도 일부 iptables 규칙은 여전히 필요하다.

- **MASQUERADE:** NodePort에서 타 노드 Pod로 전달 시 SNAT
- **REJECT/DROP:** 포트 미사용 트래픽 거부
- **NodePort 진입:** IPVS가 NodePort를 처리하기 위한 마킹

따라서 IPVS 모드는 iptables를 완전히 제거하지 않고, 최소한의 규칙만 유지한다.

---

## 5. Service 타입별 구현

### 5.1 ClusterIP

클러스터 내부 통신 전용 가상 IP.

```
클러스터 내 Pod A → ClusterIP:Port

[PREROUTING / OUTPUT hook에서 DNAT]
  VIP는 실제 인터페이스에 존재하지 않음
  iptables: KUBE-SERVICES에서 매칭 → DNAT
  IPVS: kube-ipvs0에 VIP 바인딩 → LOCAL_IN에서 처리
```

### 5.2 NodePort

모든 노드의 특정 포트로 외부 트래픽 수신.

```
외부 → NodeIP:NodePort (30000-32767)

[PREROUTING hook]
  iptables: KUBE-NODEPORTS chain → KUBE-SVC-* → DNAT
  IPVS: NodePort용 Virtual Server 별도 생성

externalTrafficPolicy 영향:
  Cluster (기본): 모든 노드로 수신, 타 노드 Pod로 SNAT 전달 → 원본 IP 손실
  Local: 해당 노드 Pod에만 전달, 원본 IP 보존, Pod 없으면 DROP
```

### 5.3 LoadBalancer

클라우드 로드밸런서 + NodePort의 조합.

```
외부 LB → NodePort → 내부 Pod
  (클라우드 컨트롤러가 외부 LB 프로비저닝)
  (kube-proxy는 NodePort 처리 동일)
```

### 5.4 ExternalName

CNAME 기반 리다이렉션. kube-proxy가 iptables 규칙을 생성하지 않으며, CoreDNS가 CNAME 응답으로 처리한다.

---

## 6. kube-proxy의 한계와 Cilium

### 6.1 구조적 한계 요약

```
[kube-proxy의 데이터 경로]

패킷
  │
  ▼
NIC → Netfilter(iptables/IPVS)
         │
         ├─ conntrack 엔트리 생성/조회 (매 패킷)
         ├─ nat 테이블 규칙 순회 (iptables 모드)
         ▼
       DNAT 적용 → Pod 전달

문제:
  1. conntrack: 모든 연결에 상태 유지, 메모리/CPU 오버헤드
  2. iptables: O(n) 규칙 순회, 전체 테이블 원자적 갱신
  3. 분산 불가: kube-proxy는 노드별로 독립 동작, 글로벌 로드밸런싱 없음
  4. 헬스체크: kube-proxy가 API Server 경유, 느린 수렴
```

### 6.2 Cilium의 kube-proxy replacement

Cilium은 eBPF를 사용하여 kube-proxy를 완전히 대체할 수 있다 (`--set kubeProxyReplacement=true`).

```
[Cilium의 데이터 경로]

패킷
  │
  ▼
NIC → XDP / tc hook (eBPF)
         │
         ├─ BPF map에서 Service → Endpoint 룩업 (O(1))
         ├─ BPF map 기반 자체 conntrack (Netfilter conntrack 미사용)
         ▼
       DNAT 적용 → Pod 전달 (Netfilter hook 완전 bypass 가능)
```

**BPF map을 이용한 Service 처리:**

```c
/* Cilium BPF map 구조 (개념) */
struct lb4_key {
    __be32 address;   /* Service VIP */
    __be16 dport;     /* Service port */
    __u8   proto;     /* TCP/UDP */
};

struct lb4_service {
    __u32  backend_id;    /* 선택된 Backend ID */
    __u16  count;         /* 백엔드 수 */
    /* ... */
};

/* O(1) 해시 맵 룩업 */
struct lb4_service *svc = map_lookup_elem(&cilium_lb4_services, &key);
```

**Cilium kube-proxy replacement 비교:**

| 항목 | kube-proxy (iptables) | Cilium |
|------|-----------------------|--------|
| 패킷 처리 위치 | Netfilter hooks | XDP / tc BPF |
| 서비스 룩업 | O(n) 규칙 순회 | O(1) BPF 해시맵 |
| conntrack | 커널 Netfilter conntrack | 자체 BPF conntrack |
| 규칙 업데이트 | 전체 iptables 재작성 | BPF 맵 증분 업데이트 |
| DSR (Direct Server Return) | 미지원 | 지원 (NodePort → 소스 IP 보존) |
| Maglev 해싱 | 미지원 | 지원 (일관된 로드밸런싱) |

### 6.3 Maglev 해싱

Cilium은 Google의 Maglev 알고리즘을 지원한다.
기존 확률 기반 랜덤 분기와 달리, 일관된 해시(consistent hashing)로 엔드포인트가 추가/삭제되어도 기존 연결이 최소한으로만 재분배된다.

```
Maglev 테이블:
  각 엔드포인트에 고르게 분산된 해시 슬롯 배정
  패킷의 (src_ip, src_port, dst_ip, dst_port) 해시 → 슬롯 → 엔드포인트

장점:
  - 엔드포인트 추가/삭제 시 최소 재분배 (1/N 비율만 영향)
  - 동일 플로우는 항상 동일 엔드포인트로 → 세션 친화성
```

---

## 7. 네트워크 정책과 kube-proxy

kube-proxy는 **NetworkPolicy를 구현하지 않는다.** NetworkPolicy 처리는 CNI 플러그인의 책임이다.

| 컴포넌트 | 역할 |
|----------|------|
| kube-proxy | Service → Pod DNAT, 로드밸런싱 |
| CNI (e.g., Calico, Cilium) | Pod 간 라우팅, NetworkPolicy 적용 |

Cilium의 경우 NetworkPolicy를 eBPF로 구현하여 iptables filter 체인 없이도 L3/L4/L7 정책을 적용한다.

## 8. 전체 아키텍처 요약

```
[Control Plane]
  API Server
    │  Service/EndpointSlice 변경 이벤트
    ▼
[kube-proxy (DaemonSet, 각 노드)]
    │  iptables 모드: nat 테이블 체인 갱신
    │  IPVS 모드: ipvsadm Virtual/Real Server 갱신
    ▼
[커널 네트워크 스택]
    │
    ├─ iptables 모드:
    │   PREROUTING → KUBE-SERVICES → KUBE-SVC-* → KUBE-SEP-* → DNAT
    │   (Netfilter conntrack이 세션 유지)
    │
    └─ IPVS 모드:
        PREROUTING → (마킹만) → LOCAL_IN → IPVS → DNAT
        (IPVS 세션 테이블 + conntrack)

[Cilium kube-proxy replacement]
    XDP/tc BPF hook → BPF 맵 O(1) 룩업 → DNAT
    (Netfilter 완전 bypass, 자체 BPF conntrack)
```
