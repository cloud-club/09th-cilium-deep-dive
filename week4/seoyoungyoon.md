# 1. eBPF의 배경

쿠버네티스 클러스터가 커질수록 네트워크 처리 성능이 병목이 됩니다. 전통적인 방식인 **iptables**는 규칙을 선형으로 탐색하기 때문에, Pod 수가 늘어날수록 규칙도 함께 늘어납니다. Pod 10,000개 클러스터에서는 수만 개의 iptables 규칙이 생성되고, 패킷 하나를 처리할 때마다 이 규칙들을 순서대로 검사합니다.

### **iptables 방식**

- 규칙 수 = Pod 수에 비례
- 선형 탐색 → O(n) 성능
- 규칙 변경 시 전체 재로드
- 커널 네트워크 스택 전체 통과 후 필터링
- 가시성(Observability) 제한적

### **eBPF (Cilium) 방식**

- eBPF Map → O(1) 해시 조회
- Pod 수 증가해도 성능 일정
- 규칙 변경 시 Map만 업데이트
- 소켓 레벨에서 바로 처리
- Hubble로 전체 트래픽 가시성 확보

# 2. Hook 지점
패킷이 커널을 통과하는 여러 지점에 eBPF 프로그램을 붙여 datapath를 제어하는 메커니즘입니다.

### **패킷 흐름과 Hook 지점**

```bash
애플리케이션
 ↓
소켓 (Socket) ← sock_ops / sk_msg Hook # L4 로드밸런싱, 소켓 정책 적용
 ↓
TCP/IP 레이어
 ↓
TC (Traffic Control) ← tc ingress/egress Hook # NetworkPolicy 적용, 암호화, NAT
 ↓
XDP (드라이버 레벨) ← XDP Hook # DDoS 방어, 초고속 패킷 드롭
 ↓
NIC (네트워크 카드)
```

### Hooks

**1. TC Hook (tc ingress/egress)**
- Cilium의 핵심 Hook입니다.
- 커널 네트워크 스택 중간에서 ingress/egress 양쪽에 hook이 존재합니다.
- NetworkPolicy 적용, 암호화, SNAT/DNAT 등 대부분의 네트워킹 기능이 여기서 처리됩니다.

**2. 소켓 레벨 Hook (sock_ops)** 
- 애플리케이션이 사용하는 소켓 레벨에서 동작하는 hook입니다.
- connect(), send(), recv() 같은 소켓 이벤트 시점에 동작합니다.
- Service IP로 요청하면 → 실제 Pod IP로 연결을 바로 매핑 (kube-proxy replacement의 핵심)
- TCP 연결이 맺어지는 시점에 개입해 목적지를 수정함으로써, 패킷이 네트워크 스택을 완전히 통과하기 전에 로드밸런싱을 처리합니다.

**3.XDP (eXpress Data Path)**
- NIC 드라이버 바로 아래에서 실행되는 가장 빠른 eBPF hook입니다.
- 커널 네트워크 스택에 진입하기 전에 패킷을 drop/redirect 처리합니다.

> 🤔 하나만 실행되는 구조인가?
>
> 여러 hook이 순차적으로 실행될 수 있다.
> 같은 패킷이 모든 hook을 다 거치는것은 아님
> 중간에 drop되는 이후 hook은 실행되지 않는다.

# **3. kube-proxy replacement**

쿠버네티스에서 Service는 여러 Pod를 하나의 가상 IP(ClusterIP)로 묶어주는 추상화 레이어입니다. 전통적으로 **kube-proxy**가 이 역할을 담당했습니다. kube-proxy는 Service IP를 실제 Pod IP로 변환하기 위해 iptables 또는 IPVS 규칙을 관리합니다.

### **기존 kube-proxy 방식**

- client Pod에서 Service로 요청할 때

```
client Pod
  → Service IP (10.96.0.1:80)          # 가상 IP로 요청
  → iptables DNAT 규칙 탐색            # 수천 개 규칙 순서대로 확인
  → Pod IP (10.244.1.254:80)로 변환    # 실제 목적지로 NAT
  → 네트워크 스택 전체 통과
```

### **Cilium kube-proxy replacement**

