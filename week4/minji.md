# Cilium TC Egress 흐름 분석


### eBPF

커널 코드를 수정하지 않고 커널 안에 프로그램을 주입하는 기술.  
Cilium은 여러 hook 지점에 eBPF 프로그램을 붙인다.

### Hook 지점

패킷이 NIC로 들어오면 커널 안을 통과한다. 그 경로 중간중간에 코드 실행을 위해 지정해두는 지점이다.

```
패킷 흐름
NIC 수신
  └── XDP hook          ← 가장 빠름, 드라이버 레벨
        └── TC hook     ← Traffic Control, ingress/egress
              └── socket hook  ← L7 정책, 소켓 레벨
```

| hook | 위치 | 용도 |
|---|---|---|
| XDP | 드라이버 | 빠른 drop/redirect (DDoS 차단) |
| TC ingress/egress | 네트워크 스택 | 패킷 필터링, NAT, LB (복잡한 로직 처리) |
| kprobe/tracepoint | 커널 함수 | 관찰/모니터링 |
| socket | 소켓 레이어 | L7 정책, 암호화 |

### kube-proxy replacement

기존 kube-proxy는 iptables로 Service → Pod IP 변환을 했다.  
규칙이 많아지면 선형 탐색이라 느려지므로 Cilium은 이걸 eBPF로 완전히 대체한다.

```
기존 kube-proxy: ClusterIP → iptables 규칙 수천 개 → Pod IP  (O(n))
Cilium:          ClusterIP → eBPF map (해시테이블)  → Pod IP  (O(1))
```

### eBPF Map

Cilium이 빠른 핵심 이유. 커널-유저스페이스 공유 해시테이블이다.

| map | 역할 |
|---|---|
| `cilium_lb4_services_v2` | Service → Backend 매핑 (어떤 pod들로 보낼지) |
| `cilium_lb4_backends_v3` | Backend Pod IP/Port 목록 (실제 Pod IP/Port 목록) |
| `cilium_policy_v2` | 네트워크 정책 |
| `cilium_ipcache` | IP → security identity |


## 개요

한 번의 Pod egress 경로 안에서 일어나는 3가지 핵심 과정

1. Service map 조회
2. ClusterIP → Pod IP DNAT
3. DNAT된 실제 목적지 기준 policy 체크

### 예시 패킷
```
src: Pod A (10.0.0.5:12345)  →  dst: ClusterIP (10.96.0.10:80)
                                  ↓ DNAT
                              dst: Pod B (10.0.1.23:8080)
```



## 전체 흐름

```
## /bpf/bpf_lxc.c
cil_from_container()
  └── tail call → CILIUM_CALL_IPV4_FROM_LXC
        └── tail_handle_ipv4()
              └── __per_packet_lb_svc_xlate_4()
                    ├── lb4_lookup_service()        ← ClusterIP → Service (O(1))
                    └── lb4_local()
                          ├── ct_lazy_lookup4()     ← CT 확인 (새 연결? 기존?)
                          ├── lb4_select_backend_id() ← backend 선택
                          ├── lb4_lookup_backend()  ← Pod IP 조회 (O(1))
                          └── ct_create4()          ← CT 기록
                    └── lb4_dnat_request()
                          └── lb4_xlate()           ← 패킷 헤더 rewrite
        └── tail call → CILIUM_CALL_IPV4_CT_EGRESS
              └── handle_ipv4_from_lxc()
                    ├── lookup_ip4_remote_endpoint() ← Pod B identity 조회
                    └── policy_can_egress4()         ← policy map 조회 (O(1))
```



## 단계별 상세

### 1. TC egress hook 진입 — `cil_from_container()`

Pod A에서 패킷이 나가는 순간, veth pair의 TC hook이 패킷을 잡아서 `cil_from_container()`로 들어온다.  
ethertype을 확인하고 프로토콜별로 tail call 분기한다.

> **tail call이란?**  
> 일반 함수 호출은 스택이 쌓이는데, tail call은 현재 BPF 프로그램을 버리고 새 프로그램으로 점프한다.  
> BPF 스택 제한(512바이트)을 우회하기 위해 쓴다. ctx(패킷 컨텍스트)는 그대로 전달된다.

