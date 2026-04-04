# 📚 Week 3 Study: Calico Deep Dive
> **학습 목표:** Calico의 핵심 아키텍처를 파악하고, BGP를 통한 L3 라우팅 원리와 iptables를 이용한 보안 정책 적용 과정을 심층 분석한다.

---

## 0. Introduction: 왜 지금 Calico인가?
4주차부터 진행될 **Cilium(eBPF)은** 기존 네트워킹의 한계를 극복하기 위해 등장했습니다. 그 '기존 방식'의 정점에 있는 것이 바로 **Calico**입니다. 
Calico가 어떻게 **BGP**를 활용해 대규모 클러스터에서 성능을 뽑아내는지, 그리고 **iptables**를 얼마나 영리하게 사용하는지 이해하는 것이 이번 주차의 핵심입니다.

Calico를 사용한 클러스터 구성은 진행해보지 않았기에, Calico 자체도 잘 몰라서 해당 구현체의 컨셉을 이해하는 것을 중점적으로 진행합니다~

---

## 1. Calico Networking의 철학
Calico는 "네트워크는 단순해야 한다"는 철학을 가지고 있습니다.

### 1.1 Why BGP? (Native L3 Routing)
대부분의 CNI(Flannel, 초기 모델들)는 **Overlay(캡슐화)** 방식을 사용합니다. 하지만 Calico는 가능한 경우 **Native L3** 방식을 지향합니다.

* **Overlay 방식의 한계:** 패킷을 한 번 더 감싸는(Encapsulation) 과정에서 CPU 리소스가 소모되고, MTU(Maximum Transmission Unit) 이슈로 인해 네트워크 성능이 저하될 수 있습니다.
* **BGP의 해답:** 캡슐화 대신, 각 노드가 서로의 Pod IP 대역 정보를 **BGP(Border Gateway Protocol)** 로 교환합니다. 패킷에 추가적인 헤더를 붙이지 않고도 목적지 노드를 찾아갈 수 있게 됩니다.



### 1.2 "Node as a Router" (노드는 라우터다)
Calico 환경에서 각 쿠버네티스 노드는 단순히 연산만 하는 곳이 아니라, 하나의 **소프트웨어 라우터**로 동작합니다.

* **BGP Peer:** 노드들은 서로 BGP Peer가 되어 라우팅 정보를 공유합니다.
* **Direct Routing:** 동일 서브넷(L2) 환경이라면, 패킷은 별도의 가공 없이 물리 스위치를 타고 마치 일반 서버 통신처럼 목적지 Pod으로 직행합니다.
* **결과:** 네트워크 홉(Hop)이 줄어들고, 가시성이 확보되며, 물리 네트워크 장비와의 연동이 매우 강력해집니다.

---

### 🔍 Checkpoint: BGP 사용에 대한 질문
> **Q: "BGP는 인터넷에서나 쓰는 거창한 프로토콜 아닌가요? 쿠버네티스 안에서 쓰기엔 너무 무겁지 않을까요?"**
>
> **A:** BGP는 전 세계의 인터넷 경로를 관리할 만큼 **확장성**이 검증된 프로토콜입니다. Calico는 이를 내부적으로 가볍게 구현한 `Bird`를 사용합니다. 오히려 수천 대의 노드가 있는 환경에서 라우팅 정보를 가장 안정적으로 동기화할 수 있는 최적의 도구입니다. (gemini의 답입니다. 틀릴 수도,,, 참고만 하십쇼)

---

## 2. BGP(Border Gateway Protocol) 분석
BGP는 Calico 네트워킹의 '두뇌' 역할을 합니다. 노드들이 서로 어디에 어떤 Pod이 있는지 대화하는 언어라고 이해하면 쉽습니다.

### 2.1 BGP의 핵심 개념: "지도 제작자"
BGP는 단순히 데이터를 보내는 통로가 아니라, **경로 정보를 교환하는 약속**입니다. 
* **AS(Autonomous System):** 하나의 관리 주체가 운영하는 네트워크 집합입니다. (쿠버네티스 클러스터 전체를 하나의 AS로 볼 수 있습니다.)
* **Peering:** 두 라우터(노드)가 연결되어 라우팅 정보를 주고받기로 약속한 상태입니다.
* **Advertisement:** "우리 노드에 `10.244.1.0/24` 대역의 Pod들이 있으니 이쪽으로 보내!"라고 주변에 알리는 행위입니다.

