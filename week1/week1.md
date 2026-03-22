## Week1. Linux Network 기본 | TCP/IP Stack · Netfilter · iptables · conntrack

### ch1. TCP/IP Stack

1-1. 계층 구조

linux 커널의 네트워크 스택은 application layer부터 물리 드라이버까지 여러 계층으로 나뉜다.
<br>Applicaion레이어에서 write()/sendmsg() 시스템콜을 호출하면 데이터는 소켓 api를 통해 커널로 내려간다.
<br>커널은 TCP 레이어에서 seq 번호를 붙이고 mss 단위로 data를 분할한 후 tcp 헤더를 추가한다.
<br>이후 IP 레이어에서 출발지/목적기 IP주소를 담은 IP헤더를 추가하고, routing-table을 조회해 어떤 network-interface로 패킷을 보낼지 결정한다.
<br>마지막으로 link 레이어에서 ARP를 통해 다음 hop의 mac주소를 조회하고 ethernet 헤더를 붙여 NIC 드라이버로 전달한다.

수신 경로는 반대로 동작한다. NIC이 패킷을 받으면 DMA를 통해 메모리를 복사하고 interrupt가 발생한다.
<br>커널은 sk_buff 구조체를 생성해 link layer부터 IP, TCP 순서로 헤더를 처리하고 최종적으로 해당 소켓의 수신 버퍼에 데이터를 넣는다.
<br>application은 read() 또는 recmsg()로 데이터를 가져간다.

1-2. sk_buff

: linux 커널에서 패킷 하나를 표현하는 핵심 자료구조이다. 
<br>패킷이 각 레이어를 통과할 때 실제 data를 복사하지 않고 pointer만 조작해 헤더를 추가, 제거한다. head: 버퍼의 시작, data: 현재 payload의 시작, tail: payload의 끝, end: 버퍼의 끝이다. 송신 시 헤더를 추가할 때는 data 포인터를 앞으로 당기고, 수신 시 헤더를 제거할 때는 data 포인터를 뒤로 민다.

1-3. 대용량 파일 전송 시 커널 동작

: application이 sendfile() 시스템콜을 통해 커널은 zero-copy 경로를 탄다. 파일 내용을 user space로 복사하지 않고 page-cache에서 NIC DMA로 전달해서 cpu 오버헤드가 줄어든다.
<br>흐름제어는 첫째, 수신 측이 자신의 버퍼 여유 공간을 receive-window로 광고하고 송신 측은 이 값을 넘지 않도록 전송하거나 congestion window를 통해 네트워크 혼잡도에 따라 전송 속도를 조절한다. 

### ch2. netfilter

2-1. netfilter란?

:linux 커널 내에 내장된 **packet-filtering framework**이다. iptables, nftables, conntrack 등 linux의 네트워크 보안 기능 대부분이 이 위에서 동작한다. 
<br>핵심 개념은 **hook point**인데 패킷이 kernel 네트워크 스택을 지나는 특정 시점에 callback 함수를 등록해두면 해당 지점을 통과하는 모든 패킷에 대해 그 함수가 호출된다. 이를 통해 커널 소스를 수정하지 않고도 packet-filtering, 주소 변환, logging 등의 기능을 추가할 수 있다.

2-2. 5가지 hook-point

패킷이 네트워크 인터페이스로 들어오면 가장 먼저 **prerouting hook**을 통과한다. 라우팅 결정이 내려지기 전이라 목적이 IP를 바꾸는 DNAT이 여기서 적용된다. 또한 k8s에서 service clusterIP로 향하는 패킷이 podIP로 변환되는 것도 이 시점이다.
<br>라우팅 결정 후 패킷이 local-process로 향한다고 판단되면 **input hook**을 통과하고 다른 interface로 포워딩되어야 한다면 **forward hook** 을 통과한다.
<br>로컬 프로세스가 패킷을 생성해 내보낼 때는 **output hook**을 통과하고 최종적으로 interface로 나가기 직전에 **postrouting hook**을 통과한다. SNAT와 MASQUEREADE가 여기서 적용된다.