```c
// bpf_lxc.c
__section_entry
int cil_from_container(struct __ctx_buff *ctx) {
    __be16 proto = 0;            // 패킷이 IPv4/IPv6/ARP 중 무엇인지
    __u32 sec_label = SECLABEL;  // 이 Pod의 security label
    __s8 ext_err = 0;

    bool valid_ethertype = validate_ethertype(ctx, &proto);
    bpf_clear_meta(ctx);           // 이전 메타데이터 초기화
    check_and_store_ip_trace_id(ctx);

    switch (proto) {
    case bpf_htons(ETH_P_IP):     // IPv4면
        edt_set_aggregate(ctx, LXC_ID);  // QoS 태깅
        ret = tail_call_internal(ctx, CILIUM_CALL_IPV4_FROM_LXC, &ext_err);
        sec_label = SECLABEL_IPV4;
        break;
    }
}
```



### 2. LB 활성화 확인 후 Service lookup 진입 — `tail_handle_ipv4()`

tail call로 받아서 per-packet LB가 켜져 있으면 서비스 변환 경로로 분기한다.

> **`ENABLE_PER_PACKET_LB`**  
> Cilium이 kube-proxy를 완전히 대체할 때 켜지는 플래그.  
> 이게 켜지면 eBPF가 Service NAT을 직접 처리한다.  
> `#ifdef`는 런타임 분기가 아니라 **컴파일 타임**에 어떤 코드를 포함할지 결정한다. 매 패킷마다 조건 체크가 없어서 더 빠르다.

```c
// bpf_lxc.c
static __always_inline int tail_handle_ipv4(struct __ctx_buff *ctx, ...) {
    // IP 헤더 파싱 (BPF는 패킷 메모리 접근 시 반드시 범위 체크 필요)
    if (!revalidate_data(ctx, &data, &data_end, &ip4))
        return DROP_INVALID;

#ifdef ENABLE_PER_PACKET_LB
    // per-packet LB 활성화 → ClusterIP를 실제 Pod IP로 변환
    return __per_packet_lb_svc_xlate_4(ctx, ip4, ext_err);
#else
    // 비활성화 → Service 변환 건너뛰고 바로 CT 처리
    return tail_ipv4_ct_egress(ctx);
#endif
}
```



### 3. 5-tuple key 구성 + Service map 조회 — `__per_packet_lb_svc_xlate_4()`

패킷에서 5-tuple을 뽑아서 LB key를 만들고, 그 key로 Service map을 조회한다.

> **5-tuple이란?**  
> 네트워크에서 패킷 flow를 유일하게 식별하는 5가지 정보
> ```
> src IP   : 10.0.0.5    (보내는 쪽 IP)
> dst IP   : 10.96.0.10  (받는 쪽 IP, 지금은 ClusterIP)
> src port : 12345       (보내는 쪽 포트)
> dst port : 80          (받는 쪽 포트)
> protocol : TCP
> ```

```c
// bpf_lxc.c
static __always_inline int
__per_packet_lb_svc_xlate_4(struct __ctx_buff *ctx, struct iphdr *ip4, ...) {

    struct ipv4_ct_tuple tuple = {};
    struct lb4_key key = {};

    // 패킷에서 5-tuple 파싱
    ret = lb4_extract_tuple(ctx, ip4, ETH_HLEN, &l4_off, &tuple);
    // tuple = {saddr:10.0.0.5, daddr:10.96.0.10, sport:12345, dport:80, TCP}

    // tuple → lb4_key 변환 (Service 찾는 데 src 정보는 필요 없으므로 dst만)
    lb4_fill_key(&key, &tuple);
    // key = {address:10.96.0.10, dport:80, proto:TCP}

    // ★ Service map 조회 (O(1))
    svc = lb4_lookup_service(&key, is_defined(ENABLE_NODEPORT));
    if (!svc) {
        goto skip_service_lookup;  // Service 없으면 일반 패킷으로 통과
    }

    // Service 찾았으면 backend 선택으로
    ret = lb4_local(get_ct_map4(&tuple), ctx, fraginfo,
                    l4_off, &key, &tuple, svc, &ct_state_new,
                    &backend, ext_err);
}
```

**`lb4_lookup_service()` 내부:**

```c
// lb.h
__lb4_lookup_service(struct lb4_key *key) {
    return map_lookup_elem(&cilium_lb4_services_v2, key);
    // key   = {10.96.0.10, 80, TCP}
    // value = {count:3, rev_nat_index:5, flags:...}
    //          └── count:3 → backend Pod이 3개
}
```



### 4. Backend 선택 + CT 상태 관리 — `lb4_local()`

Service를 찾았으니 **어떤 backend Pod으로 보낼지** 결정한다.  
단순히 고르는 게 아니라 CT(conntrack)와 함께 관리해서 **같은 연결은 항상 같은 Pod**으로 가게 한다.

