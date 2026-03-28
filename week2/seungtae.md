# Calico CNI

## 0. Overview 

### CNI(Container Network Interface)란?

Kubernetes는 자체적으로 pod 간 네트워킹을 구현하지 않음. 대신 **CNI 표준 인터페이스**를 정의하고, 실제 구현은 플러그인에 위임하는 구조임.

```
kubelet이 pod 생성
  └── CNI 플러그인 바이너리 호출 (/opt/cni/bin/)
        └── pod에 가상 NIC(veth) 생성
        └── IP 할당
        └── 라우팅 규칙 설정
```

CNI 플러그인은 **"pod가 생성될 때 네트워크를 어떻게 연결할지"** 를 담당하는 컴포넌트. Calico, Flannel 등이 대표적인 CNI 플러그인.

## 1. Calico 란?

**Calico**는 Kubernetes에서 널리 사용되는 CNI(Container Network Interface) 플러그인 중 하나로, Project Calico(현재 Tigera)에서 개발·관리됨.

단순 오버레이 네트워크를 넘어 **NetworkPolicy 엔진**, **BGP 기반 라우팅**, **eBPF dataplane** 등 엔터프라이즈급 기능을 제공하는 것이 특징.

---

### Calico가 CNI로서 동작하는 방식

#### 1단계: pod 생성 시 (CNI 역할)

kubelet이 pod를 생성하면 Calico CNI 바이너리가 호출됨.

```
kubelet → /opt/cni/bin/calico 호출
  ├── pod의 network namespace 생성
  ├── veth pair 생성
  │     └── 한쪽 끝은 pod 내부 (eth0)
  │     └── 다른 쪽 끝은 호스트 (caliXXXXX)
  ├── pod에 IP 할당 (calico-ipam)
  └── Felix에 Endpoint 등록
```

#### 2단계: pod 간 통신 시 (라우팅 역할)

pod에서 다른 노드의 pod로 패킷을 보낼 때, Calico(Felix + BIRD)가 라우팅을 처리함.

```
[Pod A: 10.244.0.2] → 패킷 전송 시도
  │
  ▼
호스트 라우팅 테이블 조회
  └── "10.244.1.0/24은 Node B(tunl0)로"  ← Felix가 설정한 규칙
  │
  ▼
IPIP encapsulation (기본값)
  └── Outer IP: Node A → Node B
  │
  ▼
Node B 도착 → decapsulation → Pod B(10.244.1.2) 전달
```

**핵심 포인트**: Flannel도 비슷한 구조지만, Calico는 여기에 **BGP 라우팅**과 **NetworkPolicy 엔진**이 추가됨.

### 핵심 컴포넌트

| 컴포넌트 | 역할 |
|----------|------|
| **Felix** | 각 노드에서 실행되는 agent. iptables/eBPF 룰을 관리하여 실제 패킷 제어 |
| **BIRD** | BGP Speaker. 노드 간 pod 라우팅 정보(CIDR) 교환 |
| **Typha** | 대규모 클러스터에서 Felix ↔ API Server 간 캐싱 프록시 (부하 분산) |
| **kube-controllers** | K8s 리소스 변화 감지 및 Calico Datastore 동기화 |
| **CNI Binary** | kubelet이 pod 생성 시 호출. veth pair 생성 및 IP 할당 담당 |

---

### Calico vs Flannel 비교

가장 많이 비교되는 두 CNI 이다.

#### 아키텍처 비교

```
Flannel
  ├── flanneld (단일 데몬)
  └── VXLAN overlay로 pod 간 통신

Calico
  ├── Felix (iptables/eBPF 관리)
  ├── BIRD (BGP 라우팅)
  └── IPIP / VXLAN / BGP native 선택 가능
```

#### 기능 비교

| 항목 | Calico | Flannel |
|------|--------|---------|
| **설치 난이도** | 중간 | 쉬움 |
| **NetworkPolicy** | 지원 (K8s 표준 + 확장 정책) | 미지원 |
| **라우팅 방식** | BGP L3 / IPIP / VXLAN 선택 가능 | VXLAN (기본) / host-gw |
| **성능** | BGP native 모드 시 오버헤드 최소 | VXLAN 오버헤드 존재 |
| **대규모 클러스터** | 적합 | 수백 노드 이상 시 한계 |
| **eBPF 지원** | 지원 | 미지원 |

#### 언제 무엇을 쓸까?

- **Flannel**: 빠르게 클러스터를 구성하거나, Network Policy가 불필요한 소규모 환경
- **Calico**: NetworkPolicy가 필요하거나, 온프레미스 BGP 연동, 대규모 클러스터 운영 환경

---

## 2. 네트워크 동작 원리

