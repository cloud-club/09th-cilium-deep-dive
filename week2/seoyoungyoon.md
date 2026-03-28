# CNI

**CNI(Container Network Interface)** 란 Pod에 ip를 부여하고, 네트워크를 연결해주는 플러그인

CNI의 역할
1. Pod마다 고유 IP할당
2. 네트워크 연결(veth 생성)
3. 라우팅 설정(노드간 통신 가능)
4. overlay(VXLAN)


## CNI 흐름

```bash
1. Pod 생성 요청 (API Server -> kubelet)
2. kubelet이 컨테이너 생성
3. CNI plugin 호출
4. CNI가 수행하는 역할
	- 네트워크 네임스페이스 생성
	- veth pair 생성
	- Pod에 IP 할당
  - 라우팅 설정
5. Pod는 eth0 인터페이스를 갖고 통신 가능
```

## CNI 모델, 네트워크 구현 방식

**1. Native Routing** 
    
“Pod IP를 네트워크가 직접 라우팅할 수 있는 경우”

<img width="572" height="420" alt="image" src="https://github.com/user-attachments/assets/f8cde8a0-f8bc-4306-bf6c-34714e211d3b" />


```bash
Pod → veth → bridge(cni0) → host(L3) → 다른 노드 → Pod
```

- encapsulation 없음 → 성능 좋음
- NAT 없음
- 그냥 “일반 네트워크 라우팅”
- 수동 설정 ```ip route add 10.244.2.0/24 via Node2```

라우터(네트워크)가 Pod IP까지 알고있는 경우 가능한 방식 <br>
대표적으로 Cilium (native-routing), Flannel (host-gw)
    
**2. Overlay (VXLAN)**
    
  “Pod IP를 라우팅 못하면 감싸서 보내자”
  
  ```bash
  [원래 패킷]
  PodIP → PodIP
  
  ↓ encapsulation (VXLAN으로 감쌈)
  
  [외부 패킷]
  NodeIP → NodeIP
  ```
  
  장점
  
  - 어디서든 동작 (클라우드 OK)
  - 네트워크 독립적
  
  단점
  
  - 성능 감소 (encap/decap)
  - MTU 문제
  - CPU 사용 증가
  
  대표적으로 Flannel (VXLAN), Calico (VXLAN/IPIP)
  
**3. BGP(Border Gateway Protocol)**

<img width="720" height="405" alt="image" src="https://github.com/user-attachments/assets/be20159b-2d5d-4311-b5c2-6da997dfc8e3" />


“라우터한테 Pod IP를 알려버리자”

```bash
Node → Router:
"나는 10.244.1.0/24 Pod 가지고 있음"

각 노드가 BGP daemon 실행
라우터에 Pod subnet 광고
-> 라우터가 Pod IP 위치를 알고 있음
```

장점

- encapsulation 없음 → 빠름
- 확장성 좋음

단점

- 네트워크 구성 복잡
- 라우터 필요
- 운영 난이도 높음

대표적으로 Calico (BGP mode)
  

### Pod IP를 네트워크가 직접 라우팅할 수 있냐?
가능: Native, BGP <br>
불가능: Overlay

# 전체 흐름

```bash
Pod → (CNI) → Node Network
          ↓
     Service (ClusterIP)
          ↓
   kube-proxy (iptables/IPVS)
          ↓
        Pod
```

쿠버네티스 네트워크는 CNI를 통해 Pod 간 네트워크를 구성하고 kube-proxy가 Service를 구현하여 ClusterIP로 들어온 트래픽을 iptables 또는 IPVS를 통해 실제 Pod로 전달한다.
Service는 가상 IP이며 DNAT을 통해서 Pod IP로 변환된다.

| 역할 | 담당 |
| --- | --- |
| kube-proxy | 어디로 보낼지 결정 |
| CNI | 어떻게 보낼지 결정 |


# kube-proxy (iptables vs IPVS)

Pod는 IP가 계속 바뀐다. 따라서 Service로 고정된 IP로 해결할 수 있다. <br>
이 고정된 Service IP를 실제 Pod IP로 트래픽을 전달하는 역할이 kube-proxy이다.

- iptables방식

```bash
Service -> iptables -> Pod
```

netfilter hook에서 패킷을 가로채서 DNAT를 수행한다.

iptables 룰 기반으로 동작하며 리스트 형식으로 O(n) 따라서 성능이 느려질 수 있다.

디버깅은 어려움 (룰이 많고, 순서 기반이며(어떤 룰이 먼저 매칭되냐에 따라 결과 달라짐), 체인 점프와 NAT/conntrack 때문에 패킷 흐름을 추적하기 어렵기 때문)

- IPVS (IP Virtual Server) 방식

```bash
Service → IPVS → Pod
```

리눅스 커널에 내장된 L4 Load Balancer 엔진으로 테이블로 마찬가지로 service의 고정 IP를 실제 Pod로 연결하는 역할을 한다.

테이블 + 알고리즘 기반으로 동작하고 IP를 해시테이블로 관리한다. O(1)

iptables는 정해진 규칙에 따라 pod 하나를 선택하는 구조라면 IPVS는 알고리즘에 따라 어떤 pod를 고를지 결정한다.

<details>
<summary>IPVS의 알고리즘</summary>
<div markdown="1">

1. Round Robin
    
    가장 기본적인 방식으로, 요청이 들어올때마다 순서대로 Pod를 돌려가면서 선택한다.
    모든 스펙이 비슷하고, 요청 처리시간이 비슷할때 적합하다.
    
2. Least Connection
    
    현재 활성 연결 수가 가장 적은 Pod를 선택한다.
    요청 처리 시간이 제각각인 서비스
    
3. Weighted Round Robin
    
    Pod를 똑같이 보지 않고, 가중치를 둬서 더 많이 처리할 파드를 결정하는방식
    
    백엔드 파드가 항상 동일한 성능은 아닐수있기 때문에 의도적으로 차이를 두고 분배

</div>
</details>
     

# eBPF와 비교

## 기존

```
Pod → Service IP
  ↓
netfilter
  ↓
iptables / IPVS (처리)
  ↓
Native / Overlay / BGP (전달)
  ↓
Pod
```

## eBPF 기반

```
Pod → Service
   ↓
eBPF (처리)
   ↓
Native / Overlay / BGP (전달)
   ↓
Pod
```

1. 네트워크 층
    
    CNI가 Pod에 IP를 할당하는 과정
    네트워크 네임스페이스, eth0, veth pair 생성, Pod IP 할당 등..
    
2. Service 층 (패킷 처리방식)
    
    기존: Service로 들어온 패킷을 iptables, IPVS 가 처리
   
    eBPF: eBPF에서는 이 Service 처리를 함. BPF map + eBPF program으로 처리
    - TC/XDP hook에서 eBPF가 패킷을 직접 처리
    - BPF map: Service → Pod 리스트 저장
    - eBPF program: backend 선택 + rewrite
    - 라우팅, 정책, encapsulation 여부를 결정하고 전달

4. 패킷 전달 층 (패킷이 어떻게 이동하는지)
    
    CNI가 Native / Overlay / BGP 방식으로 패킷을 전달하고,
    Linux networking stack + iptables가 경로를 제어
