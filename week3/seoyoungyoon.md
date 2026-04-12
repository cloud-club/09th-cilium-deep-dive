# Calico 네트워크 플러그인

쿠버네티스에서 네트워크 플러그인은 노드와 파드 간 통신을 원활하게 해주는 핵심 역할을 한다. 따라서 쿠버네티스 노드와 밀접하게 연결되어 있다.

### 네트워크 플러그인의 주요 역할

네트워크 플러그인은 크게 두 가지로 나뉜다.

- **네트워크 플러그인**은 파드를 네트워크에 연결하는 역할을 한다.
- **IPAM 플러그인**은 파드에게 IP 주소를 할당하고 IP 주소를 관리하는 역할을 한다.

많은 CNI 플러그인들이 오버레이 네트워크(VXLAN)를 사용해서 문제를 해결하지만, **Calico**는 L3 네트워킹을 기본으로 한다. Calico는 파드 IP를 실제로 라우팅 가능한 IP로 만들어 BGP를 통해 노드 간 라우트를 공유한다.

<img width="1024" height="359" alt="image" src="https://github.com/user-attachments/assets/8b56cff0-8a51-4e96-a75b-660a505b28a0" />


### Calico의 주요 컴포넌트

<img width="1200" height="628" alt="image" src="https://github.com/user-attachments/assets/92dd463c-043b-4ea3-be13-3bfa70714584" />


Calico는 컨트롤 플레인과 데이터 플레인으로 구성된다. 주요 컴포넌트는 다음과 같다.

1. **CNI 플러그인**은 파드가 생성될 때 네트워크 인터페이스를 붙여주는 역할을 한다.
2. **IPAM 플러그인**은 파드에게 IP 주소를 할당한다.
3. **Felix**는 각 노드에서 정책과 라우팅 상태를 확인하고, 실제 커널 데이터 플레인에 반영하는 핵심 에이전트이다.
4. **BIRD**는 BGP 라우팅이 필요할 때 경로를 광고하는 라우팅 데몬이다.
5. **Typha**는 대규모 클러스터에서 API 서버와 데이터스토어의 부하를 줄여주는 fan-out 중계 계층이다.
6. **kube-controllers**는 쿠버네티스 리소스와 Calico 리소스 간 동기화, 가비지 컬렉션 등을 담당한다.

VXLAN-only 모드를 사용할 경우 BGP를 사용하지 않아도 되므로, 움직이는 컴포넌트를 줄일 수 있다.

### Calico의 주요 특징

Calico는 쿠버네티스에서 기본적으로 사용하는 L2 브리지를 사용하지 않는다. 이로 인해 불필요한 복잡성을 줄이고 성능 오버헤드를 제거한다.

- **Calico CNI IPAM**은 하나 이상의 설정 가능한 IP 주소 범위에서 파드 IP를 할당한다.
- **Overlay 네트워크 모드**로는 VXLAN, IP-in-IP, 서브넷 간 전용 모드가 있다.
- **Non-overlay 네트워크 모드**는 일반적인 L2 네트워크 위, L3 네트워크 위(퍼블릭 클라우드), BGP가 가능한 네트워크에서 사용할 수 있다.
- **네트워크 정책**은 쿠버네티스 네트워크 정책과 Calico 네트워크 정책 기능을 모두 지원한다.

### Calico의 동작 방식

<img width="764" height="331" alt="image" src="https://github.com/user-attachments/assets/8728e4ae-4040-4253-9560-00bd46d770e4" />


Calico는 데몬셋(DaemonSet)으로 각 노드에 calico-node 파드가 동작한다. 이 파드 안에는 BIRD, Felix, confd 등이 함께 실행된다. Calico 컨트롤러는 디플로이먼트 형태로 생성된다.

BGP를 이용해 각 노드에 할당된 파드 대역 정보를 전달한다. 따라서 쿠버네티스 내부뿐만 아니라 물리적인 라우터와도 연동이 가능하다. (Flannel은 불가능)

calico-node 파드 안에서 오픈소스 라우팅 데몬인 **BIRD**가 실행되어 각 노드의 파드 정보를 다른 노드에 전파한다. 이후 **Felix**가 리눅스 커널의 라우팅 테이블과 iptables 규칙에 해당 정보를 주입한다. **confd**는 설정 변경 시 이를 즉시 반영하는 트리거 역할을 한다.

### 파드 생성 시 Calico 네트워크 구조

