# Week1. Linux Network 핵심 정리 (TCP/IP · Netfilter · iptables · conntrack)


## 1. TCP/IP Stack

### 1-1. 송신 경로

Application 레이어에서 `write()` 또는 `sendmsg()` 시스템콜을 호출하면 데이터는 Socket API를 통해 커널로 전달된다.  
TCP 레이어에서는 시퀀스 번호를 부여하고 MSS 단위로 데이터를 분할한 뒤 TCP 헤더를 추가한다.  
IP 레이어에서는 출발지/목적지 IP를 포함한 IP 헤더를 붙이고, 라우팅 테이블을 조회하여 어떤 네트워크 인터페이스로 보낼지 결정한다.  
Link 레이어에서는 ARP를 통해 다음 홉의 MAC 주소를 조회하고 Ethernet 헤더를 추가한 후 NIC 드라이버로 전달된다.

### 1-2. 수신 경로

NIC이 패킷을 수신하면 DMA를 통해 메모리에 복사되고 interrupt가 발생한다.  
커널은 `sk_buff` 구조체를 생성하고 Link → IP → TCP 순서로 헤더를 처리한다.  
이후 해당 소켓의 수신 버퍼에 데이터를 넣고, Application은 `read()` 또는 `recvmsg()`로 데이터를 가져간다.

### 1-3. sk_buff

`sk_buff`는 리눅스 커널에서 패킷 하나를 표현하는 핵심 자료구조이다.  
패킷이 각 레이어를 통과할 때 실제 데이터를 복사하지 않고 포인터만 이동시켜 헤더를 추가하거나 제거한다.

- head: 버퍼 시작
- data: 현재 payload 시작 위치
- tail: payload 끝
- end: 버퍼 끝

송신 시에는 data 포인터를 앞쪽으로 이동시키며 헤더를 추가하고, 수신 시에는 data 포인터를 뒤로 이동시키며 헤더를 제거한다.

### 1-4. Zero-copy (sendfile)

`sendfile()` 시스템콜을 사용하면 파일 데이터를 user space로 복사하지 않고 page cache에서 NIC로 직접 전달한다.  
이 과정에서 DMA를 활용하여 CPU 오버헤드를 줄인다.

### 1-5. 흐름 제어

- Receive Window: 수신 측 버퍼 여유 공간 기반 제어
- Congestion Window: 네트워크 혼잡도 기반 제어



## 2. Netfilter

### 2-1. 정의

Netfilter는 리눅스 커널 내부의 packet filtering framework이다.  
iptables, nftables, conntrack 등 대부분의 네트워크 기능이 이 위에서 동작한다.

핵심 개념은 Hook Point이며, 패킷이 특정 지점을 지날 때 callback 함수를 실행할 수 있다.

### 2-2. 5가지 Hook Point

- PREROUTING: 라우팅 결정 전 (DNAT 수행)
- INPUT: 로컬 프로세스로 들어오는 패킷
- FORWARD: 다른 인터페이스로 전달되는 패킷
- OUTPUT: 로컬에서 생성된 패킷
- POSTROUTING: 인터페이스로 나가기 직전 (SNAT 수행)

### 2-3. Hook Callback 처리

각 Hook에는 여러 callback이 등록될 수 있으며, 우선순위에 따라 실행된다.

- NF_ACCEPT: 패킷 계속 처리
- NF_DROP: 패킷 폐기
- NF_STOLEN: 패킷 소유권 획득
- NF_QUEUE: 유저 공간 전달



## 3. eBPF vs Netfilter

### 3-1. 처리 위치

패킷 처리 흐름:

NIC → XDP → tc → netfilter → TCP/IP stack → Application

- XDP: NIC 드라이버 레벨 (커널 진입 전)
- tc: 커널 진입 직후
- netfilter: 커널 네트워크 스택 내부

### 3-2. 차이점

eBPF 기반(XDP, tc)은 패킷이 커널 스택에 들어오기 전에 처리한다.  
iptables는 패킷이 이미 커널 네트워크 스택에 들어온 이후 처리한다.

이로 인해 eBPF는 처리 경로가 짧고 성능이 더 좋다.

### 3-3. Cilium

Cilium은 XDP와 tc hook을 활용하여 패킷을 초기에 처리한다.  
불필요한 트래픽을 커널 깊숙이 보내지 않기 때문에 CPU와 메모리 사용량을 줄일 수 있다.



## 4. iptables