### 2.2 Calico의 두 가지 BGP 운영 모드
클러스터의 규모와 네트워크 환경에 따라 두 가지 방식 중 하나를 선택하게 됩니다.

#### ① Node-to-Node Mesh (Full Mesh)
모든 노드가 서로 1:1로 BGP Peering을 맺는 방식입니다.
* **장점:** 설정이 매우 간단하며 별도의 중앙 설정이 필요 없습니다.
* **단점:** 노드 수가 $N$개일 때 연결 수가 $N(N-1)/2$로 급증합니다. 노드가 많아지면 메시지 오버헤드가 커져 대규모 클러스터(보통 100개 이상)에서는 권장되지 않습니다.

#### ② Route Reflector (RR)
특정 노드(또는 전용 장비)를 '반장(RR)'으로 지정하고, 모든 노드는 반장에게만 경로 정보를 보고하는 방식입니다.
* **장점:** 연결 수가 획기적으로 줄어들어 수천 대 규모의 확장이 가능합니다.
* **동작:** 노드 A가 반장에게 정보를 주면, 반장이 클러스터 내의 모든 다른 노드에게 해당 경로 정보를 전파합니다.

### 2.3 [Deep Dive] 실제 환경에서의 확인 (Hands-on)
이론을 넘어 실제 노드에서 BGP가 어떻게 작동하는지 확인하는 것이 핵심입니다.

* **BGP 상태 확인 (`calicoctl` 활용):**
    ```bash
    # 노드 간 피어링 상태 확인 (State가 Established여야 정상)
    root@calico-cp:~# calicoctl node status
    Calico process is running.

    IPv4 BGP status
    +---------------+-------------------+-------+------------+-------------+
    | PEER ADDRESS  |     PEER TYPE     | STATE |   SINCE    |    INFO     |
    +---------------+-------------------+-------+------------+-------------+
    | 192.168.0.211 | node-to-node mesh | up    | 2026-03-30 | Established |
    | 192.168.0.38  | node-to-node mesh | start | 15:32:21   | Passive     |
    +---------------+-------------------+-------+------------+-------------+

    IPv6 BGP status
    No IPv6 peers found.
    # 여기서 211은 worker 1번 노드, 38은 2번 노드입니다
    ```
* **커널 라우팅 테이블 확인:**
    BGP가 성공적으로 동작한다면, 리눅스 커널 라우팅 테이블에 다른 노드의 Pod 대역이 등록됩니다.
    ```bash
    # 'proto bird'라고 표시된 항목이 BGP(Bird)가 넣어준 경로입니다.
    root@calico-cp:~# ip route | grep bird
    192.168.0.0/26 via 192.168.0.211 dev tunl0 proto bird onlink
    blackhole 192.168.0.128/26 proto bird
    ```

### 🔍 Checkpoint: 실습 결과 심층 분석

실제 환경에서 `calicoctl node status`를 실행했을 때 나타나는 현상들에 대한 해석입니다.

#### Q1. 왜 어떤 노드는 `Established`이고, 어떤 노드는 `Passive`인가요?
* **답변:** BGP는 TCP(Port 179)를 사용합니다. 두 노드(A, B)가 서로 연결을 맺으려 할 때, 동시에 서로에게 "연결하자!"라고 요청하면 연결이 꼬일 수 있습니다. 이를 막기 위해 BGP는 **Router ID(보통 IP 주소)** 가 높은 쪽이 주도권을 갖게 하거나, 먼저 연결을 시도한 쪽을 수용하는 메커니즘을 가집니다.
    * **Established**: worker1 노드와는 이미 서로 대화가 끝나서 지도를 주고받는 상태입니다.
    * **Passive:** worker2 노드가 먼저 CP(Control Plane) 노드에게 연결을 시도했고, CP 노드는 "알았어, 네가 연결 요청을 보냈으니 나는 가만히 서서(Passive) 네 요청을 받아줄게"라고 응답한 상태입니다.
    * **결론:** 한쪽이 `Established`라면 통신은 정상이지만, 만약 양쪽 모두 `Passive`에서 멈춰 있거나 `Connect` 상태라면 **방화벽(TCP 179 포트)** 설정을 의심해야 합니다.
    * **추가:** 상태가 start이면서 Passive라면 (현재 worker2 상황) 상대방(38번)으로부터 연결이 오기를 기다리고 있는 대기 상태를 의미합니다. (클러스터에 원인 모를 네트워크 문제때문에 그렇습니다.. ㅠ)

