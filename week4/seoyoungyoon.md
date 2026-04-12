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

**TC Hook (tc ingress/egress)** 은 Cilium의 핵심 Hook입니다. 네트워크 인터페이스에서 패킷이 들어오고 나갈 때 실행되며, NetworkPolicy 적용, 암호화, SNAT/DNAT 등 대부분의 네트워킹 기능이 여기서 처리됩니다.

**소켓 레벨 Hook (sock_ops)** 은 kube-proxy replacement의 핵심입니다. TCP 연결이 맺어지는 시점에 개입해 목적지를 수정함으로써, 패킷이 네트워크 스택을 완전히 통과하기 전에 로드밸런싱을 처리합니다. 이것이 iptables NAT보다 효율적인 이유입니다.

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
# **실습1. eBPF 알아보기 **
```bash
bpftrace 문법구조

probe이름 /필터/ { 액션 }

probe: 무엇을 추적할지
필터: 언제 추적할지
액션: 다음 조건을 만족할때 무엇을 할지 정의

interval:s:1        { printf("hi\n"); }
↑                     ↑
언제 실행할지          뭘 할지

sudo bpftrace -e 'interval:s:1 { printf("Hello eBPF!\n"); }'

# 시스템에서 실행 중인 모든 execve (프로그램 실행) 감지
sudo bpftrace -e 'tracepoint:syscalls:sys_enter_execve { printf("%s → %s\n", comm, str(args->filename)); }'
```
<img width="2048" height="239" alt="image" src="https://github.com/user-attachments/assets/947b3de3-b305-409b-b5ff-63aff7605db2" />

```bash
# 어떤 프로세스가 어떤 파일을 여는지 실시간 확인(open 시스템 콜)
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_openat {
    printf("PID:%-6d %-16s %s\n", pid, comm, str(args->filename));
}'

curl google.com

curl → /etc/passwd        # 사용자 정보 확인
curl → /home/ubuntu/.curlrc  # curl 설정파일
curl → /etc/resolv.conf   # DNS 서버 주소
curl → /etc/hosts         # 로컬 DNS
curl → /etc/gai.conf      # IPv4/IPv6 우선순위 설정
DNS 조회전에 많은 파일을 열어보는것을 볼 수 있음
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
# 슬로우 파일 읽기 감지
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

@start[tid]: eBPF map 저장소

# latency 히스토그램 분석
sudo bpftrace -e '
tracepoint:syscalls:sys_enter_read { @start[tid] = nsecs; }

tracepoint:syscalls:sys_exit_read
/@start[tid]/
{
    @lat_us = hist((nsecs - @start[tid]) / 1000);
    delete(@start[tid]);
}'

Attaching 2 probes... # 컴파일 중!!!

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

# **실습2. NetworkPolicy로 직접 확인하기**

아래 실습은 Ubuntu 22.04 (커널 5.15), kind 기반 Cilium 1.17.2 환경에서 진행했습니다.

### **환경 구성**

```
# Pod 생성
kubectl run app --image=nginx --labels="app=web"
kubectl run client --image=curlimages/curl --command -- sleep 9999

# Pod IP 확인
kubectl get pods -o wide
# app: 10.244.1.254 / client: 10.244.1.9
```

### **Step 1 — 기본 통신 확인**

```
kubectl exec client -- curl -s --connect-timeout 3 http://10.244.1.254
```

결과

```
<!DOCTYPE html><html>...Welcome to nginx!...
```

NetworkPolicy가 없으면 같은 네임스페이스의 Pod 간 통신은 기본적으로 허용됩니다.

### **Step 2 — deny-all 정책 적용**

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
```

Cilium Agent가 이 정책을 감지하면 해당 Pod의 TC Hook에 차단 규칙을 eBPF Map으로 적재합니다. iptables 규칙은 생성되지 않습니다.

결과

```
command terminated with exit code 28 (타임아웃)
```

### **Step 3 — 특정 Pod만 허용**

```
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-client-only
spec:
  podSelector:
    matchLabels:
      app: web
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          run: client
```

결과

```
Welcome to nginx! → client Pod에서만 접근 허용
```

**Cilium이기 때문에 가능한 것**

NetworkPolicy는 쿠버네티스 표준 스펙입니다. 하지만 Cilium은 이를 eBPF로 구현하기 때문에 **iptables 규칙 없이 소켓 레벨에서 처리**합니다. Pod가 수천 개로 늘어나도 O(1) 해시 조회로 일정한 성능을 유지합니다.


---
[BPF Architecture — Cilium 1.20.0-dev documentation](https://docs.cilium.io/en/latest/reference-guides/bpf/architecture/)

[Installation Using Kind — Cilium 1.19.2 documentation](https://docs.cilium.io/en/stable/installation/kind/)
