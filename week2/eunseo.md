## 1. CNI (Container Network Interface) 구조와 역할

Kubernetes는 자체적으로 네트워크를 구축하지 않고, CNI라는 '표준 인터페이스'를 통해 외부 네트워크 플러그인(Calico, Flannel, Cilium 등)이 네트워크를 구성하도록 위임합니다.

### CNI Plugin의 핵심 역할
* **IP 주소 할당:** IPAM(IP Address Management)을 통해 새로 생성된 Pod에 고유한 IP를 부여합니다.
* **네트워크 인터페이스 생성:** 호스트(Node)와 Pod를 연결하기 위해 가상 이더넷 인터페이스(`veth pair`)를 생성합니다.
* **라우팅 및 통신 보장:** 다른 Node에 있는 Pod와 통신할 수 있도록 라우팅 테이블과 네트워크 오버레이를 구성합니다.

### Pod 네트워크 구성 흐름
Pod가 생성될 때 네트워크가 연결되는 단계적인 과정입니다.
1. **Pod 스케줄링:** API Server가 Pod를 특정 Node에 할당합니다.
2. **CRI 호출:** 해당 Node의 Kubelet이 컨테이너 런타임(containerd 등)에 컨테이너 생성을 요청합니다.
3. **네트워크 네임스페이스 격리:** Pod를 위한 독립적인 네트워크 공간(Network Namespace)이 만들어집니다.
4. **CNI 호출:** Kubelet이 설정된 CNI 플러그인을 실행합니다.
5. **veth pair 연결:** CNI가 호스트의 Root 네트워크 네임스페이스와 Pod의 네트워크 네임스페이스를 연결하는 `veth pair`(가상의 랜선)를 생성합니다.
6. **IP 할당 및 라우팅:** Pod에 IP가 부여되고, 호스트의 브리지 네트워크 또는 라우팅 테이블에 규칙이 추가되어 통신이 가능해집니다.

---

## 2. kube-proxy: Service 트래픽 처리 방식

Pod의 IP는 쉽게 생성되고 사라지기 때문에(Ephemeral), 고정된 진입점인 **Service**가 필요합니다. `kube-proxy`는 각 Node에서 실행되는 데몬으로, 이 Service의 개념을 Node의 실제 네트워크 규칙으로 번역해 줍니다.

### iptables vs IPVS 방식 비교

현재 Kubernetes 환경에서 Service 트래픽을 처리하는 두 가지 주요 모드입니다.

| 구분 | iptables 모드 (기본값) | IPVS (IP Virtual Server) 모드 |
| :--- | :--- | :--- |
| **동작 방식** | 패킷 필터링 프레임워크인 iptables 룰을 순차적으로 탐색 | 리눅스 커널의 L4 로드밸런싱 기술(해시 테이블) 활용 |
| **시간 복잡도** | `O(N)` - 룰이 많아질수록 탐색 시간이 길어짐 | `O(1)` - 룰 개수와 무관하게 빠른 속도 유지 |
| **성능 및 스케일** | 소규모 ~ 중규모 클러스터에 적합 | 수만 개의 Service가 있는 대규모 클러스터에 최적화 |
| **로드밸런싱** | Random 선택 (단순함) | Round Robin, Least Connection 등 다양한 알고리즘 지원 |

---

## 3. Service → Pod 패킷 흐름 추적

가장 대표적인 내부 통신 방식인 **ClusterIP**를 기준으로 트래픽이 어떻게 이동하는지 추적해 봅니다.

### ClusterIP 접근 시 패킷 흐름 (iptables 기준)
1. **요청 발생:** Client Pod가 특정 Service의 ClusterIP(예: `10.96.0.10`)로 패킷을 전송합니다.
2. **veth pair 통과:** 패킷이 Client Pod의 네트워크 네임스페이스를 벗어나 Node의 호스트 네트워크로 나옵니다.
3. **iptables 룰 매칭 (PREROUTING):** Node의 커널 네트워크 스택에서 iptables 규칙을 만납니다.
4. **DNAT (Destination NAT):** iptables는 요청된 ClusterIP를 확인하고, 연결된 실제 Backend Pod의 IP(예: `192.168.1.15`) 중 하나로 도착지 주소를 변환(DNAT)합니다.
5. **라우팅 및 전달:** 도착지가 실제 Pod IP로 바뀌었으므로, CNI가 만들어둔 라우팅 테이블을 타고 목적지 Node와 Pod로 패킷이 최종 전달됩니다.

### 💡 kube-proxy가 개입하는 지점 
처음에는 트래픽이 kube-proxy라는 프로세스를 직접 통과한다라고 생각했는데, 실제 트래픽은 kube-proxy를 거치지 않는다.
* kube-proxy는 Kubernetes의 **Control Plane(API Server)을 계속 감시(Watch)** 합니다.
* Service가 생성되거나 Endpoint(Pod)가 변경되면 이를 감지합니다.
* 감지된 변화를 바탕으로 **각 Node의 운영체제 커널(iptables 또는 IPVS)에 라우팅 룰을 작성하고 업데이트**만 해놓고 빠집니다.
* 즉, 실제 패킷 통신(Data Plane)은 전적으로 리눅스 커널과 CNI가 처리하며, kube-proxy는 룰을 관리하는 역할만 수행합니다.