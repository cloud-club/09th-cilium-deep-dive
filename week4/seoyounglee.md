# eBPF와 Cilium

## eBPF란 무엇인가

- 운영체제 커널과 같은 특별한 권한이 있는 환경에서 샌드박스 프로그램을 실행시킬 수 있게 해주는 커널 기술
- 샌드박스 프로그램이란 격리된 환경에서 프로그램을 실행시키는 것

운영 체제 커널은 강력한 권한을 가지고 있지만 동시에 안정성과 보안에 중요한 역할을 하여 빠르게 진화하기는 어렵다

하지만 eBPF 프로그램을 통해 커널 코드를 수정하지 않고도 기존의 운영 체제(커널 내부의 특정 지점)에 프로그램을 붙여 실행함으로써 추가적인 기능을 추가할 수 있게 됨

- eBPF = 기술
- eBPF program = 그 기술로 커널에 붙여 실행하는 코드

네트워크, 보안, observablility 등 다양한 영역에서 사용된다.

---

## Cilium이란

eBPF 기반 Kubernetes CNI(Container Network Interface)로, 워크로드 간 네트워크 연결(Networking)과 네트워크 관측 기능(Observability)을 모두 제공한다.

일반적인 CNI는 Linux 커널의 iptables를 기반으로 패킷을 처리한다.

반면 Cilium은 kube-proxy 및 iptables에 대한 의존도를 줄이고, Cilium Agent가 Kubernetes 상태를 기반으로 eBPF 프로그램을 생성하여 이를 커널에 로드함으로써 패킷 처리(라우팅, NAT, 로드밸런싱 등)를 수행한다.

```go
Pod -> iptables -> routing -> Pod
가 아닌
Pod -> eBPF program -> Pod
```

이를 통해 패킷 처리를 커널 내부에서 직접 수행할 수 있으며, iptables 기반 방식보다 더 낮은 지연과 높은 성능을 제공한다.

---

## Cilium의 3계층 구조

Cilium은 크게 다음 3가지 계층으로 구성된다.

1. Control Plane: Cilium Agent
2. Datapath: eBPF Program
3. Observability: Hubble

---

### 1. Control Plane: Cilium Agent

Cilium Agent는 각 노드에서 실행되는 컨트롤 플레인 컴포넌트로, 클러스터의 상태를 기반으로 eBPF 프로그램을 구성하고 관리한다.

**Kubernetes 상태 감시**

Cilium Agent는 Kubernetes API 서버를 watch하면서 다음과 같은 리소스를 감시한다.

- NetworkPolicy
- Service
- Endpoint/EndpointSlice
- Pod

이를 통해 클러스터 네트워크 상태 변화를 지속적으로 수신한다.

**eBPF 프로그램 및 Map 관리**

Agent는 수신한 상태 정보를 기반으로 커널에 로드될 eBPF 프로그램과 Map을 구성한다.

```yaml
Kubernetes API watch
↓
NetworkPolicy / Service 정보 수신
↓
eBPF bytecode 생성
↓
노드 커널에 로드 및 attach
```

실제 동작에서는 eBPF 프로그램을 매번 새로 생성하기보다는,

**미리 로드된 프로그램을 유지하고 eBPF Map을 업데이트하는 방식으로 상태를 반영한다.**

예를 들어 새로운 Pod가 생성되면,

- Cilium Agent가 Pod의 네트워크 인터페이스를 감지하고
- 해당 Pod에 적용될 정책을 계산하여
- eBPF Map을 업데이트를 통해 정책 반영

즉, Cilium Agent는 Kubernetes 클러스터 상태를 기반으로 eBPF 프로그램과 Map을 관리하여 datapath를 동적으로 구성하는 역할을 수행한다.

### 2. Datapath: eBPF Program

Datapath란 실제 패킷이 흐르는 경로에서 패킷을 처리하는 영역이며, Cilium에서 이는 Linux 커널에서 실행되는 eBPF 프로그램으로 구현된다.