> **CT(conntrack)를 쓰는 이유**  
> TCP 연결은 여러 패킷으로 이루어진다.  
> 첫 패킷에서 Pod B를 선택했으면 그 연결의 나머지 패킷들도 Pod B로 가야 한다.  
> CT가 없으면 패킷마다 다른 backend로 갈 수 있어서 TCP가 깨진다.

```c
// lb.h
lb4_local(...) {

    state->rev_nat_index = svc->rev_nat_index;
    // reply 패킷 역방향 NAT할 때 쓸 인덱스 저장

    // CT map에서 이 flow가 기존에 본 적 있는지 확인
    ret = ct_lazy_lookup4(map, tuple, ctx, ..., CT_SERVICE, SCOPE_REVERSE, ...);

    switch (ret) {
    case CT_NEW:  // 처음 보는 연결
        // 1) session affinity 있으면 같은 client → 같은 backend
        if (lb4_svc_is_affinity(svc)) {
            backend_id = lb4_affinity_backend_id_by_addr(svc, &client_id);
        }

        // 2) affinity 없거나 못 찾으면 새로 선택 (random or maglev hash)
        if (backend_id == 0) {
            backend_id = lb4_select_backend_id(ctx, key, tuple, svc);
            backend = lb4_lookup_backend(ctx, backend_id);
            // cilium_lb4_backends_v3[backend_id] → {10.0.1.23, 8080}
        }

        // 3) CT에 기록 → 다음 패킷은 CT_ESTABLISHED로 재사용
        state->backend_id = backend_id;
        ct_create4(map, NULL, tuple, ctx, CT_SERVICE, state, ext_err);
        break;

    case CT_ESTABLISHED:  // 기존 연결 → CT에 저장된 backend 재사용
    case CT_RELATED:
        backend_id = state->backend_id;
        backend = lb4_lookup_backend(ctx, backend_id);
        break;
    }
}
```

**map 두 단계:**

```
1단계: Service map에서 Service 찾기
  key   = {10.96.0.10, 80, TCP}   ← 패킷의 목적지 정보
  value = {count:3, rev_nat_index:5}
              └── count:3 이면 backend Pod이 3개 있다는 뜻

2단계: backend map에서 실제 Pod IP 꺼내기
  key   = backend_id(7)            ← 1단계에서 선택된 backend id; id로 backend map을 한 번 더 조회해서 실제 Pod IP 구함
  value = {address:10.0.1.23, port:8080}  ← 실제 Pod IP/port
```



### 5. DNAT — `lb4_dnat_request()` + `lb4_xlate()`

backend가 정해졌으니 실제 패킷 헤더를 바꾼다.  
tuple을 먼저 수정하고, 그 다음 실제 패킷에 반영하는 두 단계로 나뉜다.
바로 패킷을 건드리는 게 아니라 tuple 먼저 수정하고 그 다음 실제 패킷에 반영한다.

**`lb4_dnat_request()` — tuple 수정:**

```c
// lb.h
static __always_inline int
lb4_dnat_request(struct __ctx_buff *ctx,
                 const struct lb4_backend *backend, ...,
                 struct ipv4_ct_tuple *tuple, bool loopback) {

    __be32 saddr = tuple->saddr;   // 10.0.0.5 (Pod A)
    __be32 daddr = tuple->daddr;   // 10.96.0.10 (ClusterIP) - 바뀔 값
    __be16 dport = tuple->sport;   // 80 
    
    // loopback이 아니면 목적지를 backend로 tuple 변경
    if (!loopback)
        tuple->daddr = backend->address;  // 10.96.0.10 → 10.0.1.23

    // port도 backend port로
    if (likely(backend->port))
        tuple->sport = backend->port;     // 80 → 8080

    // 실제 패킷 헤더 + checksum 수정
    return lb4_xlate(ctx, &new_saddr, &saddr,
                     tuple->nexthdr, l3_off, l4_off,
                     daddr, dport, backend,
                     ipfrag_has_l4_header(fraginfo));
}
```

**`lb4_xlate()` — 실제 패킷 바이트 수정:**

```c
// IP 헤더 dst 필드 직접 수정
ctx_store_bytes(ctx, l3_off + offsetof(struct iphdr, daddr),
                &backend->address, 4, 0);

// checksum incremental update (전체 재계산 아님, 바뀐 필드만 반영)
csum_replace4(&csum, old_daddr, backend->address);

// TCP dst port 수정
ctx_store_bytes(ctx, l4_off + offsetof(struct tcphdr, dest),
                &backend->port, 2, 0);
```