### 4-1. 구조

iptables는 Netfilter 위에서 동작하는 rule-based 패킷 필터링 시스템이다.

구성:

- Table: 기능 단위
- Chain: 패킷 흐름 단위
- Rule: 조건과 동작 정의

### 4-2. 주요 테이블

- filter: 허용/차단
- nat: SNAT/DNAT
- mangle: 헤더 수정
- raw: conntrack 제외 처리

### 4-3. 패킷 매칭

패킷은 체인의 룰을 위에서 아래로 순차적으로 검사한다.  
매칭되는 룰이 발견되면 해당 타겟을 실행한다.

- ACCEPT, DROP
- DNAT, SNAT
- JUMP (다른 체인으로 이동)

### 4-4. Kubernetes에서 동작

kube-proxy는 Service 생성/변경 시 iptables 규칙을 자동 생성한다.

예시:

- ClusterIP: 10.96.0.1:80
- Pod 2개

동작 흐름:

PREROUTING → KUBE-SERVICES → KUBE-SVC → KUBE-SEP → Pod 선택 → DNAT

확률 기반으로 Pod를 선택하여 로드밸런싱을 수행한다.



## 5. iptables 성능 문제

### 5-1. 선형 탐색

모든 패킷은 룰을 처음부터 순차적으로 검사한다.  
시간 복잡도는 O(N)이며, Service 수 증가 시 성능 저하가 발생한다.

### 5-2. 규칙 업데이트 비용

iptables는 ruleset 전체를 교체하는 방식이다.  
Service 변경 시 lock이 발생하며, 대규모 환경에서는 지연이 발생한다.

### 5-3. 체인 점프 오버헤드

KUBE-SERVICES → KUBE-SVC → KUBE-SEP 구조로 인해 체인 점프가 많아질수록 오버헤드가 증가한다.

### 5-4. 대안 (Cilium)

eBPF hash map 기반으로 O(1) 조회가 가능하여 Service 수와 무관한 성능을 제공한다.



## 6. conntrack

### 6-1. 정의

conntrack은 커널이 네트워크 연결 상태를 추적하는 시스템이다.  
Netfilter의 일부이며 NAT 동작의 핵심 구성 요소이다.

### 6-2. 역할

DNAT/SNAT 수행 시 연결 정보를 저장하고, 응답 패킷을 원래 주소로 복원한다.

### 6-3. 테이블 구조

각 연결은 두 방향으로 저장된다.

- 원본 방향: client → service
- 응답 방향: pod → client

conntrack은 이를 기반으로 응답 시 주소를 복구한다.

### 6-4. 상태

- NEW
- ESTABLISHED
- RELATED
- INVALID

### 6-5. 문제점

1. 테이블 한계  
   `nf_conntrack_max` 초과 시 패킷 드롭 발생

2. 락 경합  
   다중 CPU 환경에서 동시 접근 시 성능 저하

3. UDP DNS 문제  
   동일 5-tuple 충돌로 race condition 발생

### 6-6. 해결 방법

- nf_conntrack_max 증가
- bucket 수 증가
- nodelocaldns 사용
- eBPF 기반 처리 (Cilium)



## 7. Kubernetes 전체 흐름

Client Pod에서 `curl http://my-service:80` 실행 시:

1. DNS 조회로 ClusterIP 획득
2. OUTPUT hook 통과
3. iptables에서 KUBE-SERVICES 체인 진입
4. 백엔드 Pod 선택 (확률 기반)
5. DNAT 수행 (ClusterIP → Pod IP)
6. conntrack에 연결 정보 저장
7. 패킷이 대상 Pod로 전달
8. Pod에서 응답 생성
9. conntrack이 응답 패킷의 주소를 ClusterIP로 복구
10. Client Pod에 전달

Client 입장에서는 처음부터 끝까지 Service와 통신한 것처럼 보인다.


## 8. 전체 핵심 정리

- TCP/IP 스택은 sk_buff 기반으로 동작하며 포인터 이동으로 효율적인 처리 수행
- Netfilter는 커널 내 패킷 처리 프레임워크이며 hook 기반 구조
- iptables는 rule 기반 엔진이지만 O(N) 구조로 인해 성능 한계 존재
- conntrack은 NAT 상태를 추적하여 응답 패킷 복구 수행
- Cilium은 eBPF(XDP/tc)를 활용하여 초기 단계에서 패킷을 처리하고 성능을 개선