#### Q2. `ip route` 결과에서 `dev tunl0`는 무엇을 의미하나요?
* **답변:** 현재 Calico가 **IPIP(IP-in-IP) 모드**로 동작하고 있다는 증거입니다.
    * `tunl0`는 커널의 터널 인터페이스로, 다른 노드로 패킷을 보낼 때 원본 패킷을 다른 IP 패킷 안에 감싸서(Encapsulation) 보냅니다.
    * 만약 순수 L3(Native) 모드라면 `dev eth0`와 같이 물리 인터페이스가 직접 나타납니다.

#### Q3. 특정 노드의 경로가 `ip route`에 보이지 않는다면?
* 실습 결과에서 211번 노드 경로는 보이지만 38번 노드 경로가 보이지 않는다면, 38번 노드와의 **BGP Peering이 아직 완료되지 않았음**을 뜻합니다. 38번 노드에 접속하여 `calico-node` 로그를 확인해 봐야 합니다.
---
## 3. Calico 핵심 컴포넌트: 네트워크의 설계자와 집행자

Calico는 여러 독립적인 컴포넌트가 협력하여 동작합니다. 각 노드에 떠 있는 `calico-node` 파드 안에는 다음의 핵심 프로세스들이 함께 실행되고 있습니다.

### 3.1 Felix (노드의 총괄 관리자)
Felix는 각 노드에서 실행되는 에이전트로, Calico의 **실질적인 뇌** 역할을 합니다.
* **주요 역할:** * 인터페이스 관리: Pod이 생성될 때 가상 인터페이스(Veth)를 생성하고 관리합니다.
    * 라우팅 설정: 커널 라우팅 테이블에 정보를 직접 써넣습니다.
    * **보안 정책 집행:** Kubernetes Network Policy를 해석하여 **iptables** 규칙으로 변환/적용합니다.

### 3.2 BIRD (BGP의 전령)
* **주요 역할:** Felix가 설정한 로컬 라우팅 정보를 다른 노드들에게 BGP 프로토콜로 전파합니다.
* **특징:** 매우 가볍고 안정적인 오픈소스 BGP 라우팅 스택입니다.

### 3.3 Confd (BIRD의 비서)
BIRD는 정적인 설정 파일을 읽어 동작하는데, 쿠버네티스 환경은 동적입니다.
* **주요 역할:** 데이터 저장소(Etcd/KDD)의 변화를 감시하다가, 네트워크 설정이 바뀌면 BIRD의 설정 파일을 실시간으로 갱신하고 프로세스를 리로드합니다.



---

## 4. [Deep Dive] Felix vs kube-proxy: 역할의 경계

실습 시 `iptables` 규칙을 보면 두 컴포넌트가 만든 규칙이 섞여 있어 혼란스러울 수 있습니다. 하지만 목적지는 완전히 다릅니다.

### 4.1 핵심 차이점 요약
| 구분 | **kube-proxy** | **Felix (Calico)** |
| :--- | :--- | :--- |
| **관리 대상** | **Service** (ClusterIP 등) | **Pod** (Endpoint) |
| **주요 작업** | VIP -> 실 IP 변환 (**DNAT**) | 패킷 허용/차단 (**Filtering**) |
| **적용 테이블** | `nat` 테이블 | `filter`, `raw` 테이블 |

### 4.2 패킷의 여정 (Service 호출 시)
패킷이 노드에 도착하여 실제 Pod에 도달하기까지의 순서는 다음과 같습니다.

1.  **PREROUTING (nat):** `kube-proxy` 규칙에 의해 목적지가 Service IP에서 실제 Pod IP로 바뀝니다.
2.  **FORWARD (filter):** `Felix`가 만든 `cali-` 체인 규칙들이 작동합니다. "이 출발지에서 이 Pod으로 가는 패킷이 정책상 허용되는가?"를 검사합니다.
3.  **POSTROUTING (nat):** 필요 시 출발지 IP를 노드 IP로 바꾸는 Masquerade가 일어납니다.

---