**eBPF 프로그램 attach 위치**

- Pod의 네트워크 인터페이스 (veth)
- Node의 네트워크 인터페이스
- TC (Traffic Control) hook 또는 XDP

**주요 기능**

- NetworkPolicy 검사
- Service Load Balancing
- Routing
- NAT 처리
- 방화벽 필터링

이러한 처리는 커널 내부에서 수행되기 때문에 기존 iptables 기반 방식보다 컨텍스트 스위칭이 줄어들고 지연이 낮다.

또한 eBPF는 커널 재부팅 없이도 프로그램을 교체할 수 있어 Cilium Agent를 통해 네트워크 로직을 동적으로 업데이트할 수 있다.

**Flow Event 생성**

또한, eBPF datapath는 패킷 처리 과정에서 네트워크 flow 정보를 이벤트 형태로 생성할 수 있다. 

```yaml
source: podA
destination: kube-dns
protocol: UDP
port: 53
verdict: DROPPED
reason: Policy denied
```

이제 이러한 이벤트는 이후에 observability 계층에서 수집된다.

### 3. Observability: Hubble

Hubble은 Cilium의 네트워크 observability layer로, eBPF 프로그램에서 생성된 flow 이벤트를 수집하여 사용자에게 제공한다.

Hubble은 **패킷을 직접 캡처하는 도구가 아니다.**

Hubble은 다음과 같은 구조로 동작한다.

```yaml
eBPF datapath
↓
flow event 생성
↓
Cilium Agent
↓
Hubble
↓
CLI / UI / metrics exporter
```

즉,

1. eBPF datapath(kernel)가 패킷 처리 중 flow event 생성 후 ring buffer에 기록
2. Cilium Agent(user space)가 커널로부터 flow event를 수집해서 가공하고 이를 상위 계층에 전달
3. Agent 내부 기능인 Hubble에서 이벤트 스트림을 관리하고 관찰 기능을 제공한다.

Hubble은 또한 이 데이터를 CLI, Prometheus metrics exporter를 통해 활용할 수 있도록 제공한다.

---

### 전체 구조

정리하면 Cilium 아키텍처는 다음과 같이 동작한다.

```bash
Kubernetes API
↓
Cilium Agent (Control Plane)
↓
eBPF Program 설치
↓
Datapath에서 패킷 처리
↓
Flow Event 생성
↓
Hubble 수집
↓
Metrics / Flow Observability 제공
```

---

## 현재 클러스터의 Cilium의 동작 모드

```bash
root@control-plane:~# cilium config view | grep kube-proxy
kube-proxy-replacement                            false
```

현재 클러스터는 Cilium이 kube-proxy를 대체하지 않는 모드로 동작하고 있다.

### 동작 구조

이 구성에서는 Kubernetes의 기본 네트워크 구조가 유지된다.

- Service 트래픽 처리는 kube-proxy가 담당하고,
- Cilium은
    - Pod 네트워크 구성 (CNI)
    - NetworkPolicy enforcement (eBPF)
    - Pod 간 트래픽 처리 및 관찰

### Service 트래픽 처리 흐름

Service로 트래픽이 유입될 경우의 흐름은 아래와 같은데

```bash
Client → Service(ClusterIP) → kube-proxy → Pod
```

1. kube-proxy가 Service/Endpoint 정보를 기반으로
iptables 또는 IPVS rule을 생성한다.
2. Service IP로 들어온 트래픽은
kube-proxy에 의해 **Pod IP로 DNAT** 된다.
3. 이후 패킷은 해당 Pod로 전달된다.

이때 Cilium은 kube-proxy에 의해 DNAT이 완료된 이후의 패킷 경로에서 동작하며,

해당 트래픽에 대해 eBPF 기반의 네트워크 처리 및 NetworkPolicy를 적용한다.

즉, 

- Service → Pod 매핑 (DNAT): kube-proxy 담당
- DNAT 이후 Pod 간 통신