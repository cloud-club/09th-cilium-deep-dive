# [학습 리포트] 쿠버네티스 CNI 연결 구조와 IP 할당의 실체

## Part 1. 기본 학습 정리

## 1. 개요: 쿠버네티스 소스 코드에 CNI가 없는 이유
쿠버네티스 메인 레포지토리(`kubernetes/kubernetes`)의 `pkg/kubelet/network`에서 CNI 관련 코드가 보이지 않는 것은 **설계의 분리** 때문입니다.

* **과거 (v1.23 이전):** Kubelet이 `dockershim`을 통해 직접 CNI 바이너리를 실행하는 코드가 포함되어 있었습니다.
* **현재 (v1.24 이후):** Kubelet은 네트워크 설정을 직접 하지 않습니다. 대신 **CRI(Container Runtime Interface)**라는 표준 규격(gRPC)을 통해 컨테이너 런타임(containerd 등)에 네트워크 준비를 위임합니다.

기존 내용을 바탕으로, Protobuf의 동작 원리와 실제 데이터가 어떻게 실려 나가는지에 대한 기술적 깊이를 더해 보강했습니다.

## 2. Kubelet의 네트워크 인지: gRPC를 통한 추상화

쿠버네티스가 파드에 IP를 할당하라고 명령을 내리는 "추상화된 코드"는 **CRI(Container Runtime Interface)**라고 불리는 gRPC 메시지 규격 안에 존재합니다. 쿠버네티스 소스 코드에서 CNI 바이너리를 직접 호출하는 로직이 보이지 않는 이유는, Kubelet이 이 규격에 맞춰 "요청"만 던지기 때문입니다.

### **핵심 정의 (IDL: Interface Definition Language)**
* **경로:** `[staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto](https://github.com/kubernetes/kubernetes/blob/master/staging/src/k8s.io/cri-api/pkg/apis/runtime/v1/api.proto)`

```proto
// RunPodSandbox는 파드 수준의 샌드박스를 생성하고 시작합니다.
// 런타임은 성공 시 샌드박스가 준비 상태(Ready)임을 보장해야 합니다.
rpc RunPodSandbox(RunPodSandboxRequest) returns (RunPodSandboxResponse) {}

message RunPodSandboxRequest {
    // 샌드박스 생성을 위한 상세 설정 보따리
    PodSandboxConfig config = 1; 
    
    // RuntimeClass 등에 따른 런타임 핸들러 지정 (예: runc, kata)
    string runtime_handler = 2;
}
```

### **기술적 포인트: 숫자의 의미와 데이터의 실체**
코드에 적힌 `= 1`, `= 2`와 같은 숫자는 단순한 값이 아니라 **필드 번호(Field Number)**입니다. 

* **번호표의 역할:** gRPC는 데이터를 보낼 때 "이름(config)" 대신 "번호(1번)"를 사용해 이진(Binary) 데이터로 압축합니다. 이를 통해 텍스트 기반의 JSON보다 훨씬 빠르고 작게 데이터를 주고받습니다.
* **재귀적 구조:** `config = 1`은 단순한 숫자가 아니라, 아래와 같은 거대한 `PodSandboxConfig` 메시지 객체를 포함하고 있습니다.



### **실제 네트워크 설정이 담기는 경로**
`PodSandboxConfig` 메시지를 타고 더 깊이 들어가면, 드디어 네트워크와 관련된 구체적인 추상화 필드들이 등장합니다.

```proto
message PodSandboxConfig {
    PodSandboxMetadata metadata = 1;
    // ...
    DNSConfig dns_config = 4;
    
    // 실제 네트워크 인터페이스 설정을 유도하는 핵심 필드
    PodSandboxNetworkConfig network = 5; 
    
    LinuxPodSandboxConfig linux = 6;
}

message PodSandboxNetworkConfig {
    // Kubelet이 노드 할당 시 결정한 PodCIDR 정보 등이 담길 수 있음
    string pod_cidr = 1;
}
```

