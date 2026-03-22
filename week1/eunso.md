# Week 1: Linux Network 기본기

> **목적:** Cilium/eBPF 기반 CNI의 datapath를 이해하기 위한 Linux 네트워크 선행 지식 정리.
> 패킷이 NIC에서 애플리케이션 소켓까지 커널을 어떻게 통과하는지, 그 경로에서 작동하는 메커니즘을 이해한다.

---

## 1. Linux TCP/IP Stack

### 1.1 커널 레이어 구조

Linux는 TCP/IP 스택을 다음과 같은 레이어로 구현한다.

```
┌──────────────────────────────────────┐
│         Application (User Space)     │  socket syscall
├──────────────────────────────────────┤
│         Socket Layer (INET)          │  sock, proto_ops
├──────────────────────────────────────┤
│      Transport Layer (TCP/UDP)       │  tcp_v4_rcv, udp_rcv
├──────────────────────────────────────┤
│       Network Layer (IP)             │  ip_rcv, ip_output
├──────────────────────────────────────┤
│  Netfilter Hooks (iptables/conntrack)│
├──────────────────────────────────────┤
│     Link Layer / Net Device          │  net_device, NAPI
├──────────────────────────────────────┤
│       NIC Driver (Kernel Space)      │
└──────────────────────────────────────┘
```

이 구조는 RFC 1122 ("Requirements for Internet Hosts -- Communication Layers")의 모델을 Linux 커널이 구현한 형태다.

### 1.2 sk_buff (Socket Buffer)

패킷은 커널 내에서 `sk_buff` 구조체로 표현된다. 이 구조체는 패킷 데이터를 직접 담는 것이 아니라, 데이터를 가리키는 포인터와 각 레이어의 헤더 위치를 추적하는 메타데이터를 담는다.

```c
struct sk_buff {
    /* 데이터 포인터 */
    unsigned char    *head;   /* 버퍼 시작 */
    unsigned char    *data;   /* 현재 데이터 시작 (레이어별로 이동) */
    unsigned char    *tail;   /* 데이터 끝 */
    unsigned char    *end;    /* 버퍼 끝 */

    /* 네트워크/전송 헤더 위치 */
    struct iphdr     *nh.iph;
    struct tcphdr    *h.th;

    /* 메타데이터 */
    struct net_device *dev;   /* 수신/송신 net_device */
    __u32            priority;
    /* ... */
};
```

패킷이 레이어를 내려갈수록 `data` 포인터가 앞으로 이동하며 헤더가 추가(prepend)된다. 올라갈수록 `data`가 뒤로 이동하며 헤더가 소비(consume)된다.

