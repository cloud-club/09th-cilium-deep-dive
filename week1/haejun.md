# Linux Network Deep Dive (Kernel Level)

## 패킷이 커널을 통과하는 실제 흐름과 내부 동작

---

## 1. 들어가며

리눅스 네트워크를 제대로 이해하기 위해서는 “계층 모델”이 아니라 **패킷이 커널 내부를 어떻게 흐르는지**를 기준으로 접근해야 합니다.

이 문서는 다음 질문에 답하는 것을 목표로 합니다.

* 패킷은 커널 내부에서 어떤 경로를 따라 이동하는가
* 각 단계에서 어떤 커널 함수와 구조체가 관여하는가
* 실제 네트워크 문제는 어디에서 발생하는가

---

## 2. 전체 흐름 개요

송신과 수신을 기준으로 나누어 이해하는 것이 가장 효과적입니다.

### 송신 흐름

```text
Application
 → syscall (send)
 → Socket Layer
 → TCP/UDP
 → IP
 → Routing
 → Netfilter
 → Qdisc
 → NIC Driver
 → Hardware
```

### 수신 흐름

```text
NIC
 → Interrupt / NAPI
 → netif_receive_skb
 → Netfilter (PREROUTING)
 → Routing
 → TCP/UDP
 → Socket Queue
 → recv()
```

---

## 3. 핵심 데이터 구조: sk_buff

리눅스 네트워크 스택의 중심에는 `sk_buff`가 존재합니다.

```c
struct sk_buff {
    unsigned char *data;
    unsigned int len;
    struct net_device *dev;
    struct sock *sk;
}
```

### 역할

* 패킷의 메타데이터와 payload를 함께 관리
* 각 계층 간 전달되는 단위 객체

### 중요한 특징

* 데이터 복사를 최소화하기 위해 포인터 기반으로 이동
* headroom / tailroom 구조를 통해 헤더 추가/제거 가능

> 커널 내부에서 “패킷”은 항상 sk_buff 형태로 존재합니다.

---

## 4. 송신 경로 상세 분석

### 4.1 User Space → Kernel 진입

```c
send(fd, buf, len, 0);
```

내부 호출 흐름:

```text
sys_sendto
 → sock_sendmsg
 → tcp_sendmsg
```

---

### 4.2 TCP Layer

```text
tcp_sendmsg()
```

주요 동작:

* 사용자 데이터를 MSS 단위로 분할
* sequence number 할당
* congestion control 적용
* retransmission queue 등록

관련 구조체:

```c
struct tcp_sock {
    u32 snd_nxt;
    u32 snd_una;
}
```

---

### 4.3 IP Layer

```text
ip_queue_xmit()
```

주요 역할:

* IP header 생성
* routing lookup 수행

---

### 4.4 Routing

```text
fib_lookup()
```

결정 요소:

* destination IP
* routing table
* policy routing (rule)

결과:

* next hop
* output interface

---

### 4.5 Netfilter

Hook 지점:

```text
OUTPUT → POSTROUTING
```

주요 기능:

* SNAT / DNAT
* packet filtering

---

### 4.6 Qdisc (Traffic Control)

```text
dev_queue_xmit()
 → qdisc_enqueue()
```

역할:

* 패킷 큐잉
* 스케줄링
* rate limiting

---

### 4.7 NIC Driver

```text
ndo_start_xmit()
```

* DMA를 통해 NIC로 전달
* 하드웨어 큐에 적재

---

## 5. 수신 경로 상세 분석

### 5.1 NIC → Kernel

패킷 수신 시:

* 인터럽트 발생
* NAPI로 전환

---

### 5.2 NAPI

목적:

* interrupt storm 방지
* polling 기반 처리

---

### 5.3 패킷 전달

```text
netif_receive_skb()
```

이 함수가 네트워크 스택 진입점 역할을 수행합니다.

---

### 5.4 Netfilter

```text
PREROUTING
```

가능한 작업:

* DNAT
* firewall

---

### 5.5 Routing Decision

```text
ip_rcv()
 → ip_route_input_noref()
```

결과:

* LOCAL → INPUT
* FORWARD → 다른 인터페이스

---

### 5.6 TCP 처리

```text
tcp_v4_rcv()
```

역할:

* sequence 검증
* ACK 처리
* out-of-order 처리

---

### 5.7 Socket 전달

```text
sock_queue_rcv_skb()
```

이후 사용자 영역에서:

```text
recv()
```

---

## 6. Netfilter 내부 구조

Hook 체인:

```text
PREROUTING
INPUT
FORWARD
OUTPUT
POSTROUTING
```

### 흐름 요약