### 2.1 IP Autodetection

Calico는 각 노드의 **대표 IP(Node IP)** 를 자동으로 감지하여 BGP peering 및 IPIP tunnel endpoint로 사용함.

기본 동작은 `first-found` 방식으로, 시스템의 첫 번째 non-loopback 인터페이스 IP를 선택함.

**IP Autodetection 방법 종류:**

| 방법 | 설명 |
|------|------|
| `first-found` | 첫 번째 non-loopback 인터페이스 (기본값) |
| `kubernetes-internal-ip` | `kubectl get node`의 InternalIP 사용 |
| `interface=<regex>` | 특정 인터페이스명 패턴으로 지정 |
| `cidr=<CIDR>` | 특정 IP 대역에 속하는 인터페이스 선택 |
| `can-reach=<IP>` | 특정 IP에 도달 가능한 인터페이스 선택 |

```yaml
# calico-node DaemonSet 환경변수 설정 예시
env:
  - name: IP_AUTODETECTION_METHOD
    value: "kubernetes-internal-ip"
```

### 2.2 Encapsulation 모드

#### IPIP (IP-in-IP)
- 기본 encapsulation 모드
- pod 패킷을 host IP로 감싸서(encap) 전송
- `tunl0` 가상 인터페이스를 통해 동작
- 다른 서브넷 노드 간 통신에 적합

```
[Pod A 패킷] → [IPIP 캡슐화: Outer IP = Node A IP] → [Node B] → [디캡슐화] → [Pod B]
```

#### VXLAN: Virtual eXtensible LAN
- L2 over UDP encapsulation
- BGP 불필요 (Felix가 직접 FDB 관리)
- UDP 4789 포트 사용

#### No Encapsulation (BGP Native)
- encapsulation 없이 BGP로 직접 L3 라우팅
- 가장 낮은 오버헤드
- **노드가 동일 L2 세그먼트에 있거나 BGP 라우터가 있을 때만 사용 가능**

### 2.3 Pod 간 통신 흐름 (IPIP 모드)

```
Pod A (Node A)
  │
  ▼
veth pair (caliXXXX) → routing table 조회
  │
  ▼
tunl0 (IPIP encapsulation)
  → Outer Src: Node A IP (Calico가 감지한 IP)
  → Outer Dst: Node B IP (BGP로 학습한 Pod B의 next-hop)
  │
  ▼
물리 NIC → 네트워크 전송
  │
  ▼
Node B 물리 NIC
  │
  ▼
tunl0 (IPIP decapsulation)
  │
  ▼
routing table → veth pair → Pod B
```

> **핵심**: IPIP tunnel의 **Outer IP = Calico가 감지한 Node IP** 여야 함.  
> 노드마다 다른 인터페이스가 감지되면, BGP로 advertise된 next-hop과 실제 tunnel endpoint IP 불일치로 통신 단절 발생.

---

## 3. 트러블슈팅: GPU 노드 Pod 간 통신 단절

### 3.1 환경

| 항목 | 내용 |
|------|------|
| **Kubernetes 버전** | v1.28 |
| **Calico 버전** | v3.26 |
| **인프라** | On-Premises |
| **Encapsulation** | IPIP (기본값) |
| **Master/Worker 노드** | VM 추정, NIC 2개: `172.x` (VM NIC) + `10.x` (Physical NIC) 공존 |
| **GPU 노드** | 물리 서버 추정, NIC 1개: `10.x` (Physical NIC) 단일 구성 |
| **할당받은 IP** | 운영팀으로부터 `10.x` 대역만 전달받음. `172.x` 존재는 `ifconfig`로 직접 발견 |

### 5.2 증상

| 통신 | 상태 |
|------|------|
|Master ↔ Worker 노드 간 | OK |
|GPU 노드 ↔ GPU 노드 간 | OK |
|Master 노드 ↔ GPU 노드 | OK |
|Master/Worker Pod ↔ GPU Pod 간 통신 | X |

GPU 노드의 calico-node pod 로그에서 다음 에러 확인:

```
Failed to connect to https://10.96.0.1:443/api/v1/namespaces/kube-system/
serviceaccounts/calico-cni-plugin/token
```

> node-to-node 통신은 정상인데 pod-to-pod 통신만 단절되어 초기 진단이 어려웠던 케이스.

### 3.3 원인 분석

#### 3.3.1 Calico IP Autodetection 오감지

Calico의 기본 IP 감지 방식(`first-found`) 설정

`first-found`는 `ip addr` 또는 `ifconfig` 기준으로 **열거 순서상 첫 번째 non-loopback 인터페이스**를 선택