### **동작 방식 및 추상화 수준**
1.  **명령의 위임:** Kubelet은 `RunPodSandboxRequest` 객체에 파드 이름, 네임스페이스, DNS 설정, 그리고 네트워크 모드 등을 채워 넣습니다.
2.  **직렬화(Serialization):** 이 객체는 Protobuf에 의해 이진 데이터로 압축되어 gRPC 통신을 통해 컨테이너 런타임(containerd 등)으로 전송됩니다.
3.  **추상화의 핵심:** Kubelet 코드는 **"172.16.0.5 IP를 할당해"**라고 명령하지 않습니다. 대신 **"이 `network` 설정에 맞춰서 네트워킹이 가능한 샌드박스를 하나 뽑아줘"**라고만 말합니다. 
4.  **결과 수신:** 런타임이 외부 CNI(Cilium 등)를 실행해 IP를 받아오면, 그 결과값(IP 주소 등)은 `RunPodSandboxResponse`가 아니라 이후 `PodSandboxStatus` gRPC 호출을 통해 Kubelet에 보고됩니다.

## 3. CNI 실행의 주체: 컨테이너 런타임(Runtime)
실제로 호스트 노드에 깔린 CNI 바이너리를 실행하는 것은 **containerd**나 **CRI-O** 같은 런타임입니다.

1.  **gRPC 수신:** 런타임이 Kubelet으로부터 `RunPodSandbox` 요청을 받습니다.
2.  **설정 로드:** 런타임은 호스트의 `/etc/cni/net.d/` 디렉토리에서 JSON 설정 파일을 읽습니다.
3.  **바이너리 호출:** 설정에 적힌 CNI 플러그인(예: Cilium, Calico)을 호스트의 `/opt/cni/bin/`에서 찾아 직접 실행(fork/exec)합니다.


## 4. Cilium과 같은 외부 구현체의 설치 및 연결
Cilium이 노드에 직접 바이너리를 설치하는 것과 쿠버네티스의 연결 고리는 다음과 같습니다.

| 구성 요소 | 위치 | 역할 |
| :--- | :--- | :--- |
| **Cilium CNI Plugin** | `/opt/cni/bin/cilium-cni` | 컨테이너 런타임에 의해 실행되는 짧은 수명의 바이너리. |
| **Cilium Agent** | 노드 내 프로세스 (데몬셋) | 실제 eBPF 프로그램을 커널에 로드하고 IPAM(IP 관리)을 수행하는 '뇌'. |

**연결 흐름:**
`Runtime` → `cilium-cni (바이너리)` → `Cilium Agent (IPC 통신)` → `IP 할당 및 커널 설정`


## 5. IPAM(IP Address Management): IP는 누가 결정하는가?
쿠버네티스 메인 코드에는 IP 할당 로직이 없지만, 노드 단위의 대역 설정 로직은 존재합니다.

* **NodeIPAM Controller (`pkg/controller/nodeipam/`):** 클러스터 전체에서 각 노드가 사용할 큰 IP 대역(`PodCIDR`)을 할당합니다.
* **CNI IPAM:** CNI 플러그인은 노드에 할당된 `PodCIDR` 범위 내에서 개별 파드에 부여할 구체적인 IP 주소를 결정합니다.
* **응답(Response):** 결정된 IP 주소는 다시 gRPC 응답을 타고 Kubelet으로 전달되어 `Pod.status.podIP`에 기록됩니다.

## 요약 및 결론
쿠버네티스 소스 코드에서 CNI 연결 코드를 찾는다면, 바이너리 실행 코드가 아닌 **CRI gRPC 인터페이스 정의**와 **NodeIPAM 컨트롤러**를 봐야 합니다. 실제 네트워크 구현은 쿠버네티스 외부(CNI 바이너리 및 에이전트)에서 이루어지며, 이는 쿠버네티스가 다양한 네트워크 솔루션을 유연하게 수용할 수 있게 해주는 핵심 설계입니다.

---
## Part 2. 질문 1

### 질문
Service가 10,000개인 클러스터에서 iptables 모드를 사용할 때, 특정 서비스의 IP가 변경되면 노드 전체의 iptables 규칙을 어떻게 갱신하는가? 이때 발생하는 Latency 스파이크의 원인은 무엇인가?

### 학습 정리 1: 배경
#### 1. 전사(Pre-story): `iptables`의 한계
쿠버네티스 초기에 서비스(Service)를 구현하기 위해 사용한 도구는 리눅스 커널의 **iptables**였습니다. 