- Cilium이 활성화된 경우

```
client Pod
  → Service IP (10.96.0.1:80)          # 가상 IP로 요청
  → sock_ops Hook 실행                 # 소켓 연결 시점에 개입
  → eBPF Map 조회 (O(1) 해시)          # 즉시 목적지 확인
  → Pod IP로 직접 연결                 # NAT 없이 바로 통신
```

# **4. Cilium Datapath 동작 원리**

Cilium의 datapath는 패킷이 실제로 처리되는 경로입니다. 같은 노드 내 Pod 간 통신과 다른 노드 간 통신의 경로가 다릅니다.

### **같은 노드 내 Pod 간 통신 (same-node)**

```
Pod A (10.244.1.9)
  → veth pair (가상 이더넷)
  → TC egress Hook    ← Cilium이 여기서 정책 확인
  → 리눅스 브리지
  → TC ingress Hook   ← 수신 측에서 한 번 더 확인
  → veth pair
  → Pod B (10.244.1.254)
```

### **다른 노드 간 Pod 통신 (cross-node)**

```
Pod A (Node 1)
  → TC egress Hook
  → VXLAN 또는 Geneve 터널 캡슐화    ← overlay 모드
  → 물리 NIC
  → (네트워크)
  → 물리 NIC (Node 2)
  → XDP 또는 TC ingress Hook
  → 터널 디캡슐화
  → Pod B (Node 2)
```
# 실습1. eBPF 알아보기
```bpftrace```: eBPF를 이용해서 커널/애플리케이션 **동작을 실시간으로 관찰하는 tracing 도구** 입니다.
eBPF가 커널에서 어떻게 동작하는지 실습
```bash
bpftrace 문법구조, 명령어 구조

probe이름 /필터/ { 액션 }

probe: 무엇을 추적할지
필터: 언제 추적할지
액션: 다음 조건을 만족할때 무엇을 할지 정의

interval:s:1        { printf("hi\n"); }
↑                     ↑
언제 실행할지          뭘 할지
```
```bash
# 1. 'Helo eBPF' 출력
sudo bpftrace -e 'interval:s:1 { printf("Hello eBPF!\n"); }'

# 2. 시스템에서 실행 중인 모든 execve (프로그램 실행) 감지
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s → %s\n", comm, str(args->filename)); }'
```
다른 창에서 `ls`, `ps` 실행시 실시간으로 결과가 출력됨
<img width="2048" height="239" alt="image" src="https://github.com/user-attachments/assets/947b3de3-b305-409b-b5ff-63aff7605db2" />

```bash
# 도메인 프로세스 실시간 확인(open 시스템 콜)
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_openat {
    printf("PID:%-6d %-16s %s\n", pid, comm, str(args->filename));
}'

# google.com 입력
curl google.com

curl → /etc/passwd        # 사용자 정보 확인
curl → /home/ubuntu/.curlrc  # curl 설정파일
curl → /etc/resolv.conf   # DNS 서버 주소
curl → /etc/hosts         # 로컬 DNS
curl → /etc/gai.conf      # IPv4/IPv6 우선순위 설정
# DNS 조회전에 커널에서 많은 파일을 열어보는것을 확인할 수 있음
```

> ❓ irqbalance 중간중간 뜨는 이유?
> 
> 인터럽트(IRQ)를 CPU코어에 균등하게 분배하는 리눅스 데몬
> 
> ```bash
> 하드웨어 이벤트 (네트워크 패킷, 디스크 I/O 등)
>         ↓
>    인터럽트(IRQ) 발생
>         ↓
>  irqbalance가 어느 CPU 코어가 한가한지 보고
>         ↓
>   적절한 코어에 인터럽트 배분
> ```