**패킷 변화:**

```
before: [IP src=10.0.0.5, dst=10.96.0.10] [TCP sport=12345, dport=80]
after:  [IP src=10.0.0.5, dst=10.0.1.23]  [TCP sport=12345, dport=8080]
```



### 6. CT egress tail call + identity 조회 — `handle_ipv4_from_lxc()`

DNAT 끝나면 상태 저장하고 CT egress tail call로 넘어간다.  
CT egress 처리 후 `handle_ipv4_from_lxc()`에서 목적지 Pod의 identity를 조회한다.

> **identity란?**  
> Cilium에서 모든 Pod은 security identity라는 숫자 ID를 가진다.  
> IP는 Pod 재시작 시 바뀔 수 있지만, identity는 라벨 기반이라 안 바뀐다.  
> ClusterIP는 가상 IP라 identity가 없으므로, DNAT된 실제 Pod IP 기준으로 조회한다.

```c
// bpf_lxc.c

// 1. DNAT 결과 저장 후 tail call
lb4_ctx_store_state(ctx, &ct_state_new, ...);
tail_call_internal(ctx, CILIUM_CALL_IPV4_CT_EGRESS, ...);

// 2. identity 조회
info = lookup_ip4_remote_endpoint(ip4->daddr, cluster_id);

// 3. 조회 결과 처리
if (info) {
    *dst_sec_identity = info->sec_identity; 
    skip_tunnel = (info->flag_skip_tunnel) || same_subnet_id;
} else {
    *dst_sec_identity = WORLD_IPV4_ID;
}
```
1. DNAT까지 끝났으니 그 결과(어떤 backend 골랐는지 등)를 ctx에 저장하고 다음 프로그램으로 점프한다. ctx에 저장하는 이유는 tail call 넘어가도 ctx는 유지되기 때문이다.
2. ip4->daddr는 이미 DNAT로 바뀐 10.0.1.23 (Pod B IP)이다. 이 IP로 cilium_ipcache에서 "이 IP가 어떤 Pod인지" 조회한다.
cilium_ipcache: 10.0.1.23 → {sec_identity: 1234}
3. sec_identity: 다음 단계 policy 체크에서 "Pod B에 접근 허용인지" 판단할 때 씀
<br> skip_tunnel: Pod B가 같은 노드나 서브넷에 있으면 VXLAN으로 감쌀 필요 없으니까 터널 스킵
<br> ipcache에 없다 = Cilium이 모르는 IP = 클러스터 외부 트래픽으로 취급.



### 7. Policy 체크 — `policy_can_egress4()`

Pod A → Pod B 패킷이 **정책상 허용되는지** policy map에서 검사한다.

```c
// bpf_lxc.c
verdict = policy_can_egress4(ctx, tuple, l4_off,
                              SECLABEL_IPV4,      // Pod A identity
                              *dst_sec_identity,  // Pod B identity (1234)
                              ...);
```

```c
// policy.h
__policy_can_access(map, ..., local_id, remote_id, dport, proto, ...) {

    struct policy_key key = {
        .sec_label = remote_id,  // Pod B identity (1234)
        .protocol  = proto,      // TCP
        .dport     = dport,      // 8080
    };

    // 1차: L3+L4 정확한 매치
    // "Pod A(identity:xxx) → Pod B(identity:1234) TCP:8080 허용?"
    policy = map_lookup_elem(map, &key);

    // 2차: L4-only wildcard
    // identity 조건 없애고 "누구든 → TCP:8080 허용?"
    key.sec_label = 0;
    l4policy = map_lookup_elem(map, &key);

    // 더 구체적인 정책(precedence 높은 것) 적용
    // 둘 다 없으면 기본 deny
    return DROP_POLICY;
}
```

**결과에 따른 처리:**

| verdict | 동작 |
|---|---|
| `ALLOW` | 패킷 통과 |
| `DENY` | 패킷 버림 (DROP_POLICY) |
| `PROXY_REDIRECT` | Envoy로 넘김 (HTTP 헤더 같은 L7 정책 처리) |
| `AUTH_REQUIRED` | mutual TLS 등 인증 필요 |

> `PROXY_REDIRECT`는 "GET /admin 은 막아" 같은 HTTP 레벨 정책은 BPF에서 볼 수 없어서  
> Envoy(L7 프록시)로 넘겨서 처리하게 한다.