```
Master/Worker 노드 (VM)
  ├── NIC 1: 172.x.x.x (VM NIC) ← first-found 선택
  └── NIC 2: 10.x.x.x  (Physical NIC)
  → Calico Node IP = 172.x.x.x

GPU 노드 (Physical Server)
  └── NIC 1: 10.x.x.x  (Physical NIC만 존재) ← first-found 선택
  → Calico Node IP = 10.x.x.x
```

> **발견**: 인프라 담당자가 아닌 관계로 최초에 `10.x` 대역 IP만 전달받은 상황. Master/Worker 노드에 `172.x` NIC가 존재한다는 사실을 직접 NIC를 탐색하는 과정에서 파악.

#### 3.3.2 node-to-node 정상, pod-to-pod 단절인 이유

트러블슈팅에서 원인 분석에 어려움 존재.

| 구분 | 동작 방식 | 결과 |
|------|----------|------|
| **node-to-node** | OS 라우팅 테이블 기반 L3 통신. 172.x ↔ 10.x 간 라우팅 경로 존재 | OK |
| **pod-to-pod** | Calico IPIP tunnel 사용. Outer IP = Calico가 감지한 Node IP | X |

pod-to-pod 통신 단절 흐름:

```
1. Master/Worker 노드의 Calico: "GPU 노드 Pod의 next-hop = 10.x (GPU Node IP)"
                                  ← BGP로 advertise된 정보

2. IPIP 패킷 전송 시:
   Outer Src = 172.x (Master/Worker의 Calico Node IP)
   Outer Dst = 10.x  (GPU 노드의 Calico Node IP)

3. GPU 노드 도착:
   tunl0 인터페이스는 10.x를 listen → 수신 가능

4. GPU 노드 → Master/Worker로 응답 시:
   Outer Src = 10.x (GPU의 Calico Node IP)
   Outer Dst = 172.x (Master/Worker의 Calico Node IP)

5. Master/Worker의 tunl0:
   172.x를 listen하고 있음 → 이론상 수신 가능
   But, BGP routing table의 next-hop 불일치로 인한 비대칭 라우팅 문제 발생
```

추가로, GPU 노드의 calico-node가 **Kubernetes API Server(`10.96.0.1:443`)에 접근 실패**한 것도 동일한 원인에서 비롯됨. GPU 노드의 Calico가 `10.x`를 Node IP로 인식하면서, service network 접근에 필요한 kube-proxy의 iptables 룰이 올바른 인터페이스에 미적용된 상태였음.

### 3.4 진단 과정

#### Step 1. Pod 레벨 연결성 확인

```bash
# dnsutils pod를 GPU 노드에 배치
kubectl run dnsutils --image=<network-debug-image> --overrides='
{
  "spec": {
    "nodeSelector": {"kubernetes.io/hostname": "<GPU 노드명>"},
    "tolerations": [{"operator": "Exists"}]
  }
}' --restart=Never -- sleep 3600

# Master/Worker 노드 Pod IP로 ping
kubectl exec -it dnsutils -- ping <worker-pod-ip>

# DNS 해석 확인
kubectl exec -it dnsutils -- nslookup kubernetes.default

# API Server 직접 curl
kubectl exec -it dnsutils -- curl -k https://10.96.0.1:443/healthz
```

#### Step 2. kube-proxy 상태 확인

```bash
# GPU 노드의 kube-proxy 로그 확인
kubectl logs -n kube-system -l k8s-app=kube-proxy \
  --field-selector spec.nodeName=<GPU 노드명>

# iptables 룰 확인 (GPU 노드에서)
iptables -t nat -L KUBE-SERVICES -n | grep 10.96.0.1
```

#### Step 3. NIC 구성 및 네트워크 세그먼트 확인

노드에 직접 접속하여 NIC 탐색.

```bash
# 각 노드에 ssh 접속 후 NIC 전체 확인
ifconfig -a

# Master/Worker 노드에서 172.x NIC 발견
# 1: lo: <LOOPBACK> ...
# 2: eth0: 172.x.x.x/24   ← VM NIC 
# 3: eth1: 10.x.x.x/24    ← Physical NIC

# GPU 노드에서는 10.x NIC 하나만 존재
# 1: lo: <LOOPBACK> ...
# 2: eth0: 10.x.x.x/24    ← Physical NIC 
```

```bash
# Calico가 실제로 어떤 IP를 Node IP로 등록했는지 어노테이션으로 검증
kubectl get node <노드명> -o jsonpath='{.metadata.annotations}'
# projectcalico.org/IPv4Address 어노테이션 확인

# 라우팅 테이블 확인 (GPU 노드에서)
ip route show
# 10.244.x.x (Pod CIDR) via <next-hop IP> dev tunl0 형태 확인
```