2-3. hook callback 처리

각 hook point에는 여러 모듈이 동시에 callback을 등록할 수 있고 우선순위 숫자가 낮을수록 먼저 실행된다. 각 callback 함수는 패킷을 처리한 뒤 반환값으로 커널에 다음 동작을 알린다. 
<br>NF_ACCEPT를 반환하면 패킷이 계속 처리되고 NF_DROP을 반환하면 패킷이 즉시 폐기된다. NF_STOLEN은 콜백이 패킷 소유권을 가져간다는 의미이며 NF_QUEUE는 패킷을 유저스페이스로 전달한다.

2-4. netfilter와 eBPF의 위치 관계

eBPF 기반 도구들은 netfilter보다 더 앞단의 hook을 사용한다. XDP는 NIC 드라이버 레벨에서 동작하며 packet이 kernel 네트워크 스택에 진입하기도 전에 처리된다. 
<br>traffic control hook은 XDP보다 뒤지만 netfilter보다 앞에서 동작한다. 
<br>cilium은 이 xdp와 tc-hook을 주로 사용해서 netfilter 기반의 iptables/kube-proxy보다 패킷 처리 경로가 짧고 성능이 좋다.
<br>[네트워크 카드(NIC)] -> [XDP (eBPF)] -> [tc (eBPF)] -> [netfilter (iptables)]: 커널 네트워크 스택 안, 패킷을 처리한 상태 -> [커널 TCP/IP 스택] -> [애플리케이션]
<br>iptables: 패킷 → 커널 안으로 들어옴 → 검사 → 처리 - 비용이 많이 든다 / cilium: 패킷 → 초입에서 검사 → 필요 없으면 바로 버림 - cpu/mem 절약

### ch3. iptables

3-1. 구조

iptables는 netfilter hook 위에 구축된 rule-based 패킷 필터링 도구이다.
<br>내부는 테이블, 체인, 룰 3계층으로 구성된다. 
<br>테이블은 기능별로 나뉘는데 **filter** 테이블은 패킷 허용/차단을 담당하고 **nat 테이블**은 IP 주소 변환(SNAT/DNAT)을 담당한다. **mangle 테이블**은 TTL이나 TOS 같은 패킷 헤더 값을 수정할 때 쓰고 **raw 테이블**은 conntrack 처리에서 특정 패킷을 제외할 때 사용한다. 
<br>각 테이블 안에는 체인이 있고, 체인 안에 실제 매칭 조건과 동작을 정의한 룰들이 순서대로 나열된다.

3-2. 패킷 매칭 방식

패킷이 iptables를 통과할 때 체인의 룰을 위에서 아래로 순서대로 비교한다. 
<br>매칭되는 룰이 있으면 해당 룰의 타겟(ACCEPT, DROP, DNAT, JUMP 등)을 실행한다. JUMP 타겟은 다른 체인으로 이동하는데 k8s에서는 KUBE-SERVICES → KUBE-SVC-XXX → KUBE-SEP-XXX 순서로 체인을 점프하며 최종 목적지 Pod를 결정한다.

3-3. k8s에서 iptables 동작

kube-proxy는 Service 오브젝트가 생성되거나 변경될 때마다 iptables 규칙을 자동으로 생성하고 관리한다.
<br>예를 들어 ClusterIP가 10.96.0.1:80이고 백엔드 Pod가 2개인 Service가 있다면 PREROUTING에서 10.96.0.1:80으로 향하는 패킷을 KUBE-SERVICES 체인으로 보내고 <br>거기서 확률 기반으로 두 백엔드 Pod 중 하나를 선택해 DNAT(목적지 ip 바꿈)를 적용한다. 응답 패킷은 conntrack이 자동으로 원래 Service IP로 되돌려준다. (client가 요청한 것과 응답한 ip값이 다르므로)