* **동작 방식:** 패킷이 들어오면 정의된 규칙(Rule)들을 위에서부터 아래로 하나씩 대조하며 "이 서비스 IP는 이 파드 IP로 보내라"는 명령을 수행합니다.
* **문제점:** 규칙이 수천 개, 수만 개로 늘어나면 패킷 하나가 통과하기 위해 수만 번의 `if-then` 검사를 거쳐야 합니다($O(n)$ 성능). 이는 대규모 클러스터에서 심각한 **네트워크 지연(Latency)**과 **CPU 부하**를 유발했습니다.

#### 2. IPVS의 등장: "검색 말고 해시 테이블"
이 문제를 해결하기 위해 도입된 것이 바로 **IPVS (IP Virtual Server)**입니다. IPVS는 리눅스 커널 내부에 구축된 고성능 L4 로드밸런서입니다.

* **동작 방식:** 규칙을 리스트가 아닌 **해시 테이블(Hash Table)** 구조로 관리합니다. 패킷이 들어오면 서비스 IP를 키(Key)로 사용하여 목적지를 즉시 찾아냅니다($O(1)$ 성능).
* **장점:** 서비스가 10개든 10,000개든 패킷 처리 속도가 일정합니다. 또한 RR(Round Robin), LC(Least Connection) 등 다양한 로드밸런싱 알고리즘을 지원하죠.


#### 3. 그런데 왜 Cilium은 IPVS를 버리고 eBPF로 갔을까?
IPVS가 $O(1)$ 성능으로 `iptables`의 검색 문제를 해결했는데도 불구하고, Cilium이 eBPF로 데이터 패스를 새로 짠 이유는 **"IPVS 역시 낡은 커널 프레임워크의 부품"**이었기 때문입니다.

#### **A. "길목"의 문제 (Netfilter 병목)**
IPVS는 독립적으로 작동하는 게 아니라, 여전히 리눅스의 **Netfilter**라는 거대한 네트워크 스택 위에서 동작합니다.
* 패킷이 들어오면 커널의 복잡한 표준 네트워크 처리 과정을 다 거치고 나서야 "아, 이건 IPVS 규칙이네?" 하고 처리가 됩니다. 
* 반면, **Cilium(eBPF)**은 네트워크 드라이버 바로 위(XDP)나 최하단(`tc`)에서 패킷을 낚아챕니다. 커널의 무거운 표준 로직을 타기 전에 **"지름길"**로 보내버리는 것이죠.

#### **B. 제어부(Control Plane)의 비효율**
질문하셨던 **Control Plane 병목**의 핵심입니다.
* `kube-proxy`가 IPVS 모드일 때, 새로운 파드가 생기면 IPVS 규칙뿐만 아니라 이를 보조하는 `iptables`(마스커레이딩용)와 `ipset` 규칙들도 같이 갱신해야 합니다. 이 과정은 **전체 동기화(Full Sync)** 방식에 가까워 서비스가 많아질수록 업데이트 속도가 느려집니다.
* **eBPF**는 공유 메모리인 **Map**을 씁니다. 전체를 갈아엎을 필요 없이, 그냥 특정 지도(Map)의 데이터 한 줄만 살짝 바꾸면 즉시 반영됩니다.

#### **C. "지능"의 부재**
IPVS는 단순한 로드밸런서입니다. 패킷이 어디서 왔는지(Identity), 어떤 보안 정책을 적용해야 하는지 알지 못합니다. 
* **Cilium**은 eBPF 프로그램을 통해 **"로드밸런싱 + 보안 정책 + 가시성(Logging)"**을 하나의 로직 안에서 한꺼번에 처리합니다. IPVS를 쓰면 "로드밸런싱은 IPVS가, 보안은 다시 iptables가" 하는 식으로 패킷이 여기저기 들러야 하지만, eBPF는 한 곳에서 다 끝냅니다.


#### 4. 요약: 진화의 방향성

| 단계 | 핵심 기술 | 복잡도 | 비유 |
| :--- | :--- | :--- | :--- |
| **1단계** | `iptables` | $O(n)$ | 모든 문을 일일이 열어보며 길 찾기 |
| **2단계** | `IPVS` | $O(1)$ | 인덱스(색인)를 보고 길 찾기 (하지만 여전히 건물 안 복도를 다 지나가야 함) |
| **3단계** | `eBPF (Cilium)` | $O(1) + \alpha$ | **건물 입구에서 목적지로 바로 텔레포트하기** |

