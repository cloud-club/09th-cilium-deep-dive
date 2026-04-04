# Week2. k8s network (CNI, kube-proxy, service-pod)


## 1. CNI (Container Network Interface)
<img src="https://miro.medium.com/0*3L7yN7Wa3rO4nBBA.png" width="600">
<br>k8s에서 container들이 어떻게 통신할지를 정의한 표준 인터페이스다. 실제 네트워크 구현은 외부 플러그인(Calico, Flannel, Cilium)에게 맡긴다.

### 하는 일

1. 새로운 Pod가 생성되면 kubelet이 CNI 플러그인을 호출한다.
2. CNI 플러그인은 그 Pod에 가상 네트워크 인터페이스(**veth pair**)를 만들어 붙여준다.
   - veth는 양쪽이 연결된 가상 랜선으로 한쪽엔 Pod가 한쪽엔 host node가 꽂혀 있다.
3. Pod에 IP 주소를 할당하고 라우팅 규칙을 설정해서 다른 Pod나 노드와 통신할 수 있게 된다.

### k8s 1.24 이후 변경점

이전에는 `--network-plugin`, `--cni-bin-dir` 같은 옵션으로 kubelet이 CNI를 직접 관리했다.

1.24부터는 **컨테이너 런타임(containerd, CRI-O 등)** 이 CNI 플러그인을 로드하고 관리한다. 
<br>kubelet은 컨테이너 런타임에 **CRI(Container Runtime Interface)**로 명령을 내리고 컨테이너 런타임이 내부적으로 CNI를 호출한다.

```
kubelet → (CRI) → containerd → (CNI) → Cilium/Calico/Flannel
```




## 2. kube-proxy (iptables vs IPVS)

Pod의 IP는 Pod가 죽고 새로 뜰 때마다 바뀐다. 그래서 **Service**라는 고정된 가상 IP(ClusterIP)를 앞에 세워두고 실제 트래픽은 그 뒤에 있는 Pod들로 분산한다.
<br>이 분산을 실제로 처리하는 컴포넌트가 **kube-proxy**다. 
<br>kube-proxy는 모든 노드에 하나씩 떠 있으면서 Service와 Pod 사이의 연결 규칙을 노드의 커널에 심어두는 역할을 한다.

### 2-1. iptables 모드

리눅스의 iptables라는 packet-filtering 시스템을 이용한다.

- kube-proxy가 Service 하나당 여러 개의 iptables 규칙을 만들어둔다.
- 패킷이 들어오면 커널이 규칙들을 **위에서부터 하나씩 순서대로** 확인한다.
- 매칭되는 규칙에 따라 목적지 Pod로 **DNAT(Destination NAT)** 를 수행한다.
- Service 수가 많아질수록 규칙도 선형으로 증가 → 탐색 시간이 길어져 성능이 떨어진다. **(O(n))**


**CNI ↔ kube-proxy 커널 연동 주의**
<br>CNI가 컨테이너를 리눅스 브릿지에 연결하는 방식을 쓴다면 `net/bridge/bridge-nf-call-iptables` 커널 설정값을 **1**로 켜야 한다.
<br>이 값이 꺼져 있으면 브릿지를 통과하는 패킷이 iptables 규칙을 거치지 않고 그냥 지나가버려서, kube-proxy가 DNAT 규칙을 심어놔도 작동하지 않는다.
<br>CNI와 kube-proxy는 독립적인 컴포넌트처럼 보이지만 커널 수준에서 이렇게 맞물려 있다.

```
PREROUTING
  → KUBE-SERVICES        # ClusterIP 매칭
  → KUBE-SVC-XXXXX       # Pod 확률적 선택 (statistic 모듈)
  → KUBE-SEP-XXXXX       # DNAT 실행 (ClusterIP → Pod IP)
  → 실제 Pod IP:Port
```
<br>Pod가 여러 개면 KUBE-SVC 체인에서 확률 계산으로 균등 분배한다.

### 2-2. IPVS 모드

리눅스 커널의 IPVS(IP Virtual Server)를 사용한다. 원래 L4 로드밸런서로 설계됐다.

- 내부적으로 **해시 테이블**을 사용 → Service가 아무리 많아져도 일정한 속도로 목적지를 찾는다. **(O(1))**
- Round Robin 외에도 Least Connection, Source Hash 등 **다양한 LB 알고리즘** 선택 가능
- 대규모 클러스터에서 유리하다.

