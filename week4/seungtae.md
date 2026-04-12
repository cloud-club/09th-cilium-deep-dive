# CILIUM

## 01. Why Cilium

### iptables 기반 CNI의 한계점

Kubernetes에서 Service를 만들면, kube-proxy가 각 노드 iptables 규칙을 생성.
이때, pod 개수, service 타입 등에 따라 iptables 규칙은 급격히 증가함.

**문제 1) O(n) 선형 탐색:** iptables는 규칙을 위에서 아래로 순차 탐색하므로, 서비스 수에 비례하여 지연시간이 증가함.

**문제 2) 규칙 업데이트 비효율:** Pod의 추가/삭제 시, kube-proxy는 전체 규칙 셋을 업데이트하며, service가 많아질수록 비효율성 증가함.

**문제 3) L3/L4 까지의 기능:** HTTP 규칙과 같은 L7 정책을 표현하지 못함.

### Cilium 등장 배경

|시점 | 발생 이벤트|
|--|---|
| 2016.08 | Linux Con에서 Cilium 처음 발표 |
| 2018.04 | Cilium 1.0 첫 릴리스 |
| 2020.08 | GKE Cilium 채택 |
| 2021.09 | AWS Cilium 채택 |
| 2023.10 | CNCF Graduated 프로젝트 승격 (CNI 최초) |

Cilium 프젝젝트를 만든 회사는 Isovalent 이며, 2023년 Cisco가 인수하며 엔터프라이즈 지원 체계도 강화됨.

### Calico vs. Cilium

| 비교 | Calico | Cilium |
| -- | --- | --- |
| Dataplane | iptables (최근 eBPF) | eBPF Native |
| Service Routing | kube-proxy (iptables) | BPF Map 해시 테이블 |
| 패킷 매칭 성능 | O(n) 선형 | O(1) 해시 룩업 |
| Network Policy | L3/L4 | L3/L4 + L7 |
| 정책 기반 | IP 기반 | Identity 기반 (라벨 -> 숫자 ID) |

* Calico 의 장점: 오래된 커널에서도 안정적 동작. Windows 컨테이너를 사용해야하는 환경일 경우.

* Identity 기반이란: Calico에서 Network Policy를 쓸 때, pod의 ip로 iptables 규칙이 생성됨. Pod 재생성 시, 해당 내용도 재 업데이트. Cilium에서는 pod의 라벨 조합을 숫자 identity로 매핑 후, 패킷에 identity 태깅. 즉, pod ip가 변경되어도 규칙을 다시 작성할 필요 없음.

### Calico + Istio vs. Cilium
Cilium이 대세가 된 주된 이유.

**Calico + Istio 조합:** Calico가 L3/L4 네트워킹과 Network Policy 담당, Istio의 Envoy Sidecar가 L7 트래픽 관리 (라우팅, retry, 서킷 브레이커) 담당.

1. **Resource Overhead:** Pod마다 Envoy Sidecar 가 붙어 메모리와 CPU 리소스 낭비.
2. **Latency:** 모든 트래픽이 Sidecar를 거치면서 지연 증가.
3. **운영 복잡성:** Calico와 Istio 별개 관리로, 운영 복잡도 증가.

**Cilium의 대체:** eBPF를 통해 L3/L4 네트워킹을 커널에서 처리하고, L7 정책의 경우 노드당 하나의 Envoy 프록시로 트래픽 리다이렉트.

| 비교 | Calico + Istio | Cilium |
| -- | --- | --- |
| Proxy | Pod 마다 사이트카 | 노드당 하나 |
| L7 | Istio | Cilium 내장 |
| 운영 주체 | 2개 독립 프로젝트 | 통합 |
| Observability | Jaeger 등 별도 설치 | Hubble 내장 |

***Cilium이 iptables 성능 한계를 eBPF로 해결하고, CNI + NetworkPolicy + ServiceMesh + Observability 를 하나의 통합 플랫폼으로 제공***

## 02. eBPF

