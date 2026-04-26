## eBPF 프로그램 유형

- 커널은 eBPF 프로그램을 아무 데서나 실행하게 두지 않는다.
- 어디에 붙일지에 따라 허용되는 기능/컨텍스트가 달라지므로,
- 이 eBPF 코드는 어떤 이벤트/훅에서 실행될 프로그램인가를 명시해야 하는데
- 이게 바로 BPF 프로그램 유형이다.

## 패킷 경로

```bash
[ NIC Hardware ]
패킷 수신
   ↓
DMA로 RX Ring 기록
   ↓
IRQ 발생
   ↓
[ Driver IRQ Handler ]
napi_schedule()
   ↓
[ SoftIRQ ]
커널이 Driver의 napi_poll 호출
   ↓
[ Driver napi_poll ]
RX Ring 읽기
skb 생성
   ↓
Linux Networking Stack 전달
```

- XDP가 붙는 시점: napi_poll 내부에서 링버퍼의 packet descriptor를 읽은 직후 skb를 만들기 전
- TC가 붙는 시점: skb 생성 후 아직 본격 적인 IP 라우팅 전

```bash
napi_poll()이 RX Ring에서 패킷 batch 처리
 ↓
(이 시점에 XDP Hook 가능)
 ↓
XDP_PASS면 skb 생성
 ↓
(이후 TC Ingress Hook 가능)
 ↓
Linux Networking Stack 진입
 ↓
L2 / L3 / Routing / Netfilter / Socket 처리
```

## skb란

- raw packet buffer를 Linux가 이해하는 packet metadata 객체로 래핑하는 역할
- skb는 패킷 자체를 다른 형식으로 변환하는 것이 아니라
- 패킷 데이터 버퍼를 참조하면서 Linux 네트워크 스택이 활용할 메타데이터/상태 정보를 담는 래퍼 객체이다.

## Program Type

| Program Type | Attach 위치 | 용도 |
| --- | --- | --- |
| XDP | NIC Driver RX | 초고속 packet 처리 |
| TC | Traffic Control | ingress/egress packet 처리 |
| Kprobe | Kernel Function Entry | 커널 함수 추적 |
| Kretprobe | Kernel Function Return | 함수 반환 추적 |
| Tracepoint | Static Kernel Trace Hook | 안정적 tracing |
| Perf Event | Performance Event | profiling |
| Cgroup SKB | Cgroup Net Hooks | cgroup별 네트워크 제어 |
| Socket Filter | Socket Layer | 패킷 필터링 |
| SockOps | TCP Socket Ops | TCP connection tuning |
| LSM | Linux Security Module Hook | 보안 정책 |
| Uprobe | User-space Function Hook | 유저 프로세스 tracing |

## XDP

- eXpress Data Path 약자
- 네트워크 드라이버가 패킷을 수신하는 순간, 즉 소프트웨어상 가능한 가장 빠른 시점에 BPF 프로그램을 실행한다.
- 패킷을 네트워크 스택 상위로 전달하기 위한 메모리 할당이나 GRO 엔진으로 패킷을 전송하는 등의 비용이 많이 드는 작업을 수행하지 않은 단계
- 리눅스 네트워크 스택에서 드랍 여부를 판단하기 전에 skb를 만드므로 버릴 패킷인데도 skb를 만드는 경우가 있음
- 이때 XDP는 드라이버가 패킷 받자마자 skb 생성 전에 패킷을 가로채서 초고속 의사결정을 내린다.

### 수행 과정

BPF 프로그램에 아래와 같은 객체가 전달되어, XDP는 DMA Buffer 안의 Raw Packet Bytes를 직접 읽는다.

```bash
struct xdp_buff {
    void *data;
    void *data_end;
    void *data_meta;
    void *data_hard_start;
    struct xdp_rxq_info *rxq;
};

```

이를 통해 BPF 맵 등을 참조하여

- Ethernet Header 파싱
- IP Header 파싱
- TCP/UDP Port 확인
- Payload 일부 검사

를 수행해서 이 패킷을 어떻게 처리할지에 대한 판정을 반환한다.