### 학습 정리 2: 질문 직접 답변

#### 1. `iptables` 규칙의 갱신 방식: "전부 아니면 전무(All or Nothing)"

리눅스 커널의 `iptables`는 특정 규칙 하나만 골라서 '수정'하는 기능이 매우 제한적입니다. `kube-proxy`가 서비스 변경 사항을 반영하는 과정은 다음과 같습니다.

1.  **전체 복사 (Read):** `kube-proxy`는 현재 커널에 적용된 모든 `iptables` 규칙을 메모리로 읽어옵니다.
2.  **메모리 내 수정 (Update):** 메모리상에서 변경된 서비스의 IP나 엔드포인트 정보를 수정하고, 나머지 수만 개의 규칙과 합쳐서 **새로운 전체 규칙 리스트**를 만듭니다.
3.  **전체 덮어쓰기 (Write):** `iptables-restore` 명령어를 사용하여 수정된 **전체 리스트를 커널에 통째로 다시 붓습니다.**


#### 2. Latency 스파이크의 원인 (3가지 핵심 이유)

서비스가 10,000개일 때, 규칙 하나가 바뀌었다고 수만 줄의 규칙을 다시 적용하는 과정에서 다음과 같은 병목이 발생합니다.

#### **① 커널 락(Kernel Lock) 및 순차 처리**
`iptables`를 업데이트하는 동안 커널은 규칙 테이블에 **Lock**을 겁니다. 규칙이 10,000개라면 이 리스트를 커널 메모리에 쓰고 검증하는 데 시간이 꽤 걸립니다. 이 "짧은 찰나" 동안 노드를 통과하는 패킷들은 업데이트가 끝날 때까지 대기하거나 처리가 지연되어 **Latency 스파이크**가 발생합니다.

#### **② $O(n)$ 탐색 시간의 누적**
패킷이 들어오면 `iptables` 체인을 위에서부터 아래로 하나씩 대조합니다. 
* 서비스가 10,000개라면, 운 나쁘게 리스트 제일 아래에 있는 서비스로 향하는 패킷은 **10,000번의 비교 연산**을 거쳐야 합니다. 
* 업데이트 직후 커널 스택이 새로 구성되면서 캐시 효율이 떨어지는 순간, 이 탐색 시간은 더욱 길어집니다.

#### **③ CPU 점유율 폭발 (Control Plane Overhead)**
`kube-proxy`는 서비스 변경을 감지할 때마다 수만 줄의 텍스트(iptables-save 형태)를 생성하고 파싱해야 합니다. 
* 서비스가 빈번하게 생성/삭제되는 환경이라면, `kube-proxy` 프로세스가 규칙을 계산하고 커널에 밀어넣느라 **CPU 한 코어를 100% 점유**하게 됩니다. 
* 이로 인해 같은 노드에 떠 있는 다른 앱(Pod)들이 CPU 스케줄링에서 밀려나며 연쇄적인 성능 저하가 일어납니다.


#### 3. Cilium(eBPF)은 무엇이 다른가?

Cilium은 이 "전체 덮어쓰기" 굴레에서 완전히 벗어났습니다.

* **Incremental Update (증분 업데이트):** Cilium은 규칙을 **eBPF Map(Hash Table)**에 저장합니다. 10,000개의 서비스가 있어도 특정 서비스의 IP가 바뀌면 **Map의 특정 Key값 하나만 딱 교체**합니다. 
* **Lock-free에 가까운 설계:** 전체 테이블을 잠그지 않고 데이터 구조만 변경하므로 패킷 흐름에 지연이 거의 없습니다.
* **상수 시간 탐색:** 서비스가 몇 개든 상관없이 Hash 기반으로 목적지를 찾으므로 $O(1)$ 성능을 보장합니다.


---
## Part 3. 질문 2

### 질문
ExternalTrafficPolicy: Local 설정이 되어 있는 Service의 경우, kube-proxy는 이를 구현하기 위해 iptables 규칙을 어떻게 다르게 생성하는가?

`ExternalTrafficPolicy: Local` 설정은 쿠버네티스 네트워킹에서 **"불필요한 네트워크 홉(Hop)을 줄이고 클라이언트의 원본 IP(Source IP)를 보존"**하기 위해 사용되는 아주 중요한 옵션입니다.