#### Step 4. 원인 특정

```bash
# Master 노드에서 확인
kubectl describe node <master-node>
# 172.x.x.x/24  ← VM NIC

# GPU 노드에서 확인
kubectl describe node <gpu-node>
# 10.x.x.x/24  ← Physical NIC

# Calico가 노드 타입별로 서로 다른 NIC를 Node IP로 사용하고 있음을 확인
```

### 3.5 해결

`IP_AUTODETECTION_METHOD`를 `kubernetes-internal-ip`로 명시 설정하여 모든 노드가 Kubernetes Internal IP(동일 네트워크 대역)를 사용하도록 변경. (모든 노드가 설치 IP는 동일 네트워크 대역으로 설정되었기 때문)

#### Step 1. calico-node DaemonSet 수정

```bash
kubectl edit daemonset calico-node -n kube-system
```

```yaml
spec:
  template:
    spec:
      containers:
        - name: calico-node
          env:
            - name: IP_AUTODETECTION_METHOD
              value: "kubernetes-internal-ip"  # 추가
```

> `kubernetes-internal-ip`는 `kubectl get node`의 `INTERNAL-IP` 값을 그대로 사용
> 이 값은 kubelet 시작 시 `--node-ip` 플래그로 지정되거나 클라우드/온프레미스 환경에서 자동 등록된 값

#### Step 2. DaemonSet 재시작

```bash
kubectl rollout restart daemonset calico-node -n kube-system
```

#### Step 3. 검증

```bash
# 1) 모든 노드의 Calico Node IP가 동일 대역(10.99.x.x)으로 통일되었는지 확인
kubectl describe <node>

# 2) GPU 노드에서 라우팅 테이블 확인
# Master/Worker 노드와 동일한 10.99.x.x 대역으로 라우팅되는지 확인
ip route show | grep tunl0

# 3) Pod 간 통신 확인
kubectl exec -it <GPU-노드-Pod> -- ping <Worker-노드-Pod-IP>
kubectl exec -it <GPU-노드-Pod> -- curl -k https://10.96.0.1:443/healthz
```

### 3.6 정리

#### 근본 원인

```
Calico IP Autodetection (first-found, 기본값)
  ↓
Master/Worker 노드: NIC 2개 (172.x VM NIC, 10.x Physical NIC)
  → first-found가 열거 순서상 먼저 나온 172.x 선택
GPU 노드: NIC 1개 (10.x Physical NIC)
  → 선택지가 하나뿐이므로 10.x 선택
  ↓
노드 타입별로 Calico Node IP 대역이 달라짐
  ↓
BGP advertise된 next-hop IP와 실제 IPIP tunnel endpoint IP 불일치
  ↓
IPIP tunnel 패킷 라우팅 실패
  ↓
Pod-to-Pod 통신 단절
```

#### 배운점

**1. Heterogeneous 노드 환경에서 IP Autodetection 명시적 설정 필요**

VM과 물리 서버가 혼재하거나 노드마다 NIC 구성이 다른 환경에서는 `first-found` 기본값이 노드별로 서로 다른 인터페이스를 선택할 수 있음. 명시적 설정 권장. 단, NIC 명으로 설정은 자제하는 것이 좋음. (노드별 NIC 명 다를 수 있기 때문)

**2. node-to-node 통신 정상 ≠ pod-to-pod 통신 정상**

node-to-node 통신은 OS 라우팅 테이블 기반 L3 통신이고, pod-to-pod 통신은 Calico IPIP tunnel과 BGP 라우팅 테이블을 별도로 사용함. 두 레이어가 독립적으로 동작하므로 node-to-node가 정상이더라도 Calico 레이어 문제로 pod-to-pod 단절 가능.

**3. 인프라 구성 직접 검증 필요**

해당 트러블슈팅 시 인프라 담당자에게 `10.x` 대역 IP를 전달받았지만, 실제 Master/Worker 노드에는 `172.x` NIC도 추가로 존재했음. 직접 탐색하기 전까지 파악 불가. 네트워크 트러블슈팅 시 전달받은 IP 정보 외에 실제 노드 NIC 구성을 직접 확인하는 것이 중요.

**4. IP_AUTODETECTION_METHOD 옵션 선택 기준**

| 상황 | 권장 방법 |
|------|----------|
| 모든 노드의 Internal IP가 동일 대역 | `kubernetes-internal-ip` |
| 특정 인터페이스명을 알고 있음 | `interface=eth0` 또는 정규식 (비권장) |
| 특정 IP 대역을 알고 있음 | `cidr=10.99.0.0/16` |
| 특정 외부 IP 도달 가능 여부로 판단 | `can-reach=<gateway-ip>` |