3-4. iptables는 왜 느린가

iptables의 가장 큰 성능 문제는 **선형 탐색**이다. 모든 패킷은 체인의 룰을 처음부터 하나씩 순서대로 비교하기 때문에 시간 복잡도가 O(N)이다. 
<br>노드 하나에 Service가 1000개 있으면 iptables 룰이 수천 개에서 수만 개까지 늘어나고 모든 패킷마다 이 룰들을 처음부터 탐색해야 한다. 
<br>두 번째 문제는 **규칙 업데이트 비용**이다. iptables는 원자적 교체 방식을 사용하기 때문에 Service 하나가 추가되거나 삭제될 때마다 전체 규칙 셋을 lock 걸고 새 규칙 셋으로 교체한다. Service가 수만 개인 대규모 클러스터에서는 이 교체 작업 자체가 수 초씩 걸릴 수 있고 그 동안 패킷 처리가 지연된다. 
<br>세 번째 문제는 체인 **점프 오버헤드**다. KUBE-SERVICES → KUBE-SVC-XXX → KUBE-SEP-XXX처럼 체인이 중첩될수록 커널 내 점프 횟수가 늘어나고 처리 오버헤드가 커진다. <br>Cilium이 이를 eBPF hash-map로 대체하면 목적지 IP:Port를 키로 O(1) 조회가 가능해 Service 수와 무관하게 일정한 성능을 낸다.

3-5. 주요 명령어
```
# 현재 nat 테이블 규칙 조회
iptables -t nat -L -n -v --line-numbers

# K8s Service 규칙 추적
iptables -t nat -L KUBE-SERVICES -n -v
iptables -t nat -L KUBE-SVC-XXXXXX -n -v

# 전체 규칙 수 확인
iptables-save | wc -l

# 규칙 저장 및 복원
iptables-save > rules.txt
iptables-restore < rules.txt
```

### ch4. conntrack

4-1. conntrack이란?

: linux 커널이 모든 네트워크 연결의 상태를 추적하는 메커니즘이다. Netfilter의 일부로 동작하며 DNAT/SNAT가 동작하기 위한 핵심 인프라다. 
<br>iptables가 패킷의 IP 주소를 바꾸더라도 응답 패킷이 돌아올 때 원래 주소로 자동으로 되돌릴 수 있는 것은 conntrack이 연결 정보를 기억하고 있기 때문이다. 
<br>k8s에서 클라이언트가 Service ClusterIP로 요청을 보내면 DNAT로 Pod IP로 변환되는데 응답 패킷에서 Pod IP를 다시 ClusterIP로 바꾸는 것도 conntrack이 한다.

4.2 conntrack 테이블 구조

conntrack은 각 연결에 대해 원본 방향과 응답 방향 두 가지 정보를 함께 저장한다. 
<br>원본 방향은 클라이언트가 보낸 패킷의 출발지/목적지 IP:Port이고 응답 방향은 서버가 보내는 응답 패킷의 출발지/목적지 IP:Port다. 
<br>DNAT가 적용된 경우 응답 방향의 출발지 IP가 실제 Pod IP이지만 커널은 conntrack 테이블을 보고 이를 자동으로 Service ClusterIP로 바꿔서 클라이언트에 전달한다. 
<br>연결 상태는 NEW, ESTABLISHED, RELATED, INVALID 네 가지로 분류되며 각 상태마다 타임아웃 값이 다르게 설정되어 있다.

4.3 conntrack 테이블 한계

