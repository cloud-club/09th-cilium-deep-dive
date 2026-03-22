# 📚 내용 정리

## 쿠버네티스 네트워크

Kubernetes는 Pod 간 직접통신이 가능하도록 설계되어 있다.

```bash
Pod → TCP/IP Stack → CNI → Network → Pod
```

Pod는 각각 고유한 IP를 가지며, TCP/IP 스택을 통해 통신한다. Pod도 리눅스 프로세스라서 TCP/IP stack을 그대로 사용한다. Pod는 보통 eth0 인터페이스를 가지고 있다. 이 인터페이스는 veth(virtual ethernet)으로 namespace안에서 `eth0  ←→  veth pair  ←→  host namespace (cni0 or bridge)`로 이동한다.

CNI는 패킷이 파드 밖으로 나가도록 길을 만들어준다. CNI는 Pod의 가상 인터페이스와 노드의 네트워크를 연결하는 `veth pair`를 관리한다. 쿠버네티스 환경은 파드가 수시로 생성되고 사라지기 때문에 매번 수동으로 IP를 할당하고 경로를 지정하기 어려우니까 이런 작업을 CNI가 해준다.

쿠버네티스는 네트워크 구현을 하지 않는다. 대신에 CNI(Container Network Interface)를 지원하는데, 다양한 네트워크 환경(클라우드 환경, 온프레미스, 속도가중요한곳, 보안이 중요한 곳 등등..)이 존재하기 때문에 내 환경에 맞게 CNI를 골라서 선택하면 된다.

이러한 CNI에는 **Cilium, Calico, Flannel** 같은 네트워크 전문 솔루션들이 있다. CNI는 Pod에 ip를 부여하고 어디로 보내야하는지 라우팅을 설정한다.

Network부분에서 실제로 패킷이 이동한다. 같은 노드의 경우 bridge를 통해 빠르게 이동하지만, 다른 노드로 이동하는 경우 CNI별로 이동에 차이가 있다. 

- TCP/IP stack

패킷을 만들고, 어디로 보낼지 결정하는 기본 엔진

- netfilter

커널에서 패킷을 특정 지점에서 가로채는 hook 시스템

역할
1. 패킷 필터링(방화벽)
2. NAT(주소 변환)
3. 트래픽 제어

패킷은 커널 내부에서 이동하면서 정해진 체인(chain)을 순서대로 통화한다.

- 체인종류
    - PREROUTING → 들어오자마자
    - INPUT → 내 컴퓨터로 들어올 때
    - FORWARD → 다른 곳으로 전달할 때
    - OUTPUT → 내 컴퓨터에서 나갈 때
    - POSTROUTING → 나가기 직전

쿠버네티스는 Service를 통해 가상IP(실제로는 존재하지 않음)를 제공한다. 따라서 Service IP로 들어온 패킷을 실제 Pod IP로 변환하는 작업이 필요한데 **netfilter**가 그 역할을 한다.

```bash
Pod → Service IP
      ↓
netfilter (PREROUTING / OUTPUT)
      ↓
DNAT (Service → Pod IP)
      ↓
실제 Pod로 전달
```

- iptables (룰 설정도구)

netfilter에서 어떤 처리를 할지 정의하는 롤 관리 도구

ex) 어떤 패킷을 허용할지 ACCEPT (허용) / 어떤 패킷을 버릴지 DROP (차단) / 어떤 패킷의 주소를 바꿀지 DNAT (목적지 변경), SNAT (출발지 변경)

- conntrack (상태추적)

netfilter에 포함된 기능으로, 네트워크 연결의 상태를 추적하고 기록하는 테이블

예) Service IP를 Pod IP로 변환하는 DNAT가 수행되면 conntrack에  원본 ip&포트, 변환 IP&포트 등이 기록된다. 이 기록을 보고 일관성있게 역방향을 수행한다.

## 쿠버네티스 네트워크 흐름 3가지 경우

