# Multi-Cluster Networking: CNI & Tool 비교 분석

멀티 클러스터(Multi-Cluster) 환경은 고가용성(HA), 재해 복구(DR), 지리적 분산(Edge), 클라우드 마이그레이션 등을 위해 필수적인 아키텍처로 자리 잡고 있습니다. 

하지만 클러스터 간 통신을 어떻게 구현하느냐에 따라 **네트워크(Layer 3/4) 수준의 CNI 결합** 방식과 **애플리케이션(Layer 7) 수준의 Service Mesh 방식**으로 나뉩니다. 이 문서에서는 멀티 클러스터를 지원하는 대표적인 도구들의 아키텍처, 성능, 역할을 비교하고 실전 설정 코드를 정리합니다.

---

## 1. 멀티 클러스터 기능을 사용하면 무엇이 가능한가? (주요 유즈케이스)

![Cilium ClusterMesh Concept](https://cdn.sanity.io/images/xinsvxfu/production/52bea7b709bca9f1ccfddbf4a7c050095768cada-1600x900.jpg?auto=format&q=80&fit=clip&w=1080)
*(그림: 여러 클러스터를 단일 네트워크 통신망으로 묶어주는 멀티 클러스터 개념도)*

하나의 거대한 쿠버네티스 클러스터를 운영하는 대신 멀티 클러스터를 연동하면, 마치 여러 위치에 있는 데이터센터나 클라우드 리전을 하나의 시스템처럼 다룰 수 있습니다.

* **고가용성 및 재해 복구 (Disaster Recovery)**: A 리전(클러스터 A)에 장애가 발생하거나 유지보수 작업 중이어도, 트래픽을 중단 없이 B 리전(클러스터 B)의 동일 서비스로 자동 페일오버(Failover) 시킵니다. (해당 기능은 Active-Active 혹은 Active-Passive로 구현 가능)
* **지연시간 최소화 (Geo-Locality)**: 사용자와 가장 가까운 지리적 위치에 있는 클러스터 안의 파드로 트래픽을 지능적으로 라우팅하여 레이턴시를 줄입니다.
* **클러스터 워크로드 분산 (크기 한계 극복)**: 단일 클러스터가 수용할 수 있는 최대 노드 수나 파드 수 한계를 넘어서, 여러 개의 작은 클러스터로 나누어 관리하면서도 통신은 하나인 것처럼 묶습니다.
* **공유 서비스 (Shared Services)**: 무거운 백엔드 DB, 중앙 집중식 모니터링 시스템(Prometheus, ELK), 혹은 보안(Vault) 등 '관리용 서비스'를 한 클러스터에만 띄워두고 다른 수많은 워크로드 클러스터가 쉽게 접근하여 사용할 수 있도록 만듭니다.
* **안전한 제로 다운타임 업그레이드 (Blue-Green Routing)**: 쿠버네티스 버전 자체를 업그레이드 해야 할 때, 기존 운영 중인 1.25 클러스터에서 새로 구축한 1.28 클러스터로 라우팅 가중치(Weight)를 서서히 넘겨 무분단 클러스터 마이그레이션이 가능해집니다.

---

## 2. CNI 레벨 vs Service Mesh 레벨의 접근 방식

| 비교 항목 | CNI 레벨 멀티 클러스터 (Cilium, Calico, Submariner) | Service Mesh 레벨 멀티 클러스터 (Istio, Linkerd) |
| :--- | :--- | :--- |
| **동작 계층** | L3 / L4 (네트워크 패킷 레벨) | L4 / L7 (네트워크 및 애플리케이션 프록시) |
| **주요 기술** | eBPF, BGP, 터널링(VXLAN, IPsec/WireGuard) | Envoy, Linkerd2-proxy (유저스페이스 프록시) |
| **성능 / Latency** | **매우 높음 (수행 단계 단축, 커널 레벨)** | 상대적으로 낮음 (프록시를 거치며 오버헤드 발생) |
| **보안 및 인증** | Identity 기반 Network Policy (L3/L4 중심) | mTLS, JWT 인증, 세밀한 L7 정책 제어 |
| **IP 대역(CIDR)** | 대부분 클러스터 간 Pod/Service CIDR 중복 불가 | CIDR 중복 허용 가능 (Gateway를 통해 NAT/Proxying) |

---

## 3. 주요 멀티 클러스터 도구 심층 비교

### 3.1 Cilium (ClusterMesh)
- **역할**: eBPF 기반의 초고속 데이터플레인을 확장하여 여러 클러스터를 평면적(Flat)인 단일 네트워크 공간처럼 묶습니다.
- **성능**: Kube-proxy 없이 eBPF 소켓 로드밸런싱(Socket LB)을 사용하므로, 클러스터 간 통신이 노드 내 통신에 가까운 **최상의 성능**을 보장합니다.
- **특징**:
  - 클러스터 간 **Pod IP와 전역 Identity(Security Identity)** 가 동기화됩니다.
  - 전제 조건: 모든 클러스터 간에 **Pod/Service CIDR 겹침이 없어야** 합니다.
- **공식 문서**: [Cilium ClusterMesh Architecture](https://docs.cilium.io/en/stable/network/clustermesh/clustermesh/)


### 3.2 Istio (Multi-Cluster Service Mesh)
- **역할**: 인프라(CNI)와 무관하게 애플리케이션 프록시(Envoy) 단에서 클러스터 간 트래픽을 라우팅하고 제어합니다.
- **성능**: CNI 방식보다 Latency가 높습니다(모든 패킷이 Kube-proxy 및 Envoy 사이드카를 거침). 단, 최근 **Ambient Mesh(Ztunnel)**의 도입으로 사이드카 오버헤드를 줄이려는 시도가 진행 중입니다.
- **특징**:
  - `Primary-Remote` 또는 `Multi-Primary` 아키텍처 지원.
  - East-West Gateway를 통해 IP 대역이 겹치는 클러스터 간(서로 다른 VPC) 통신 가능 (Network 분리 모델).
  - 클러스터 경계를 넘어 완벽한 End-to-End mTLS 보장.


### 3.3 Submariner
- **역할**: CNCF 샌드박스 프로젝트로, 여러 Kubernetes 클러스터 간에 **CNI에 구애받지 않고(Agnostic)** L3 네트워크 연결을 제공합니다.
- **성능**: Gateway 노드 간 터널링(IPsec, WireGuard, VXLAN)을 사용하므로, VXLAN이나 암호화 헤더 오버헤드에 의한 **네트워크 성능 저하**가 존재합니다. 특정 노드가 Gateway 병목이 될 수 있습니다.
- **특징**:
  - Kube-proxy 기반의 CNI(예: Flannel, Weave 등)를 쓰더라도 Submariner를 올리면 멀티 클러스터가 가능합니다.
  - Broker(중앙 제어소)를 통해 서비스 검색(Service Discovery) 정보와 IP 라우팅 경로를 교환합니다.
- **공식 문서**: [Submariner Architecture](https://submariner.io/architecture/)


### 3.4 Calico (BGP 기반 Multi-Cluster / Federation)
- **역할**: BGP를 활용하여 외부 라우터(Underlay)를 통해 각 클러스터의 Pod Network 대역을 전파하는 전통적인 방식.
- **성능**: eBPF를 쓰지 않는 고전 Calico 모드에서는 커널의 라우팅과 iptables를 사용하므로 준수한 성능(Native L3)을 냅니다.
- **특징**: 
  - 엄밀히 말해 독자적인 "Multi-Cluster 싱크 도구"를 제공한다기보다는, BGP Route Reflector 구조를 활용하여 AS(Autonomous System) 간 라우팅을 맞추는 인프라 중심의 설계입니다.
  - "Tigera Secure (Calico Enterprise/Cloud)"에서는 ClusterMesh와 비슷한 Federation(Federated Endpoint Identity)을 상용 기능으로 제공.

---

## 4. 요약 및 선택 가이드 (Conclusion)

- **Cilium ClusterMesh (추천 ✨)**
  - 가장 **빠르고 직관적인 데이터플레인** 경험을 원할 때. 
  - IP 대역이 충돌하지 않는 환경에서 **eBPF 성능 극대화** 및 단일 Security Identity(Global Network Policy) 관리가 필요할 때.
- **Istio (Service Mesh)**
  - CNI 레벨의 수정(접근 권한 제한 등)이 불가능하거나 **IP 대역 충돌(Overlapping CIDR)**이 있는 클러스터들을 연결할 때.
  - L7 수준의 카나리 배포, 완벽한 mTLS 중심 통제가 인프라보다 우선순위일 때. (단백한 속도는 다소 희생됨)
- **Submariner**
  - 서로 다른 CNI(Flannel vs Calico)를 쓰는, **이미 격리 구축된 이종 클러스터**를 터널(WireGuard 등)로 단순하게 L3 연결만 시키고 싶을 때.

### 참고 문헌 및 논문 자료
1. [CNCF - Submariner Architecture & Use Cases](https://submariner.io/getting-started/)
2. [Isovalent - Deep Dive into Cilium ClusterMesh](https://isovalent.com/blog/post/cilium-clustermesh/)
3. [Istio Multi-cluster Deployment Models](https://istio.io/latest/docs/setup/install/multicluster/)

---

## 5. Deepwiki로 알아보는 Cilium Cluster Mesh

Cilium 소스코드의 구조와 다중 클러스터(Multi-Cluster) 연동 구현의 핵심을 더 깊이 파악하고 싶다면, Deepwiki 분석 자료를 참고해볼 수 있습니다. 
아래 링크를 통해 Cluster Mesh 기능이 소스코드 레벨에서 어떻게 구현(어디에 위치해 있고, 컴포넌트 간 어떻게 동작하는지)되어 있는지 인사이트를 얻을 수 있습니다.

* **[Deepwiki 살펴보기: Where is the multicluster code?](https://deepwiki.com/search/where-is-the-multicluster-code_b5462709-0d39-4a40-a2b4-74908c2feea8)**

해당 가이드에서는 `clustermesh-apiserver`의 역할, 글로벌 서비스 로드밸런싱 구조, 그리고 ETCD 백엔드로 KVStore를 연동하여 Endpoint와 Identity를 교환하는 코드의 실제 경로를 추적해 볼 수 있습니다. 소스코드 레벨의 완전한 분석을 목표로 한다면 위 자료를 함께 숙지해보세요!

---

추가로 자료를 찾다가 아래 블로그를 보게 되었습니다.
엄청 자세하게 정리가 잘 되어있다고 생각하여 참고해보시면 도움이 많이 될 것 같습니다!
https://dobby-isfree.tistory.com/category/%EA%B8%B0%EC%88%A0%20%ED%86%A0%EB%A1%A0%EC%9E%A5/%5BK8s%5D%20Kubernetes


---

## 부록. 고가용성 및 재해 복구에 대하여

### 1. 관련된 모든 리소스가 양쪽에 동시에 존재해야 하는가?
**네트워크가 연결되기 위한 최소 조건**과 **실제 앱이 돌아가기 위한 조건**이 다릅니다.

* **Service (필수)**: 서로 연결하려는 두 클러스터에 **해당 Service 리소스(껍데기)는 무조건 양쪽 모두**에 있어야 합니다. 그래야 CNI(예: Cilium)가 "아, 이 클러스터에도 이 서비스가 존재하네"라고 인지하고 트래픽을 받을 준비를 합니다.
* **Pod (선택적)**: 파드는 양쪽에 동시에 떠 있을 필요는 없습니다. 클러스터 A에는 파드를 3개 띄우고, 클러스터 B에는 파드를 0개(스케일 다운)로 둬도 됩니다. 서비스 껍데기만 있다면, 클러스터 B로 들어온 요청도 클러스터 A의 파드로 알아서 넘어갑니다. (물론 진정한 고가용성을 원한다면 양쪽에 다 띄워두는 것이 좋습니다.)
* **그 외 리소스들 (ConfigMap, Secret 등)**: 파드가 B 클러스터에서도 정상적으로 "실행"되려면 당연히 앱 구동에 필요한 설정 파일들도 B 클러스터에 복사되어 있어야 합니다. (그래서 실무에서는 사람이 일일이 양쪽에 만들지 않고, **ArgoCD나 Flux 같은 GitOps 도구**를 사용해 동일한 YAML 매니페스트를 양쪽 클러스터에 똑같이 찍어내도록 배포를 자동화합니다.)

---

### 2. 서로 다른 클러스터에서 "동일한 서비스"인지 어떻게 구분하는가?
각 클러스터는 완전히 독립적이기 때문에 기본적으로 남의 클러스터에 뭐가 있는지 모릅니다. 멀티 클러스터 도구(예: Cilium ClusterMesh)는 다음 **두 가지 조건이 완벽히 일치**할 때 흩어진 두 서비스를 **"하나의 동일한 서비스"**로 묶어(Merge)버립니다.

**① 네임스페이스(Namespace)와 서비스 이름(Service Name)의 완벽한 일치**
* 클러스터 A의 `default` 네임스페이스에 있는 `backend-svc`
* 클러스터 B의 `default` 네임스페이스에 있는 `backend-svc`
* 이렇게 **이름과 네임스페이스가 똑같으면** Cilium은 "이건 같은 서비스구나!"라고 1차적으로 판단합니다.

**② 글로벌 서비스 선언 (Annotation)**
하지만 우연히 이름만 같을 수도 있기 때문에 무작정 합치지는 않습니다. 관리자가 특수한 표시(Annotation)를 달아주어야 합니다.
```yaml
# 클러스터 A와 B의 서비스에 각각 아래 권한을 달아줍니다.
metadata:
  annotations:
    io.cilium/global-service: "true"
```

**⚙️ 내부에서는 어떤 일이 일어날까요?**
1. 이 표시(`global-service="true"`)가 달리면, 각 클러스터의 Cilium Agent가 자기 클러스터에 있는 해당 서비스의 **파드 IP 리스트(Endpoints)**를 수집합니다.
2. 수집한 리스트를 전용 사서함(ClusterMesh etcd/KVStore)을 통해 다른 클러스터와 교환합니다.
3. 결과적으로 클러스터 A의 `backend-svc`에 묶인 실제 목적지 리스트에는 **클러스터 A의 파드 IP 2개 + 클러스터 B의 파드 IP 3개 = 총 5개의 파드 IP**가 하나로 합쳐져 등록됩니다.
4. 이제 클러스터 A의 어떤 프론트엔드가 이 서비스를 호출하면, 마치 자기 클러스터에 파드가 5개 있는 것처럼 트래픽을 분산해서(또는 장애 시 우회해서) 보내게 되는 것입니다!