파드가 생성되는 과정은 다음과 같다.

1. 쿠버네티스가 파드 생성을 요청한다.
2. kubelet이 CNI를 호출한다.
3. Calico CNI가 veth pair를 생성한다. 한쪽 끝은 파드 네임스페이스의 eth0 인터페이스가 되고, 다른 한쪽은 호스트의 caliXXXX 인터페이스가 된다.
    
    ```bash
    [Pod eth0]  ←→  [Host caliXXXXX]
    
    Host OS
     ├─ namespace A (Pod1)
     │    └─ eth0 (10.233.x.x)
     │
     ├─ namespace B (Pod2)
     │    └─ eth0 (10.233.x.x)
     │
     └─ namespace host
          └─ caliXXXXX
    ```
    
4. Calico IPAM이 파드에게 IP 주소를 할당한다.
5. Felix가 해당 파드의 정책과 라우팅 상태를 커널 데이터 플레인에 반영한다.

이때 파드와 호스트는 **veth pair**로 연결된다.

각 파드마다 하나의 네트워크 네임스페이스와 eth0 인터페이스, 호스트 측에 하나의 veth 인터페이스가 생성된다. 파드가 삭제되면 컨테이너가 종료되고 네트워크 네임스페이스와 인터페이스가 함께 삭제된다.

**중요한 점**은 클러스터에서는 기본 CNI를 하나만 사용할 수 있다는 것이다. /etc/cni/net.d/ 디렉토리에 CNI 설정 파일이 존재하며, 파드 생성 시 노드 단위로 하나의 CNI만 호출된다.

### Calico의 Data Plane: iptables vs eBPF

Calico는 두 가지 데이터 플레인을 지원한다.

**Standard Linux dataplane (iptables)**

- 서비스 처리는 kube-proxy가 담당한다.
- 네트워크 정책은 iptables 규칙으로 반영된다.
- 호환성이 매우 좋다.

**eBPF dataplane**

- 서비스 처리까지 eBPF 프로그램과 맵으로 구현한다.
- kube-proxy를 완전히 대체할 수 있다.
- 정책도 BPF instruction과 map으로 처리한다.
- NodePort로 들어오는 트래픽의 source IP를 보존하기에 유리하다.
- 더 낮은 latency와 높은 처리량을 제공한다.

eBPF 모드를 사용하려면 최소 Linux kernel 5.10 이상(일부 RHEL은 4.18 백포트)이 필요하며, 모든 기능을 충분히 사용하려면 kernel 6.6 이상을 권장한다. eBPF 모드에서는 etcd datastore가 지원되지 않는다.

### Calico와 Cilium의 차이점

Calico도 eBPF를 지원하지만, Cilium을 선택하는 이유는 다음과 같다.

- **Calico**는 원래 L3 + BGP 기반 네트워킹을 중심으로 설계되었으며, eBPF는 추가 옵션이다.
- **Cilium**은 처음부터 eBPF 기반으로 설계되어 kube-proxy, 정책, 로드밸런싱까지 모든 기능을 eBPF로 통합한다.

즉, Calico는 라우팅 중심에서 eBPF를 확장한 것이고, Cilium은 eBPF 중심으로 모든 기능을 구현한 것이다.

### L2와 L3의 차이

- **L2 (Data Link Layer)**: MAC 주소 기반 통신이다.
- **L3 (Network Layer)**: IP 주소 기반 통신이다.

Calico는 L3 기반 네트워크로, 파드 IP를 실제 라우팅 대상으로 취급한다. 따라서 BGP를 통해 파드 IP를 광고할 수 있고, 오버레이 네트워크 없이도 효율적인 통신이 가능하다.

### Control Plane vs Data Plane

- **Control Plane**: 어떻게 패킷을 보내야 하는지 결정하는 영역이다. (Felix, BIRD 등)
- **Data Plane**: 실제 패킷을 전달하는 영역이다. (eBPF, iptables)

파드가 생성되고 IP가 할당되며 정책이 만들어지면, Control Plane이 BGP route를 계산하고 Data Plane이 veth와 규칙을 생성해 패킷 전달을 시작한다.



---

Calico 공식문서 https://docs.tigera.io/calico/latest/networking/determine-best-networking

Calico 동작방식 https://velog.io/@200ok/Kubernetes-Calico-CNI-%EC%9D%B4%ED%95%B4%ED%95%98%EA%B8%B0