### 🔍 Checkpoint: Deep Dive 질문
> **Q: "Felix가 kube-proxy의 기능을 완전히 대신할 수는 없나요?"**
>
> **A:** Calico의 기본 모드에서는 역할이 분리되어 있습니다. 하지만 4주차에 배울 **Cilium**이나, Calico의 **eBPF 모드**를 활성화하면 `kube-proxy`를 아예 제거하고 Felix(또는 Cilium 에이전트)가 모든 처리를 도맡게 됩니다. 3주차까지는 이 둘의 "불편한 동거(iptables 공유)"를 이해하는 것이 중요합니다.

---

## 5. Overlay vs Underlay: 패킷의 옷차림 결정

실습에서 `ip route` 결과에 나온 `dev tunl0`는 Calico가 패킷을 **IPIP(IP-in-IP)** 라는 주머니에 담아 보내고 있다는 뜻입니다. 왜 이런 방식을 쓸까요?

### 5.1 Underlay (Native L3)
네트워크 인프라(물리 스위치 등)가 Pod IP 대역을 직접 알고 있는 방식입니다.
* **특징:** 패킷에 아무런 가공을 하지 않습니다. (`eth0`로 직접 나감)
* **장점:** 캡슐화 오버헤드가 없어 성능이 가장 좋고, MTU 문제가 거의 없습니다.
* **조건:** 모든 노드가 동일한 L2 네트워크에 있거나, 물리 라우터가 BGP를 통해 Pod 대역을 학습해야 합니다.

### 5.2 Overlay (IPIP / VXLAN)
물리 네트워크가 Pod IP를 모를 때, 기존 패킷을 다른 패킷으로 감싸서 전달하는 방식입니다.
* **IPIP (IP-in-IP):** IP 패킷 안에 또 다른 IP 패킷을 넣습니다. (Calico 기본값)
    * 실습 결과의 `tunl0`가 바로 이 터널 인터페이스입니다.
* **VXLAN:** L2 프레임을 UDP 패킷에 담아 보냅니다. (BGP를 쓸 수 없는 환경에서 주로 사용)
* **장점:** 하위 네트워크 구조와 상관없이 어디서든 통신이 가능합니다. (Public Cloud 등)
* **단점:** 헤더가 추가되므로 패킷 크기가 커져 **MTU(Maximum Transmission Unit)** 설정이 중요해집니다.



---

## 6. [Deep Dive] MTU의 함정과 성능 저하

Overlay 모드를 사용할 때 가장 주의해야 할 점은 **패킷 조각화(Fragmentation)**입니다.

* **MTU 1500의 법칙:** 일반적인 이더넷의 최대 패킷 크기는 1500바이트입니다.
* **오버헤드 발생:** * **IPIP:** 20바이트의 추가 헤더 사용 $\rightarrow$ 실제 데이터는 1480바이트만 가능.
    * **VXLAN:** 50바이트의 추가 헤더 사용 $\rightarrow$ 실제 데이터는 1450바이트만 가능.
* **결과:** 만약 Pod이 1500바이트를 꽉 채워 보내면, 네트워크 장비에서 패킷을 쪼개야 하므로 성능이 급격히 떨어지거나 패킷이 유실될 수 있습니다.

---

### 🔍 Checkpoint: 실습 결과와의 연결
> **Q: "내 실습 환경은 왜 IPIP(`tunl0`)로 잡혔을까요?"**
>
> **A:** Calico의 기본 설정(`ippool`)에서 `ipipMode`가 `Always` 또는 `CrossSubnet`으로 되어 있기 때문입니다. 
> ```bash
> # 설정 확인 방법
> root@calico-cp:~# calicoctl get ippool -o yaml --allow-version-mismatch
>    apiVersion: projectcalico.org/v3
>    items:
>    - apiVersion: projectcalico.org/v3
>    kind: IPPool
>    metadata:
>        creationTimestamp: "2026-03-30T13:22:00Z"
>        name: default-ipv4-ippool
>        resourceVersion: "1297"
>        uid: 3a00d1b9-7ba3-4f9b-b771-c25a3ec7900d
>    spec:
>        allowedUses:
>        - Workload
>        - Tunnel
>        blockSize: 26
>        cidr: 192.168.0.0/24
>        ipipMode: Always
>        natOutgoing: true
>        nodeSelector: all()
>        vxlanMode: Never
>    kind: IPPoolList
>    metadata:
>    resourceVersion: "636743"
>
> # 만약 모든 노드가 같은 서브넷에 있다면 `ipipMode: Never`로 설정하여 `tunl0`를 없애고 Native L3의 속도를 맛볼 수 있습니다.

---

## 실습: NetworkPolicy가 iptables로 변환되는 과정

### 테스트용 NetworkPolicy 생성
먼저, 특정 라벨(`app: nginx`)을 가진 Pod에 대해 80번 포트만 허용하고 나머지는 차단하는 정책을 만듭니다.

```yaml
# 1. 대상 파드 (Target)
apiVersion: v1
kind: Pod
metadata:
  name: haejun-nginx
  labels:
    user: haejun
    app: nginx
