# Calico

Calico는 모든 Pod가 서로 직접 IP로 통신 가능하도록 두 가지 방식을 통해 해결

- 라우팅 기반 (native routing)
- 터널 기반 (overlay)

## Underlay vs Overlay

### Underlay

- 실제 물리 네트워크
- 실제 라우터, 스위치가 존재하며 있는 그대로 라우팅

### Overlay

- 가짜 네트워크를 위에 하나 더 씌운 것 → 가상 네트워크
- Pod 네트워크를 따로 만들어서 encapsulation으로 보냄
- 대표: VXLAN, IP-in-IP

## Native Routing

→ Calico의 핵심 철학

- Calico는 가능하면 Overlay를 안 쓰고, 라우팅으로 해결하려고 함
- Pod IP를 그냥 실제 네트워크에 라우팅해버리자

### BGP (Border Gateway Protocol)

- Native Routing을 가능하게 하는 네트워크 라우팅 정보 교환 프로토콜
- Calico는 각 노드가 BGP로 이런 정보를 서로 공유함
    
    ```bash
    Node A:
      10.244.1.0/24 (이건 내가 관리하는 Pod IP야)
    
    Node B:
      10.244.2.0/24 (이건 내가 관리)
    ```
    
    → Node A는 `10.244.2.5` pod가 Node B에 있다는 걸 알고 바로 Node B로 보냄
    
- 한계
    - 클라우드(CSP)에서는 Underlay 네트워크를 우리가 제어하지 못하므로,
    - BGP를 제대로 쓰기 어려운 경우가 많다

## Overlay

이를 해결하기 위해 결국 Overlay 방식을 사용

→ Pod 패킷을 감싸서 보냄

### IP-in-IP

- 단순 encapsulation

```bash
[Outer IP (Node A → Node B)]
  [Inner IP (Pod A → Pod B)]
```

### VXLAN

- UDP(L4) 기반이라 IP-in-IP(L4 없음)보다 NAT 통과가 더 잘됨
- 더 범용적이고 표준화됨 → 대부분 클라우드/네트워크 장비 지원

→ 좀 더 표준적이고 클라우드 친화적임

### **Calico의 옵션**

**Calico는 옵션을 통해 Subnet마다 다르게 동작할 수 있다.**

```bash
# option 예시
ipipMode: CrossSubnet
# 또는
vxlanMode: CrossSubnet
```

- 같은 서브넷이면 → BGP + Native Routing
- 다른 서브넷이면 → encapsulation

→ 노드의 ip와 서브넷 마스크를 보고 어떻게 동작할지 결정

### 이런 설계의 이유

**같은 서브넷이면** 

이미 L2에서 서로 통신 가능하므로 굳이 터널 쓸 필요가 없다 → 성능 최적화를 위해 encapsulation이 불필요

**다른 서브넷이면**

라우팅이 안 될 수도 있음 → 안전하게 encapsulation

### 클라우드에서 overlay의 필요성

클라우드에서는 같은 서브넷이어도 overlay가 필요할 수 있다.

이는 클라우드 VPC가 물리적인 L2 네트워크가 아니라 L3 기반으로 구현된 가상 네트워크이기 때문이다.

겉보기에는 같은 서브넷처럼 보이지만, 브로드캐스트 기반 ARP가 자유롭게 동작하는 구조가 아니며, 네트워크는 CSP 내부 시스템에 의해 제어된다.

- 온프레미스
    
    ```bash
    # ARP 브로드캐스트 날려서 MAC 직접 알아냄
    A → (스위치) → B
    ```
    
- 클라우드
    
    ```bash
    # ARP를 직접 날리는 게 아니라 내부 시스템이 대신 응답
    A → (AWS 네트워크 시스템) → B
    ```
    

따라서 노드 IP 간 통신은 가능하지만 Pod IP 대역은 기본 네트워크에서 인식하지 못하기 때문에 Pod 간 통신을 위해 overlay가 필요해진다.

### 간단 실습: IP-in-IP 확인