### eBPF란
> 커널을 재컴파일하지 않고, 커널 안에서 내가 만든 프로그램을 실행하는 기술

기존 리눅스 커널 동작을 변경하고 싶을 경우
1. 커널 소스 수정 후 재컴파일: 완전한 제어 가능하지만, 시간/배포/버그 시 문제
2. 커널 모듈(LKM) 로드: 재컴파일은 불필요하지만, 버그 시 커널 crash 및 보안 검증 없음

eBPF는 커널이 특정 이벤트 (패킷 동작, 시스템 호출 등)이 발생할 때, 프로그램을 실행하며, 위험한 프로그램은 로드 자체를 거부

### BPF에서 eBPF
BPF(Berkely Packet Filter)는 1992년 기술로, tcpdump가 패킷을 필터링 시 사용하는 단순한 VM. 2014년에 패킷 필터링을 하는 BPF 기술에서 커널 모든 곳(cgroup, tracepoint 등 hook)에 붙을 수 있는 범용 프레임워크로 진화하여 eBPF로 확장.

### Cilium에서 BPF MAP 활용

**BPF MAP:** 커널의 eBPF 프로그램과 유저스페이스의 애플리케이션을 모두 읽고 쓸 수 있는 key-value 저장소

| MAP 용도 | KEY | VALUE |
| --- | -- | -- |
| 서비스 | Cluster IP + port | B/E Pod IP List |
| 정책 | Identity 번호 | 허용/차단 규칙 |
| Connection Tracking | 5-tuple (src/dst IP, port, proto) | 연결 상태 |

iptables를 순차 탐색하는 대신, eBPF 가 BPF MAP을 O(1)으로 DNAT 수행

## eBPF Hook
eBPF 프로그램은 hook 지점에 attach: 특정 이벤트 발생시 해당 프로그램 실행하라고 등록

[ Application ]

[ Socket Layer ] 
- cgroup hook/Socket hook (connect, sendmsg, recvmsg): 패킷이 네트워크 스택을 갔다 올 필요 없이 소켓 수준에서 목적지 변경 ==> 같은 노드 안에서 서비스 접근시, 패킷 만들어지기전에 소켓 단에서 백엔드 결정.

[ TCP/UDP IP Layer ]

[ TC (qdisc) ] 
- TC hook (ingress/egress): Traffic Control - 커널 네트워크 스택에 진입 후, ingress/egress 양쪽 모두 실행 가능. 패킷 헤더 수정, DNAT/SNAT 적용, 타 인터페이스 라다이렉션 가능 ==> Cilium에서 주된 hook.

[ NIC Driver ] 
- XDP hook: eXpress Data Path - NIC Driver Level에서 커널 네트워크 스택에 진입하기 전에 실행 (가장 빠른 hook)
패킷 통과, 드롭, 들어온 인터페이스로 되돌리거나, 타 인터페이스로 리다이렉션

[ Physical NIC ]

## Cilium의 Hook 선택
| Case | Hook | 이유 |
| -- | -- | -- |
| 같은 노드 Pod -> Service | Socket (connect) | 패킷 생성 전에 백엔드 결정 |
| 외부 -> NodePort Service | XDP/TC | 빠른 LB |
| Pod -> 외부 | TC egress | SNAT 처리 |

## 03. Cilium Architecture

<img width="800" height="700" alt="Image" src="https://github.com/user-attachments/assets/2cad9d74-3939-4c7d-8ea0-2b6b29c3c16e" />

- **Cilium Agent:** DaemonSet으로 모든 노드에 하나씩 실행되며, eBPF Lifecycle을 실행하는 주체 - Endpoint 생성, Identity 할당, eBPF 컴파일, BPF MAP 업데이트, N/W 설정
- **Cilium Operator:** IP Address Management, Identity 관리, CRD 처리 등 클러스터 Level의 관리 담당
- **Envoy Proxy:** L7 정책 관리하여 각 노드당 하나의 Envoy 할당
- **Hubble:** eBPF 기반 Monitoring system