1. Pod to Pod
    - 모든 파드는 고유 IP를 가짐
    - Pod는 서로 직접 통신이 가능해야한다.
    - 클러스터 내부에서는 NAT 없이 라우팅으로 직접 통신
        - CNI가 파드에 IP를 할당하고 노드와 파드사이에 veth pair로 연결했기 때문에 IP라우팅으로 바로 통신이 가능하다.
    - 싱글노드 Pod 네트워크
        
        <img width="662" height="527" alt="image" src="https://github.com/user-attachments/assets/23ac2b4c-7af5-4f3a-aeb1-206deda4e258" />
        
        cni0는 리눅스 브리지 역할로 Pod들의 veth를 한곳으로 모아주는 가상 스위치이다. 따라서 내부의 브리지와 라우팅만으로 통신이 가능하다.
        
    - 멀티노드 Pod 네트워크
        
        <img width="1352" height="734" alt="image" src="https://github.com/user-attachments/assets/ccb1ed72-821d-473c-a1b1-70b8b082a1cd" />

        
        마찬가지로 Pod에서 Node쪽 veth로 연결. Node에서 라우팅 테이블을 보고 IP를 찾음. 
        (그림에 있는 NAT는 외부 통신용 SNAT)
        
        해당 사례는 캡슐화(Flannel)방식으로 노드 IP기준으로 패킷을 전달하는 과정
        
2. Pod-to-Service
    
    Pod는 언제든지 재시작되어 IP주소가 변경된다. 따라서 이 IP를 고정하는 Serivce 계층을 사용할 수 있다.
    
    - Service를 통해 여러 파드에 부하를 분산하며 통신
    - kube-proxy + iptables가 핵심
    - iptables 규칙은 ServiceIP를 보는 순간 실제 Pod IP로 DNAT해준다.
    - conntrack이 이 연결을 기억해서 응답패킷도 역방향으로 변환해줌
    
    
    ```mermaid
    flowchart LR
    
    subgraph SRC["출발지"]
        A[Pod A]
    end
    
    %% ===== Node =====
    subgraph NODE["Kubernetes Node"]
        B[Service<br/>10.96.0.1:80]
        C[iptables / kube-proxy]
        D["DNAT<br/>10.96.0.1 → 10.244.1.3"]
    end
    
    %% ===== Destination Pod =====
    subgraph DST["목적지"]
        E[Pod B]
    end
    
    %% ===== Flow =====
    A --> B --> C --> D --> E
    ```
    
    | 구분 | 역할 | 내용 |
    | --- | --- | --- |
    | CNI (Calico) | 네트워크 연결 | Pod에 IP 할당,<br> Pod ↔ Pod 라우팅 구성,<br> 노드 ↔ Pod 네트워크 연결 |
    | kube-proxy | Service 처리 | iptables 규칙 생성, <br> Service IP → Pod IP 매핑 |
3. External to Service: 외부 사용자가 내부 서비스에 접속(Ingress, LoadBalancer)
    
    Pod는 사설 네트워크 IP를 사용하기 때문에 외부에서 직접 접근이 불가능하다. 따라서 내부로 들어오기 위한 진입 지점이 필요하다. 
    ex. NodePort, LoadBalancer, Ingress
    
    - NAT이 발생한다. (외부 → 내부 DNAT / 내부 → 외부 SNAT)
    
    ```mermaid
    flowchart LR
    
    subgraph EXT["외부"]
        A[Client]
    end
    
    subgraph NODE["Kubernetes Node"]
        
        subgraph ENTRY["Entry Point"]
            B[NodePort / LoadBalancer]
        end
    
        subgraph SERVICE["Service Layer"]
            C[Service<br/>10.96.0.1:80]
        end
    
        subgraph NETFILTER["Netfilter"]
            direction TB
            D[PREROUTING]
            E[iptables]
            F[DNAT]
        end
    end
    
    subgraph POD["Pods"]
        G1[Pod B1]
        G2[Pod B2]
        G3[Pod B3]
    end
    
    %% ===== Flow =====
    A --> B --> C --> D --> E --> F
    F --> G1
    F --> G2
    F --> G3
    
    ```
    


>
>❓ **NAT**
>
> 네트워크 주소 변환 기술
> 주로 내부 네트워크에서 사용하는 사설 IP를 공인 IP로 바꿀 때 사용
> 
> **쿠버네티스에서의 NAT**
> 
> egress: 파드가 외부 API를 오출하거나 인터넷에 접속할때
>
> ingress/Service: 외부에서 파드로 들어올때
> 
> 외부 사용자가 서비스에 접속할때 로드밸런서나 노드의 IP로 접속할때 쿠버네티스의 내부 시스템(kube-proxy)가 이 요청을 받아서 실제 파드의 IP로 주소를 변환. 목적지 주소로 변환한다고해서 DNAT(Destination NAT)라고 부름.
>
---

출처) [https://medium.com/finda-tech/kubernetes-네트워크-정리-fccd4fd0ae6](https://medium.com/finda-tech/kubernetes-%EB%84%A4%ED%8A%B8%EC%9B%8C%ED%81%AC-%EC%A0%95%EB%A6%AC-fccd4fd0ae6)


# 🤔 궁금한 점