conntrack 테이블의 최대 엔트리 수는 nf_conntrack_max 커널 파라미터로 제한된다. 기본값은 메모리에 따라 다르지만 보통 65536 정도인데 Pod 수가 많은 K8s 클러스터에서는 이 한계에 쉽게 도달한다. 
<br>테이블이 가득 차면 nf_conntrack: table full, dropping packet 에러와 함께 패킷이 드롭된다. 또 다른 문제는 **고트래픽 환경에서의 락 경합**이다. 
<br>여러 CPU가 동시에 conntrack 해시 테이블에 접근하면 락 경합이 발생해 성능이 저하된다. k8s 환경에서는 nf_conntrack_max를 늘리고 버킷 수도 함께 늘려야 해시 충돌을 줄일 수 있다.

4.4 K8s의 conntrack 관련 이슈 — UDP DNS 문제

k8s에서 자주 발생하는 conntrack 관련 버그 중 하나가 DNS 타임아웃 문제다. 
<br>여러 Pod가 동시에 DNS 쿼리를 보낼 때 UDP 특성상 동일한 5-tuple(출발지IP:Port, 목적지IP:Port, 프로토콜)이 겹칠 수 있고 이 과정에서 conntrack 레이스 컨디션이 발생해 패킷이 드롭된다. 
<br>해결책으로는 각 노드에 DNS 캐시를 두는 **nodelocaldns**를 배포하거나 Cilium을 사용해 eBPF 기반 conntrack으로 이 문제를 우회하는 방법이 있다.

4.5 주요 명령어
```
# conntrack 테이블 조회
conntrack -L

# 특정 IP 관련 연결 조회
conntrack -L --src 10.244.0.5

# 실시간 이벤트 모니터링
conntrack -E

# CPU별 통계 조회
conntrack -S

# 현재 사용량 확인
cat /proc/sys/net/netfilter/nf_conntrack_count
cat /proc/sys/net/netfilter/nf_conntrack_max

# 튜닝
sysctl -w net.netfilter.nf_conntrack_max=1048576
sysctl -w net.netfilter.nf_conntrack_buckets=262144
```

### 전체 요약

client Pod에서 curl http://my-service:80을 실행하면 커널은 먼저 DNS로 my-service의 ClusterIP를 조회한다. 
<br>client 파드의 IP 레이어에서 목적지 ClusterIP(예: 10.96.0.100)로 라우팅 결정이 내려지고 sk_buff가 생성된다. 
<br>패킷은 Netfilter OUTPUT Hook을 통과하면서 iptables의 nat 테이블에서 KUBE-SERVICES 체인을 거쳐 해당 ClusterIP:Port에 매칭되는 백엔드 Pod 하나를 랜덤 선택하고 DNAT로 IP를 바꾼다. 
<br>이 시점에 conntrack에 연결 정보가 기록된다. 변환된 패킷은 라우팅을 거쳐 veth pair를 통해 대상 Pod의 네트워크 네임스페이스로 전달된다. 
<br>응답이 돌아올 때 conntrack이 Pod IP를 다시 ClusterIP로 되돌려주고 클라이언트 Pod에 도달한다. 클라이언트 입장에서는 처음부터 끝까지 ClusterIP와 통신한 것처럼 보인다.

TCP/IP 스택은 패킷을 sk_buff 구조체로 표현하며 각 레이어에서 포인터 조작만으로 헤더를 추가하거나 제거한다. 
<br>Netfilter는 커널 네트워크 스택의 5개 Hook Point에 콜백을 등록할 수 있는 프레임워크이며 iptables와 conntrack이 그 위에서 동작한다.
<br>iptables는 Netfilter를 활용한 규칙 엔진이지만 선형 탐색(O(N))과 원자적 규칙 교체 방식 때문에 Service 수가 많아지면 성능 병목이 발생한다. 
<br>conntrack은 DNAT/SNAT 상태를 추적해 응답 패킷이 올바른 주소로 돌아가게 하지만 테이블 풀과 락 경합 문제가 고트래픽 K8s 환경에서 문제가 된다. 
<br>Cilium은 이 모든 것을 eBPF로 대체해 XDP/TC 레벨의 O(1) 해시 조회와 커널 내 conntrack 우회를 통해 성능을 끌어올린다.