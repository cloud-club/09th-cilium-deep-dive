# 1. 리눅스 네트워크 Datapath 이해 패킷 흐름
<img width="1122" height="732" alt="image" src="https://github.com/user-attachments/assets/94a940af-3eea-4b59-b95c-3c22adc22ae7" />

```bash
패킷 흐름: NIC → XDP → TC ingress → routing → TC egress → NIC
```

외부에서 들어온 패킷은 먼저 NIC 드라이버에 도착합니다. 이때 가장 먼저 실행되는 것이 XDP입니다. XDP는 커널 네트워크 스택에 들어가기 전에 동작하며, 매우 빠른 속도로 패킷을 DROP, PASS, REDIRECT 할 수 있습니다. 여기서 PASS된 패킷만 커널 내부로 진입합니다.

커널에 들어오면 skb 구조체로 변환되고, 이후 TC ingress 단계가 실행됩니다. 이 단계에서는 QoS나 필터링, eBPF 로직을 적용할 수 있습니다. 그 다음은 Netfilter PREROUTING으로, 여기서 DNAT과 conntrack이 시작됩니다. 이후 커널은 라우팅 결정을 수행하여 패킷의 목적지를 판단합니다.

목적지가 로컬이면 INPUT 체인을 거쳐 사용자 공간의 프로세스(예: nginx, sshd)로 전달됩니다. 반대로 다른 호스트로 전달해야 한다면 FORWARD 체인을 통과합니다.

패킷이 외부로 나가기 전에는 POSTROUTING 단계에서 SNAT이 수행됩니다. 이후 TC egress에서 마지막 트래픽 제어가 적용되고, 다시 NIC를 통해 외부로 전송됩니다.

정리하면, 패킷 흐름은
NIC → XDP → TC ingress → PREROUTING → (INPUT 또는 FORWARD) → POSTROUTING → TC egress → NIC 순서입니다.
핵심은 XDP가 가장 앞단에서 성능 최적화를 담당하고, TC와 Netfilter가 그 뒤에서 정책과 NAT 처리를 수행한다는 점입니다.

# 2. eBPF 기본 구조

hooking: eBPF는 네트워크 트래픽을 가로챈다.

프로그램 작성 및 로딩

1. eBPF 프로그램 작성: 특정 훅 포인트에 실행될 로직을 작성
2. eBPF 프로그램 검증: eBPF 검증기를 통과하면 커널에 로딩됨. 검증기는 무한루프에 빠지거나 커널 메모리에 잘못된 접근을 시도하는 위험한 동작을 검사함.
3. eBPF 프로그램 커널 로딩: 검증을 통과한 프로그램이 안전하게 로딩됨

# TC hook
## TC?
리눅스 커널에서 네트워크 트래픽을 제어하는 기능 / 큐로 대역폭 정의

TC는 커널 네트워크 경로 중간에 들어가는 제어 계층으로, skb기반으로 동작함. eBPF를 TC에 붙일때도 ingress/egress qdisc에 classifier/action 형태로 붙습니다.

### 기능

1. 패킷 차단
2. 속도 제한
3. 우선순위 제어
4. 트래픽 분류

### 구조:

- qdisc: 트래픽 제어
- filter: 패킷 조건 검사(패킷을 어떻게 분류하느냐)
    - matchall : 모든 패킷 매칭
        
        들어오고 나가는 패킷을 100% 매칭, 조건 검사 없음
        
    - flower: L2/L3/L4 헤더 기반 매칭
        
        패킷 → 헤더 파싱 → 조건 비교 → match 여부 결정
        
        특정 IP, 포트 차단, 간단 정책 기반
        
        ```bash
        tc filter add dev enp0s1 ingress protocol ip flower \
          src_ip 1.1.1.1 dst_port 80 action drop
        ```
        
    - u32: 저수준 bit 기반 필터
        
        패킷의 특정 바이트를 직접 비교하는 저수준 필터
        
        ```bash
        tc filter add dev enp0s1 protocol ip parent ffff: \
          u32 match ip src 1.1.1.1 action drop
        ```
        
- action: 실제 동작


### TC에 eBPF를 붙이는 방법

1. qdisc 먼저 생성
    
    ```bash
    sudo tc qdisc add dev enp0s1 clsact
    ```
    
2. eBPF attach
    
    ```bash
    sudo tc filter add dev enp0s1 ingress bpf da obj tc_drop.o sec classifier
    ```

의미

```bash
enp0s1 인터페이스
 → ingress
   → bpf 프로그램 실행
     → TC_ACT_SHOT → drop
```

## 실습

```bash
### 설치
sudo apt install -y clang llvm libelf-dev iproute2 linux-headers-$(uname -r)

clang -O2 -target bpf -g -I/usr/include/aarch64-linux-gnu -c tc_counter.c -o tc_counter.o
sudo tc qdisc add dev enp0s1 clsact
sudo tc filter add dev enp0s1 ingress bpf obj tc_counter.o sec tc direct-action
sudo bpftool map dump id <ID>
```

```c
// tc_drop_icmp.c
#include <linux/bpf.h>
#include <linux/pkt_cls.h>
#include <linux/ip.h>
#include <bpf/bpf_helpers.h>

SEC("tc")
int drop_icmp(struct __sk_buff *skb) {
    void *data = (void *)(long)skb->data;
    void *data_end = (void *)(long)skb->data_end;

    struct iphdr *ip = data + sizeof(struct ethhdr); // IP헤더 위치 계산
    if ((void *)(ip + 1) > data_end) //검증과정
        return TC_ACT_OK;

    if (ip->protocol == IPPROTO_ICMP) {
        bpf_printk("Dropping ICMP packet\n");
        return TC_ACT_SHOT;  // 드롭
    }
    return TC_ACT_OK;
}

char _license[] SEC("license") = "GPL";
```

ICMP 패킷을 드롭하는 TC eBPF 프로그램