참고: [Linux Kernel Source: include/linux/skbuff.h](https://elixir.bootlin.com/linux/latest/source/include/linux/skbuff.h)

---

## 2. 패킷 수신 경로 (RX Path)

패킷이 NIC에서 애플리케이션 소켓까지 전달되는 경로.

```
NIC (Hardware)
  │  DMA → Ring Buffer
  ▼
IRQ Handler → NAPI poll (softirq: NET_RX_SOFTIRQ)
  │  netif_receive_skb()
  ▼
Protocol Demux (__netif_receive_skb)
  │  ethertype 기반 dispatch
  ▼
ip_rcv()                          ← L3 진입
  │  PREROUTING hook (Netfilter)
  ▼
ip_rcv_finish()
  │  routing decision (ip_route_input)
  ├─ [로컬 수신] → ip_local_deliver()
  │     │  INPUT hook (Netfilter)
  │     ▼  ip_local_deliver_finish()
  │        transport layer dispatch
  │        tcp_v4_rcv() / udp_rcv()
  │          └─ 소켓 큐에 삽입 → application read()
  │
  └─ [포워딩] → ip_forward()
        │  FORWARD hook (Netfilter)
        ▼  ip_output()
           POSTROUTING hook (Netfilter)
```

### 2.1 NAPI (New API)

고속 트래픽에서 인터럽트를 매 패킷마다 발생시키면 오버헤드가 크다. NAPI는 첫 패킷에서 인터럽트를 발생시키고 이후 softirq 컨텍스트에서 poll 방식으로 배치 처리한다.

참고: [kernel.org: NAPI](https://docs.kernel.org/networking/napi.html)

---

## 3. 패킷 송신 경로 (TX Path)

```
Application: write() / sendmsg()
  ▼
tcp_sendmsg()
  │  세그먼트 생성, sk_buff 할당
  ▼
tcp_transmit_skb()
  │  TCP 헤더 추가
  ▼
ip_queue_xmit()
  │  IP 헤더 추가, routing
  │  OUTPUT hook (Netfilter)
  ▼
ip_output() → ip_finish_output()
  │  POSTROUTING hook (Netfilter)
  │  fragmentation if needed
  ▼
dev_queue_xmit()
  │  qdisc (트래픽 쉐이핑)
  ▼
NIC Driver: ndo_start_xmit()
  │  DMA to NIC Ring Buffer
  ▼
NIC (Hardware) → 물리 네트워크
```

---

## 4. Netfilter Framework

Netfilter는 Linux 커널의 패킷 필터링/조작 프레임워크다. 커널 네트워크 스택의 특정 지점에 hook을 제공하고, 모듈(iptables, conntrack 등)이 이 hook에 콜백을 등록하여 패킷을 처리한다.

### 4.1 Hook Points

IPv4 기준 5개의 hook point가 있다.

| Hook | 위치 | 동작 |
|------|------|------|
| `NF_INET_PRE_ROUTING` | ip_rcv() 직후, routing 전 | 모든 수신 패킷 |
| `NF_INET_LOCAL_IN` | 로컬 소켓으로 전달 직전 | 로컬 수신 패킷 |
| `NF_INET_FORWARD` | 포워딩 결정 후 | 포워딩 패킷 |
| `NF_INET_LOCAL_OUT` | ip_output() 직전 | 로컬 생성 송신 패킷 |
| `NF_INET_POST_ROUTING` | 물리 인터페이스 전송 직전 | 모든 송신 패킷 |

```
      [PREROUTING]────[routing]──┬──[FORWARD]──[POSTROUTING]──▶ out
                                 │
                             [LOCAL IN]
                                 │
                             Application
                                 │
                             [LOCAL OUT]─────────────[POSTROUTING]──▶ out
```

### 4.2 Hook 등록 메커니즘

```c
/* net/netfilter/core.c */
int nf_register_net_hook(struct net *net, const struct nf_hook_ops *reg);

struct nf_hook_ops {
    nf_hookfn       *hook;      /* 콜백 함수 */
    struct net_device *dev;
    u_int8_t        pf;         /* 프로토콜 패밀리 (NFPROTO_IPV4 등) */
    unsigned int    hooknum;    /* hook point (NF_INET_PRE_ROUTING 등) */
    int             priority;   /* 낮을수록 먼저 실행 */
};
```

hook 콜백은 다음 verdict 중 하나를 반환한다:
- `NF_ACCEPT`: 패킷 계속 처리
- `NF_DROP`: 패킷 폐기
- `NF_STOLEN`: 패킷 소유권 이전 (모듈이 자체 처리)
- `NF_QUEUE`: userspace로 전달
- `NF_REPEAT`: hook 재실행

참고:
- [netfilter.org 공식 문서](https://www.netfilter.org/documentation/)
- [Netfilter Hacking HOWTO](https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO.html)
- [Linux Kernel Source: net/netfilter/](https://elixir.bootlin.com/linux/latest/source/net/netfilter)

---

## 5. iptables

iptables는 Netfilter hook에 등록되는 가장 널리 사용되는 패킷 필터링/조작 도구다. **table → chain → rule** 의 계층 구조를 가진다.

### 5.1 Table 종류와 Hook 매핑

각 table은 특정 hook point에만 chain을 갖는다. 우선순위(priority)가 낮을수록 먼저 실행된다.

| Table | 목적 | Chain (우선순위) |
|-------|------|-----------------|
| `raw` | conntrack 이전 처리, NOTRACK | PREROUTING(-300), OUTPUT(-300) |
| `mangle` | 패킷 필드 수정 (TTL, TOS 등) | PREROUTING(-150), INPUT(-150), FORWARD(-150), OUTPUT(-150), POSTROUTING(-150) |
| `nat` | 주소 변환 (DNAT/SNAT/MASQUERADE) | PREROUTING(-100), INPUT(100), OUTPUT(-100), POSTROUTING(100) |
| `filter` | 패킷 허용/차단 (기본 table) | INPUT(0), FORWARD(0), OUTPUT(0) |

한 hook에서의 처리 순서 (PREROUTING 예시):
```
PREROUTING hook 도달:
  1. raw table        (-300 우선순위)
  2. conntrack        (-200 우선순위, 내부적으로)
  3. mangle table     (-150 우선순위)
  4. nat table DNAT   (-100 우선순위)
```

### 5.2 Rule 처리 흐름

```
패킷 도달
  │
  ▼
Chain의 첫 번째 rule 매칭 시도
  ├─ 매칭 → target 실행
  │     ├─ ACCEPT: 다음 hook으로 전달
  │     ├─ DROP: 폐기
  │     ├─ RETURN: 이 chain 종료, 호출 chain으로 복귀
  │     └─ 다른 chain 점프 (사용자 정의 chain)
  └─ 미매칭 → 다음 rule
        └─ 모든 rule 소진 → chain의 기본 policy 적용
```

### 5.3 주요 Match / Target

**Match 예시:**
```bash
-p tcp --dport 80                          # TCP 80번 포트
-s 10.0.0.0/8                             # 출발지 IP
-m conntrack --ctstate ESTABLISHED,RELATED # conntrack 상태
-m multiport --dports 80,443              # 복수 포트
```

**Target 예시:**
```bash
-j ACCEPT                                 # 허용
-j DROP                                   # 폐기 (응답 없음)
-j REJECT                                 # 폐기 + ICMP/TCP RST 응답
-j DNAT --to-destination 192.168.1.10:8080 # 목적지 변환
-j MASQUERADE                             # 동적 SNAT (출구 인터페이스 IP로)
-j LOG --log-prefix "DROPPED: "           # 커널 로그
```

참고:
- [iptables man page](https://linux.die.net/man/8/iptables)
- [netfilter.org: iptables HOWTO](https://www.netfilter.org/documentation/HOWTO/iptables-HOWTO.html)
- [Linux Kernel Source: net/ipv4/netfilter/](https://elixir.bootlin.com/linux/latest/source/net/ipv4/netfilter)

---

## 6. Connection Tracking (conntrack)

conntrack은 Netfilter의 connection tracking 서브시스템으로, 네트워크 연결의 상태를 추적한다. NAT, stateful 방화벽의 기반이 된다.

### 6.1 Connection State

| 상태 | 의미 |
|------|------|
| `NEW` | 새 연결 (기존 추적 없음) |
| `ESTABLISHED` | 양방향 패킷이 오간 연결 |
| `RELATED` | 기존 연결과 연관된 새 연결 (예: FTP data, ICMP error) |
| `INVALID` | 알 수 없는 상태, 추적 불가 |
| `UNTRACKED` | `raw` table에서 NOTRACK된 패킷 |

### 6.2 내부 동작 흐름

conntrack은 PREROUTING(-200)과 LOCAL_OUT(-200) hook에 등록되어 iptables보다 먼저 실행된다.

```
패킷 수신 → nf_conntrack_in()
  │
  ├─ 해시 테이블에서 기존 connection 탐색
  │    키: (src_ip, dst_ip, src_port, dst_port, proto, netns)
  │
  ├─ [기존 연결] → 상태 업데이트 (ESTABLISHED 등)
  │
  └─ [신규] → nf_conntrack_tuple 생성
        │  원본(original) + 응답(reply) 방향 tuple 쌍 생성
        └─ 해시 테이블에 삽입 (ct entry 확정은 POSTROUTING 후)

패킷 처리 완료 → conntrack entry confirmed 상태로 전환
```

### 6.3 NAT와 conntrack의 관계

DNAT/SNAT는 conntrack entry에 NAT 정보를 기록하여 이후 패킷에 자동 적용한다. conntrack이 없으면 stateful NAT는 불가능하다.

```
PREROUTING에서 DNAT 적용:
  10.0.0.1:12345 → 10.96.0.1:80 (Service VIP)
         ↓ conntrack entry 기록
  10.0.0.1:12345 → 192.168.1.10:8080 (실제 Pod)

응답 패킷은 conntrack이 자동으로 역방향 SNAT:
  192.168.1.10:8080 → 10.0.0.1:12345
         ↓ reply tuple 매칭 → 역변환 자동 적용
  10.96.0.1:80 → 10.0.0.1:12345
```

### 6.4 conntrack 확인 명령

```bash
# 현재 conntrack 테이블 조회
conntrack -L

# 실시간 이벤트 모니터링
conntrack -E

# 특정 연결 검색
conntrack -L --src 10.0.0.1

# /proc 직접 조회
cat /proc/net/nf_conntrack
```

출력 예시:
```
tcp  6  86400  ESTABLISHED  src=10.0.0.1  dst=10.96.0.1  sport=12345  dport=80 \
     src=192.168.1.10  dst=10.0.0.1  sport=8080  dport=12345  [ASSURED]
```
왼쪽이 original 방향 tuple, 오른쪽이 reply 방향 tuple.

참고:
- [kernel.org: nf_conntrack sysctl](https://www.kernel.org/doc/html/latest/networking/nf_conntrack-sysctl.html)
- [Cloudflare Blog: Conntrack tales - one thousand and one ways to break your Linux connectivity](https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-ways-to-break-your-linux-connectivity/)
- [Linux Kernel Source: net/netfilter/nf_conntrack_core.c](https://elixir.bootlin.com/linux/latest/source/net/netfilter/nf_conntrack_core.c)

---

## 7. 전체 패킷 흐름 (종합 다이어그램)

```
  NIC ──DMA──▶ Ring Buffer ──NAPI poll──▶ netif_receive_skb()
                                                 │
                                         [PREROUTING hook]
                                          raw → conntrack → mangle → nat(DNAT)
                                                 │
                                         [routing decision]
                                        /                   \
                           (local dest)                  (forward)
                                │                             │
                        [INPUT hook]                  [FORWARD hook]
                         mangle, filter                mangle, filter
                                │                             │
                         tcp_v4_rcv()                         │
                         udp_rcv()                    [POSTROUTING hook]
                                │                      mangle, nat(SNAT)
                         Application                         │
                         socket recv                    dev_queue_xmit
                                                             │
                                                         NIC ──▶ Network
  Application
  send()
    │
  tcp_sendmsg()
    │
  [OUTPUT hook]
   raw → conntrack → mangle → nat(DNAT) → filter
    │
  [POSTROUTING hook]
   mangle → nat(SNAT)
    │
  dev_queue_xmit ──▶ NIC ──▶ Network
```

**출처:** [Netfilter Packet Traversal](https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO-3.html)

---

## 8. Kubernetes / Cilium과의 연결

| 개념 | Kubernetes/Cilium에서의 역할 |
|------|------------------------------|
| Netfilter hook | kube-proxy는 PREROUTING/OUTPUT hook에 iptables 규칙 삽입으로 Service VIP → Pod IP 변환 |
| conntrack | Service의 DNAT 세션 유지, stateful NAT 구현 |
| iptables nat table | ClusterIP Service의 로드밸런싱 구현 (DNAT) |
| Cilium | eBPF로 위 기능을 대체. conntrack도 자체 BPF map으로 구현하여 성능 향상 |

**Cilium이 iptables를 우회하는 방식:** XDP 또는 tc hook에서 eBPF 프로그램이 패킷을 직접 처리하여 Netfilter hook을 완전히 bypass한다. 이것이 이 스터디의 핵심 주제.

---

## 참고 자료

### 공식 문서 / 표준

| 자료 | 링크 |
|------|------|
| Linux Kernel Networking Documentation | https://www.kernel.org/doc/html/latest/networking/ |
| nf_conntrack sysctl | https://www.kernel.org/doc/html/latest/networking/nf_conntrack-sysctl.html |
| NAPI Documentation | https://docs.kernel.org/networking/napi.html |
| Netfilter Project | https://www.netfilter.org/documentation/ |
| Netfilter Hacking HOWTO | https://www.netfilter.org/documentation/HOWTO/netfilter-hacking-HOWTO.html |
| iptables HOWTO | https://www.netfilter.org/documentation/HOWTO/iptables-HOWTO.html |
| RFC 791 (IP) | https://www.rfc-editor.org/rfc/rfc791 |
| RFC 793 (TCP) | https://www.rfc-editor.org/rfc/rfc793 |
| RFC 1122 (Internet Host Requirements) | https://www.rfc-editor.org/rfc/rfc1122 |

### 커널 소스 (elixir.bootlin.com)

| 경로 | 내용 |
|------|------|
| `include/linux/skbuff.h` | sk_buff 구조체 정의 |
| `net/ipv4/ip_input.c` | ip_rcv(), ip_local_deliver() |
| `net/ipv4/ip_output.c` | ip_output(), ip_queue_xmit() |
| `net/netfilter/core.c` | nf_register_net_hook(), hook 실행 로직 |
| `net/netfilter/nf_conntrack_core.c` | nf_conntrack_in() |

### 서적

| 서적 | 저자 | 출판사 |
|------|------|--------|
| Understanding Linux Network Internals | Christian Benvenuti | O'Reilly (2006) |
| Linux Kernel Development, 3rd ed. | Robert Love | Addison-Wesley (2010) |
| The Linux Programming Interface | Michael Kerrisk | No Starch Press (2010) |

### 기술 블로그 / 아티클

| 자료 | 링크 |
|------|------|
| Cloudflare: Conntrack tales | https://blog.cloudflare.com/conntrack-tales-one-thousand-and-one-ways-to-break-your-linux-connectivity/ |
| LWN.net Networking articles | https://lwn.net/Kernel/Index/#Networking |
| Cilium Docs: eBPF Datapath | https://docs.cilium.io/en/stable/reference-guides/bpf/ |

---

## 실습 아이디어

```bash
# 1. conntrack 테이블 실시간 모니터링
sudo conntrack -E

# 2. iptables 규칙 전체 확인 (모든 table)
sudo iptables -L -v -n -t filter
sudo iptables -L -v -n -t nat
sudo iptables -L -v -n -t mangle
sudo iptables -L -v -n -t raw

# 3. 패킷 드롭 디버깅: LOG target 추가
sudo iptables -I INPUT -j LOG --log-prefix "[INPUT] " --log-level 4
sudo dmesg -w | grep "\[INPUT\]"

# 4. Kubernetes 환경에서 kube-proxy가 삽입한 iptables 규칙 확인
kubectl get svc
sudo iptables -L -t nat | grep <service-name>

# 5. conntrack 테이블 통계
sudo conntrack -S
```