ClusterIP:Port를 Virtual Server로, Pod들을 Real Server로 등록하고 내부적으로 해시 테이블로 관리한다. 
<br>[프로토콜, VIP, Port] 조합을 키로 해시 테이블에서 바로 찾아간다.
```
sudo ipvsadm -Ln

TCP  10.105.180.242:80 rr
  -> 192.168.1.2:80   Masq  weight=1
  -> 192.168.1.3:80   Masq  weight=1
```

### 2-3. IPVS 모드에서도 iptables를 쓰는 이유
<br> IPVS가 처리 못하는 부분은 여전히 iptables가 보조한다.
<br>KUBE-MARK-MASQ — 패킷에 마킹해서 POSTROUTING에서 SNAT 처리
<br>NodePort 수신 — IPVS가 NodePort 패킷을 받을 수 있도록 dummy 인터페이스(kube-ipvs0)에 ClusterIP를 바인딩하고 iptables로 유도
<br>Network Policy — IPVS는 정책 필터링 기능이 없어서 iptables FORWARD 체인이 담당



| 항목 | iptables | IPVS |
|------|----------|------|
| 탐색 방식 | 선형 순회 | 해시 테이블 |
| 시간 복잡도 | O(n) | O(1) |
| LB 알고리즘 | Round Robin만 | RR, LC, SH 등 |
| 적합한 규모 | 소규모 | 대규모 |



## 3. Service → Pod 흐름

> **예시**: 클러스터 안의 클라이언트 Pod가 `my-service`라는 Service에 요청을 보내는 상황


<br>1. Pod 안에서 CoreDNS를 사용해 DNS 조회를 하고 `my-service`라는 이름을 ClusterIP로 바꾼다. 

<br>2. 클라이언트 Pod가 이 ClusterIP로 패킷을 보내면 패킷은 Pod의 veth 인터페이스를 타고 호스트 Node의 네트워크 스택으로 올라온다. 
<br>이때 패킷이 Node의 커널을 통과하면서 iptables 또는 IPVS 규칙이 동작한다.

<br>3. iptables 모드라면 kube-proxy가 미리 심어둔 규칙이 이 패킷을 가로채서 목적지 IP를 ClusterIP에서 실제 Pod IP로 바꿔치기한다(DNAT).

<br>4. 목적지가 실제 Pod IP로 바뀐 패킷은 CNI가 만들어둔 라우팅 경로를 타고 목적지 Pod로 향한다. 
<br>목적지 Pod가 같은 노드에 있으면 veth pair를 통해 바로 전달되고 다른 노드에 있으면 CNI 플러그인이 구성한 overlay network나 BGP routing을 타고 해당 노드의 Pod에 도달한다.

<br>5. Pod에서 응답이 돌아올 때는 conntrack이 이 연결을 기억하고 있다가 응답 패킷의 출발지 IP를 다시 ClusterIP로 되돌려준다. <br>덕분에 클라이언트 입장에서는 처음부터 끝까지 ClusterIP와만 통신한 것처럼 보인다.


## 4. 실습

### iptables 규칙 관찰
: kube-proxy가 Service를 만들면 iptables 규칙을 자동으로 추가한다.

```bash
# Service 만들기 전 규칙 수 확인
sudo iptables -t nat -L KUBE-SERVICES -n | wc -l

# Deployment + Service 생성
kubectl create deployment nginx --image=nginx -n <내 NS>
kubectl expose deployment nginx --port=80 -n <내 NS>
# Service ClusterIP 확인
kubectl get svc -n [ns명]

# Service 만든 후 규칙 수 확인 → 숫자가 늘어난 것 확인
sudo iptables -t nat -L KUBE-SERVICES -n | wc -l
# 실제 규칙 내용 보기
sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
```

### 결과
```
root@control-plane:~# sudo iptables -t nat -L KUBE-SERVICES -n | wc -l
9
root@control-plane:~# kubectl create deployment nginx --image=nginx -n mj
deployment.apps/nginx created
root@control-plane:~# kubectl expose deployment nginx --port=80 -n mj
service/nginx exposed
root@control-plane:~# kubectl get svc -n mj
NAME    TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
nginx   ClusterIP   10.105.180.242   <none>        80/TCP    5s
root@control-plane:~# sudo iptables -t nat -L KUBE-SERVICES -n | wc -l
10
root@control-plane:~# sudo iptables -t nat -L KUBE-SERVICES -n --line-numbers
Chain KUBE-SERVICES (2 references)
num  target                     prot opt source     destination
...
6    KUBE-SVC-RVCK3QUX57I6ZUEI  tcp  --  0.0.0.0/0  10.105.180.242   /* mj/nginx cluster IP */
...
ADDRTYPE match dst-type LOCAL
```