`kube-proxy`가 이 옵션을 처리하기 위해 `iptables` 규칙을 어떻게 다르게 생성하는지, 그 "차이점"을 중심으로 설명해 드릴게요.


### 학습 정리

#### 1. 기본값(`Cluster`) vs `Local`의 동작 차이

먼저 `iptables` 규칙이 왜 달라져야 하는지 배경을 이해해야 합니다.

* **`Cluster` (기본값):** 외부 트래픽이 노드 A로 들어왔는데 파드가 노드 B에 있다면, 노드 A의 `iptables`는 패킷을 노드 B로 포워딩합니다. 이때 응답을 받기 위해 **SNAT(Source NAT)**를 수행하므로 파드는 클라이언트의 실제 IP를 알 수 없습니다.
* **`Local`:** 외부 트래픽이 들어온 **그 노드에 파드가 있을 때만** 전달합니다. 만약 해당 노드에 파드가 없다면 패킷을 드롭(Drop)합니다. 이 덕분에 다른 노드로 보낼 필요가 없어 SNAT를 안 해도 되고, **원본 IP가 유지**됩니다.


#### 2. `iptables` 규칙의 핵심 차이점

`kube-proxy`는 `Local` 설정 시 크게 두 가지 부분에서 규칙을 다르게 생성합니다.

#### **① `KUBE-XLB-XXXX` 체인의 분기 (External Load Balancer)**
외부 트래픽이 노드포트(NodePort)나 로드밸런서 IP로 들어올 때 거치는 체인에서 분기가 일어납니다.

* **Cluster 모드:** `KUBE-SVC-XXXX` 체인으로 보내서, 노드 내부에 파드가 있든 없든 전체 클러스터의 파드 중 하나로 랜덤하게 보냅니다.
* **Local 모드:** `KUBE-XLB-XXXX` 체인에서 **해당 노드에 로컬 파드가 있는지 확인**하는 규칙을 먼저 둡니다.
    * 로컬 파드가 있다면? → `KUBE-SVL-XXXX` (Service Local) 체인으로 보냅니다.
    * 로컬 파드가 없다면? → 패킷을 **DROP**하거나 거부합니다.

#### **② SNAT(Masquerade) 생략**
가장 결정적인 차이입니다. `iptables`의 `POSTROUTING` 단계에서 발생하는 마스커레이딩 규칙이 적용되지 않습니다.

* **Cluster 모드:** 다른 노드로 패킷이 넘어갈 수 있으므로, 돌아오는 길을 찾기 위해 노드의 IP로 출발지 주소를 바꿉니다(`-j MASQUERADE`).
* **Local 모드:** 다른 노드로 절대 보내지 않음을 보장하기 때문에, **SNAT 규칙을 태우지 않고** 파드로 직접 보냅니다. 이 덕분에 파드는 TCP 헤더에서 클라이언트의 진짜 IP를 볼 수 있게 됩니다.


#### 3. 하이라이트: Health Check NodePort의 생성
`Local` 모드를 쓰면 외부 로드밸런서(AWS NLB, GCP LB 등)가 **"어느 노드에 파드가 떠 있는지"** 알아야 트래픽을 보낼 수 있습니다.

이를 위해 `kube-proxy`는 별도의 **Health Check 전용 포트**(기본 30000~32767 사이)를 엽니다.
* 노드에 해당 서비스의 파드가 **1개 이상 있으면** → `200 OK` 응답.
* 파드가 **없으면** → 응답 없음 혹은 에러.
* 외부 로드밸런서는 이 응답을 보고 파드가 있는 노드로만 트래픽을 쏴줍니다. (이 로직 역시 `kube-proxy`가 관리합니다.)

#### 4. 정리: `iptables` 규칙 비교

| 항목 | `ExternalTrafficPolicy: Cluster` | `ExternalTrafficPolicy: Local` |
| :--- | :--- | :--- |
| **핵심 목표** | 모든 파드에 균등한 부하 분산 | 원본 IP 보존 및 네트워크 홉 최소화 |
| **SNAT 여부** | 수행함 (Source IP 유실) | **수행 안 함 (Source IP 보존)** |
| **타 노드 전송** | 가능 | **불가능 (로컬 파드로만 전송)** |
| **규칙 특징** | 모든 엔드포인트가 타겟에 포함됨 | **로컬 엔드포인트만** 타겟으로 지정됨 |