### 명령어를 입력해보자
```bash
# 슬로우 파일 읽기 감지 / @start[tid]: eBPF map 저장소에 입력
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }

tracepoint:syscalls:sys_exit_read
/@start[tid]/
{
    $lat = nsecs - @start[tid];
    if ($lat > 1000000) {
        printf("SLOW READ! PID:%-6d %-16s latency:%dms\n", 
               pid, comm, $lat / 1000000);
    }
    delete(@start[tid]);
}'

# latency 히스토그램 분석
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }

tracepoint:syscalls:sys_exit_read
/@start[tid]/
{
    @lat_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'

Attaching 2 probes... # 컴파일 중!!! 왜 하는지는 아래 설명
# eBPF는 커널의 헤더 정보를 읽어옴. 

@lat_us:
[0]                   44 |@@@                                                 |
[1]                   15 |@                                                   |
[2, 4)                 1 |                                                    |
[4, 8)                12 |                                                    |
[8, 16)              643 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@|
[16, 32)             595 |@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@@    |
[32, 64)             311 |@@@@@@@@@@@@@@@@@@@@@@@@@                           |
[64, 128)             81 |@@@@@@                                              |
[128, 256)            11 |                                                    |
[256, 512)             4 |                                                    |
[512, 1K)              1 |                                                    |
[1K, 2K)              82 |@@@@@@                                              |
[2K, 4K)              14 |@                                                   |
[4K, 8K)               4 |                                                    |

'
[8, 16)   643개  ← 8~16μs 사이에 read가 643번 발생 (가장 많음)
[16, 32)  595개  ← 16~32μs
[32, 64)  311개
정상적인 상태) 대부분 8~32에 끝남 1ms 넘는건 극소수임
'

sudo bpftrace -e 'tracepoint:raw_syscalls:sys_enter { @[comm] = count(); }'
Attaching 1 probe...
# 약 1초

@[systemd-journal]: 5
@[cron]: 6
@[sshd]: 9
@[sudo]: 16
@[multipathd]: 43
@[bpftrace]: 68
@[irqbalance]: 100
'각프로세스 이름: 시스템콜 호출 횟수값'
```

> 🤔 컴파일을 하는 이유 알아보기
>
> eBPF는 **커널 헤더**의 정보를 읽어와 현재 실행중인 커널에 맞춰서 즉석에서 eBPF프로그램을 컴파일한다.
> 컴파일러는 tack_struct와 같은 커널 내부 데이터 구조체의 정확한 정의를 알아야하기 때문에 실행중인 커널과 버전이 일치하는 헤더가 필요함. 
>
> 현재 Ubuntu 22.04는 커널 5.15 BTF(BPF Type Format)가 기본으로 내장되어있음
>
> 예전에는 bpftrace가 커널헤더파일을 읽어서 구조체를 파악했다면
bpftrace → /sys/kernel/btf/vmlinux 읽어서 구조체 파악

```bash
# 패킷 드롭 감지 TCP연결 추적
sudo bpftrace -e '
kprobe:tcp_connect { 
    $sk = (struct sock *)arg0;
    $dport = ($sk->__sk_common.skc_dport >> 8) | 
             (($sk->__sk_common.skc_dport & 0xff) << 8);
    printf("PID:%-6d %-16s → %s:%d\n",
        pid, comm,
        ntop($sk->__sk_common.skc_daddr),
        $dport);
}'
# 명령어 cilium이 Pod에서 나가는 TCP 연결을 정확히 잡음
 
# 입력
curl https://google.com
curl https://github.com

# 결과
curl → 142.250.197.142:443   ← google.com IP, HTTPS(443)
curl → 20.200.245.247:443    ← github.com IP, HTTPS(443)
DNS 조회가 끝나고 다음에 TCP를 연결하는 시점에 eBPF에 잡음
```

```bash
curl → DNS 조회 (github.com → 20.200.245.247)
              ↓
     tcp_connect 호출  ← eBPF가 여기서 잡음
              ↓
   Cilium: NetworkPolicy 확인
              ↓
     허용 → 연결 진행 ✅
     차단 → 패킷 드롭 ❌
```
<img width="1380" height="1136" alt="image" src="https://github.com/user-attachments/assets/851aad42-0b26-4528-b4d8-bc1409e9d805" />


---
[BPF Architecture — Cilium 1.20.0-dev documentation](https://docs.cilium.io/en/latest/reference-guides/bpf/architecture/)

[Installation Using Kind — Cilium 1.19.2 documentation](https://docs.cilium.io/en/stable/installation/kind/)