```bash
enum xdp_action {
    XDP_ABORTED = 0,
    XDP_DROP,
    XDP_PASS,
    XDP_TX,
    XDP_REDIRECT,
};

```

1. XDP_DROP
    - 즉시 폐기
    - skb 생성 안 함, 네트워크 스택 안 감
    - DDOS 공격 같은 경우
2. XDP_PASS
    - 정상 Linux 스택으로 전달
    - 이제 skb 생성하고 GRO 엔진으로 전달하는 등 XDP를 사용하지 않을 때의 기본 패킷 처리 동작과 동일
    - 일반 정상 트래픽인 경우
3. XDP_TX
    - 들어온 NIC를 통해 네트워크 패킷을 바로 재전송
    - Hairpin Load Balancer
        
        ```bash
        클라이언트 → LB Node 도착
        LB가 Backend로 목적지 IP rewrite
        같은 NIC로 다시 송신
        ```
        
4. XDP_REDIRECT
    - 다른 NIC / CPU / AF_XDP socket 등으로 전달
    - NIC 포워딩(같은 서버 내 다른 인터페이스 → 다른 노드/네트워크로 나가도록), CPU 분산 등이 필요한 경우

## TC

- Generic Networking Stack Hook으로 NIC 수신 경로에 attach되는 XDP와 달리 범용적인 처리를 수행하여
- Ingress, Egress 두 방향 모두에서 활용된다.

### TC의 필요성

XDP는 패킷이 skb(sk_buff)로 변환되기 이전 단계에서 실행되므로,

고성능 패킷 처리에는 유리하지만 다음과 같은 커널 네트워크 스택 메타데이터에 접근할 수 없다.

- skb metadata (mark, priority, protocol, VLAN 정보 등)
- Conntrack state
- Socket context
- Routing / Forwarding state

따라서 XDP는 빠른 Drop / Redirect / 단순 필터링에는 적합하지만,

상태 기반 추적이나 복잡한 패킷 조작이 필요한 네트워크 기능 구현에는 제약이 있다

### 수행 과정

TC BPF는 skb 생성 이후 Linux 네트워크 스택의 ingress/egress 훅에서 실행되며,

커널이 생성한 `__sk_buff` 컨텍스트를 통해 패킷 데이터와 skb 메타데이터를 모두 활용할 수 있다.

이를 기반으로 다음과 같은 고수준 네트워크 처리가 가능하다.

- 패킷 데이터 읽기 및 수정
- skb metadata 읽기 및 수정
- Drop / Redirect
- NAT / Header Rewrite
- Stateful Packet Processing

### Cilium과의 연관성

Cilium이 제공하는 주요 기능들은 대부분 풍부한 패킷 메타데이터와 상태 기반 네트워크 처리를 요구한다.

- NetworkPolicy Enforcement
- Service Load Balancing
- Connection Tracking
- NAT / SNAT / DNAT
- Pod Routing / Forwarding
- Overlay / Tunnel Encapsulation

따라서 Cilium의 핵심 datapath는 TC 기반으로 구성되고, XDP는 고성능 Fast Path 최적화를 위해 선택적으로 추가된다.

## Cilium datapath 프로그램

Cilium은 여러 eBPF 프로그램을 준비해 두어 이를 네트워크 경로의 적절한 Hook Point에 Attach하여 동작한다.

```bash
Cilium Datapath Logic
 ├─ XDP Hook용 Program
 ├─ TC Ingress Hook용 Program
 └─ TC Egress Hook용 Program
```

### TC Ingress Hook

Attach 위치는 크게 두 가지이다.

- Host NIC ingress
- Pod veth ingress

이때 Host와 Pod 처리 목적이 다르므로 Host용과 Pod용으로 프로그램이 따로 구성된다.

#### 1. Host NIC Ingress

외부/다른 노드에서 들어온 패킷 처리:

→ 어느 Pod/Service 대상인지 판단

→ LB / DNAT / Routing

#### 2. Pod veth Ingress

Pod로 들어가는 패킷 처리:

→ 이 Pod가 이 패킷 받아도 되는지 검사

→ Ingress Policy Enforcement