spec:
  containers:
  - name: nginx
    image: nginx
---
# 2. 허용된 클라이언트 (Allowed)
apiVersion: v1
kind: Pod
metadata:
  name: haejun-allower
  labels:
    user: haejun
    role: allower
spec:
  containers:
  - name: alpine
    image: alpine
    command: ["/bin/sh", "-c", "sleep 3600"]
---
# 3. 보안 정책 (NetworkPolicy)
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: haejun-policy
  namespace: default
spec:
  podSelector:
    matchLabels:
      user: haejun
      app: nginx
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          user: haejun
          role: allower
    ports:
    - protocol: TCP
      port: 80
```


먼저 정책을 적용할 대상과 클라이언트를 생성하고, 어느 노드에 배치되었는지 확인합니다.

```bash
root@calico-cp:~/haejun# kubectl get pod -o wide
NAME              READY   STATUS    RESTARTS   AGE     IP              NODE        
haejun-allower    1/1     Running   0          3m59s   192.168.0.70    calico-w2   
haejun-nginx      1/1     Running   0          3m59s   192.168.0.69    calico-w2   
```

* **결과 분석:** 두 파드 모두 **`calico-w2`** 노드에 배치되었습니다. 
* **핵심 포인트:** 정책은 타겟 파드가 있는 노드에만 생성되므로, 조회를 위해서는 반드시 **`calico-w2`** 노드로 접속해야 합니다.

---

### [Step 1] iptables 규칙(Rule) 분석
`haejun-nginx`가 떠 있는 `calico-w2` 노드에서 `haejun-policy` 키워드로 규칙을 검색합니다.

```bash
root@calico-w2:~# iptables-save | grep haejun-policy

-A cali-pi-_Eb-vckmN-lr2eREXQcs -p tcp \
  -m comment --comment "Policy default/knp.default.haejun-policy ingress" \
  -m set --match-set cali40s:dSXImMSA1iXOPxYRYLVorAP src \
  -m multiport --dports 80 \
  -j MARK --set-xmark 0x10000/0x10000
```

* **`-m set --match-set ... src`**: 패킷의 출발지(src) IP가 특정 **`ipset` 바구니**에 있는지 확인합니다.
* **`-m multiport --dports 80`**: YAML에 정의한 80번 포트 전용 규칙임을 증명합니다.
* **`-j MARK`**: 조건이 맞으면 패킷에 `0x10000` 마킹을 하여 최종 통과를 예약합니다.

---

### [Step 2] ipset (허용 리스트 바구니)의 실체
위 iptables 규칙이 참조하는 `ipset` 바구니 내부를 열어 실제 허용 대상을 확인합니다.

```bash
root@calico-w2:~# ipset list cali40s:dSXImMSA1iXOPxYRYLVorAP

Name: cali40s:dSXImMSA1iXOPxYRYLVorAP
Type: hash:net
...
Members:
192.168.0.70  # haejun-allower 파드의 실제 IP가 정확히 담겨 있음!
```

* **결과 분석:** `haejun-allower`의 IP(**`192.168.0.70`**)가 바구니에 담겨 있습니다. 파드가 재시작되어 IP가 바뀌면, **Felix**가 이 Members 리스트를 즉시 업데이트합니다.



---

### 💡 실습 핵심 인사이트 (Deep Dive)

1.  **분산 정책 집행 (Distributed Enforcement):** 정책은 클러스터 전체가 아닌, **대상 파드가 있는 노드**에만 국소적으로 적용되어 성능을 최적화합니다.
2.  **ipset의 효율성:** 허용할 파드가 많아져도 iptables 규칙을 수천 줄 늘리는 대신, 해시 기반의 `ipset`을 써서 **단 한 번의 연산**으로 판단합니다.
3.  **Felix의 실시간성:** 우리가 YAML을 던지면 각 노드의 Felix가 커널의 `iptables`와 `ipset`을 실시간으로 동기화합니다.