* 수신: PREROUTING → INPUT
* 송신: OUTPUT → POSTROUTING

---

## 7. Neighbor & ARP

IP 기반 통신에서 MAC 주소는 필수입니다.

```text
neigh_lookup()
```

동작:

* ARP cache 조회
* 없으면 ARP request 발생

---

## 8. 성능 관련 핵심 메커니즘

### 8.1 SoftIRQ

```text
NET_RX_SOFTIRQ
NET_TX_SOFTIRQ
```

* 인터럽트 처리 분산
* 네트워크 처리 비동기화

---

### 8.2 Zero Copy

* sendfile()
* splice()

효과:

* CPU copy 감소
* 성능 향상

---

### 8.3 Offloading

NIC이 일부 작업 수행

* TSO (TCP segmentation offload)
* GRO (receive aggregation)

---

## 9. 실제 병목 발생 지점

| 위치    | 문제                 |
| ----- | ------------------ |
| TCP   | retransmission     |
| Qdisc | queue overflow     |
| NIC   | interrupt overload |
| User  | blocking I/O       |

---

## 10. 디버깅 관점

### 주요 명령어

```bash
ss -tulnp
tcpdump -i eth0
ip route
ip neigh
ethtool -k eth0
```

---

### 관찰 포인트

* packet drop 위치
* retransmission 발생 여부
* queue backlog 상태

---

## 11. Kubernetes와의 연결

리눅스 네트워크 이해는 Kubernetes 네트워크 이해의 전제입니다.

### Pod 통신

```text
Pod
 → veth
 → bridge / CNI
 → iptables / eBPF
```

---

### 핵심

* 모든 CNI는 결국 Linux networking 위에서 동작
* eBPF는 커널 datapath를 직접 수정

---

## 12. 핵심 요약

* 리눅스 네트워크는 계층이 아니라 **데이터 흐름 기반**입니다
* 모든 패킷은 `sk_buff`로 처리됩니다
* 각 계층은 함수 호출 체인으로 연결됩니다
* Netfilter, routing, qdisc 등 현실 기능이 포함됩니다

---

## 부록. 리눅스 커널 네트워크 코드 구조

리눅스 네트워크 스택은 커널 소스 코드의 특정 디렉토리에 모여 있으며, 각 디렉토리는 역할에 따라 구분되어 있습니다.

---

### 1. 주요 디렉토리

리눅스 커널에서 네트워크 관련 코드는 다음 세 위치에 존재합니다.

```bash
net/                # 네트워크 스택 핵심 로직
drivers/net/        # NIC 드라이버
include/net/        # 네트워크 관련 헤더 및 구조체
```

---

### 2. net/ 디렉토리 (핵심)

`net/` 디렉토리는 실제 네트워크 프로토콜과 데이터 흐름이 구현된 핵심 영역입니다.

주요 하위 디렉토리는 다음과 같습니다.

```bash
net/
├── core/        # 공통 인프라 (sk_buff, device, socket)
├── ipv4/        # TCP, UDP, IP, Routing
├── netfilter/   # iptables / nftables
├── sched/       # Traffic Control (Qdisc)
├── bridge/      # L2 bridge
```

---

### 3. 주요 코드와 역할 매핑

앞서 살펴본 네트워크 흐름은 다음과 같이 커널 코드와 대응됩니다.

#### 송신 경로

```text
tcp_sendmsg()     → net/ipv4/tcp_output.c
ip_queue_xmit()   → net/ipv4/ip_output.c
fib_lookup()      → net/ipv4/route.c
dev_queue_xmit()  → net/core/dev.c
ndo_start_xmit()  → drivers/net/*
```

---

#### 수신 경로

```text
netif_receive_skb() → net/core/dev.c
ip_rcv()            → net/ipv4/ip_input.c
tcp_v4_rcv()        → net/ipv4/tcp_input.c
```

---

### 4. 핵심 정리

* 네트워크 스택의 핵심 구현은 `net/` 디렉토리에 존재합니다
* TCP/IP 관련 로직은 `net/ipv4/`에서 확인할 수 있습니다
* 패킷 송수신의 시작과 끝은 `net/core/dev.c`에서 이루어집니다
* 실제 하드웨어 전송은 `drivers/net/`에서 처리됩니다

---

### 5. 코드 분석 접근 방법

커널 네트워크 코드를 분석할 때는 디렉토리 구조보다 **패킷 흐름을 기준으로 함수 호출을 따라가는 방식**이 효과적입니다.

추천 순서는 다음과 같습니다.

1. `dev.c` (패킷 입출력 흐름)
2. `tcp_output.c`, `tcp_input.c`
3. `ip_output.c`, `ip_input.c`
4. `route.c`

---
