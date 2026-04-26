# Hubble

Cilium 위에 구축된 네트워크, 보안 관측 플랫폼

## 구성요소

### 1. Hubble (per-node)

- 각 노드에서 **DaemonSet**으로 동작
- Cilium agent에 내장되어 있으며 eBPF로 패킷 수준의 이벤트를 캡처
- gRPC 기반의 Observer API 제공

### 2. Hubble Relay

- 클러스터 전체의 Hubble 노드를 **집계**하는 중앙 컴포넌트
- 멀티 노드 환경에서 클러스터 전체 가시성 제공
- Hubble UI / CLI와 통신하는 창구 역할

### 3. Hubble UI

- 실시간 **서비스 맵(Service Map)** 을 웹 대시보드로 시각화
- 네임스페이스별 Pod 간 트래픽 흐름을 그래프로 표현
- 드롭/포워드 정책 결과도 확인 가능

### 4. Hubble CLI (`hubble`)

- 커맨드라인에서 플로우 조회 및 상태 확인
- 필터링 옵션이 풍부해서 디버깅에 유용

### 항목

| 항목 | 설명 |
| --- | --- |
| **L3/L4 플로우** | IP, 포트, 프로토콜 수준 트래픽 |
| **L7 플로우** | HTTP, gRPC, DNS, Kafka 등 애플리케이션 레이어 |
| **정책 판정** | Allow / Drop / Forward 결과 |
| **DNS 조회** | 어떤 Pod가 어떤 도메인을 조회했는지 |
| **서비스 의존성** | 서비스 간 통신 토폴로지 자동 생성 |

## 아키텍처

기존의 네트워크 모니터링은 애플리케이션 사이드카나 iptables 로그에 의존했음. 

<img width="1380" height="1160" alt="image" src="https://github.com/user-attachments/assets/40c4bf3c-5dc3-4f64-ba57-7b8f25248673" />

|  | 기존 (사이드카 + iptables) | Cilium + Hubble |
| --- | --- | --- |
| 모니터링 위치 | Pod 내부 (유저스페이스) | 커널 (eBPF) |
| 설정 필요 | Pod마다 사이드카 주입 | 없음 |
| 사각지대 | 사이드카 미주입 Pod | 없음 |
| L7 가시성 | 사이드카에서만 | eBPF 프로그램으로 직접 |
| 오버헤드 | Pod당 CPU/메모리 추가 | 매우 낮음 |
| 규모 확장 | Pod 수에 비례해서 증가 | 노드당 고정 |

<img width="1350" height="1230" alt="image" src="https://github.com/user-attachments/assets/621bfbf9-2a19-4636-b976-fde2ac378cb2" />


```bash
패킷 발생 → eBPF가 커널에서 낚아채기 → Cilium Agent로 전달 → Hubble Relay가 클러스터 전체 집계 → CLI/UI로 조회
```

- 설치과정
    
    ```bash
    ① 커널 모듈 설정
    sudo tee /etc/modules-load.d/k8s.conf <<EOF
    overlay
    br_netfilter
    EOF
    
    sudo modprobe overlay
    sudo modprobe br_netfilter
    ② 네트워크 파라미터 설정
    sudo tee /etc/sysctl.d/k8s.conf <<EOF
    net.bridge.bridge-nf-call-iptables  = 1
    net.bridge.bridge-nf-call-ip6tables = 1
    net.ipv4.ip_forward                 = 1
    EOF
    
    sudo sysctl --system
    ③ containerd 설치
    # Docker 저장소 추가
    sudo dnf config-manager --add-repo \
      https://download.docker.com/linux/rhel/docker-ce.repo
    
    # containerd 설치
    sudo dnf install -y containerd.io
    
    # 기본 설정 생성
    sudo containerd config default | sudo tee /etc/containerd/config.toml
    
    # SystemdCgroup 활성화 (필수!)
    sudo sed -i 's/SystemdCgroup = false/SystemdCgroup = true/' \
      /etc/containerd/config.toml
    
    # 시작
    sudo systemctl enable --now containerd
    ④ kubeadm / kubelet / kubectl 설치
    # 저장소 추가
    sudo tee /etc/yum.repos.d/kubernetes.repo <<EOF
    [kubernetes]
    name=Kubernetes
    baseurl=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/
    enabled=1
    gpgcheck=1
    gpgkey=https://pkgs.k8s.io/core:/stable:/v1.32/rpm/repodata/repomd.xml.key
    EOF
    
    # 설치
    sudo dnf install -y kubelet kubeadm kubectl
    
    # kubelet 활성화
    sudo systemctl enable kubelet
    ⑤ 클러스터 초기화
    sudo kubeadm init \
      --pod-network-cidr=10.244.0.0/16 \
      --skip-phases=addon/kube-proxy
    # kube-proxy는 Cilium이 대체하므로 skip
    
    # kubeconfig 설정
    mkdir -p $HOME/.kube
    sudo cp /etc/kubernetes/admin.conf $HOME/.kube/config
    sudo chown $(id -u):$(id -g) $HOME/.kube/config
    
    # 단일 노드에서 Pod 스케줄링 허용
    kubectl taint nodes --all node-role.kubernetes.io/control-plane-
    ⑥ Cilium + Hubble 설치
    # Helm 설치
    curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
    
    # Cilium 설치
    helm repo add cilium https://helm.cilium.io/
    helm repo update
    
    helm install cilium cilium/cilium \
      --namespace kube-system \
      --set kubeProxyReplacement=true \
      --set hubble.enabled=true \
      --set hubble.relay.enabled=true \
      --set hubble.ui.enabled=true
    
    # 설치 확인
    kubectl get pods -n kube-system
    cilium status
    
    ---------------
    # 트러블슈팅
    # iptables 규칙이 없어서 10.96.0.1로 못 감
    # kube-proxy를 skip했는데 Cilium이 아직 안 떠서 ClusterIP 라우팅이 안 됨
    # 기존 Cilium 삭제
    helm uninstall cilium -n kube-system
    
    # 10초 대기
    sleep 10
    
    # VM IP 직접 지정해서 재설치
    helm install cilium cilium/cilium \
      --namespace kube-system \
      --set kubeProxyReplacement=true \
      --set k8sServiceHost=172.30.1.100 \
      --set k8sServicePort=6443 \
      --set hubble.enabled=true \
      --set hubble.relay.enabled=true \
      --set hubble.ui.enabled=true
    
    # Pod 상태 지켜보기
    watch kubectl get pods -n kube-system
    ```
    

```bash
cilium status
    /¯¯\
 /¯¯\__/¯¯\    Cilium:             1 errors, 1 warnings
 \__/¯¯\__/    Operator:           1 errors, 2 warnings
 /¯¯\__/¯¯\    Envoy DaemonSet:    1 errors
 \__/¯¯\__/    Hubble Relay:       1 errors, 2 warnings
    \__/       ClusterMesh:        disabled

DaemonSet              cilium                   Desired: 1, Unavailable: 1/1
DaemonSet              cilium-envoy             Desired: 1, Unavailable: 1/1
Deployment             cilium-operator          Desired: 2, Unavailable: 2/2
Deployment             hubble-relay             Desired: 1, Unavailable: 1/1
Deployment             hubble-ui                Desired: 1, Unavailable: 1/1
Containers:            cilium                   Pending: 1
                       cilium-envoy             Running: 1
                       cilium-operator          Running: 1, Pending: 1
                       clustermesh-apiserver
                       hubble-relay             Pending: 1
                       hubble-ui                Pending: 1
Cluster Pods:          0/4 managed by Cilium
Helm chart version:    1.19.3
Image versions         cilium             quay.io/cilium/cilium:v1.19.3@sha256:2e61680593cddca8b6c055f6d4c849d87a26a1c91c7e3b8b56c7fb76ab7b7b10: 1
                       cilium-envoy       quay.io/cilium/cilium-envoy:v1.36.6-1776000132-2437d2edeaf4d9b56ef279bd0d71127440c067aa@sha256:ba0ab8adac082d50d525fd2c5ba096c8facea3a471561b7c61c7a5b9c2e0de0d: 1
                       cilium-operator    quay.io/cilium/operator-generic:v1.19.3@sha256:205b09b0ed6accbf9fe688d312a9f0fcfc6a316fc081c23fbffb472af5dd62cd: 2
                       hubble-relay       quay.io/cilium/hubble-relay:v1.19.3@sha256:5ee21d57b6ef2aa6db67e603a735fdceb162454b352b7335b651456e308f681b: 1
                       hubble-ui          quay.io/cilium/hubble-ui-backend:v0.13.3@sha256:db1454e45dc39ca41fbf7cad31eec95d99e5b9949c39daaad0fa81ef29d56953: 1
                       hubble-ui          quay.io/cilium/hubble-ui:v0.13.3@sha256:661d5de7050182d495c6497ff0b007a7a1e379648e60830dd68c4d78ae21761d: 1
Errors:                cilium             cilium                              1 pods of DaemonSet cilium are not ready
                       cilium-envoy       cilium-envoy                        1 pods of DaemonSet cilium-envoy are not ready
                       cilium-operator    cilium-operator                     2 pods of Deployment cilium-operator are not ready
                       hubble-relay       hubble-relay                        1 pods of Deployment hubble-relay are not ready
                       hubble-ui          hubble-ui                           1 pods of Deployment hubble-ui are not ready
Warnings:              cilium             cilium-lbzpj                        pod is pending
                       cilium-operator    cilium-operator-5f474d58b6-htf8c    pod is pending
                       cilium-operator    cilium-operator-5f474d58b6-htf8c    pod is pending
                       hubble-relay       hubble-relay-dfb6695bc-lzv5h        pod is pending
                       hubble-relay       hubble-relay-dfb6695bc-lzv5h        pod is pending
                       hubble-ui          hubble-ui-78f4688679-4mxwh          pod is pending
```

```bash
# Hubble CLI 설치 (arm64)
curl -L --remote-name-all \
  https://github.com/cilium/hubble/releases/latest/download/hubble-linux-arm64.tar.gz

tar xzvf hubble-linux-arm64.tar.gz
sudo mv hubble /usr/local/bin/
rm hubble-linux-arm64.tar.gz

# 버전 확인
hubble version
```

## 실습

```bash
# 탭 2에서 실행 (VM 안에서)
kubectl port-forward -n kube-system svc/hubble-ui 12000:80 --address 0.0.0.0 &

# 대시 보드 접속
http://172.30.1.100:12000

kubectl run nginx --image=nginx
kubectl run netshoot --image=nicolaka/netshoot --command -- sleep infinity

# 상태 확인
kubectl get pods -w

# nginx IP 확인하고 트래픽 발생
NGINX_IP=$(kubectl get pod nginx -o jsonpath='{.status.podIP}')
kubectl exec netshoot -- curl -s http://$NGINX_IP
```

<img width="2048" height="1234" alt="image" src="https://github.com/user-attachments/assets/f9c9fcc4-08a1-435d-b48e-1892dd5dcd24" />


### L7 가시화

A 파드가 B 파드에게 `/api/v1/user` 경로로 `GET` 요청을 보냈고, 결과로 `404 Not Found`를 받았다는 사실 파악가능

이 기능을 활용해 “외부 클라이언트는 `/public` 경로만 접근 가능하고, `/admin` 경로는 차단한다” 같은 **보안 정책**이 가능함

```bash
# nginx Pod에 L7 가시화 annotation 추가
kubectl annotate pod nginx \
  policy.cilium.io/proxy-visibility="<Ingress/80/TCP/HTTP>"
  
# 트래픽 발생시키기
NGINX_IP=$(kubectl get pod nginx -o jsonpath='{.status.podIP}')

# 여러 번 요청
kubectl exec netshoot -- curl -s http://$NGINX_IP
kubectl exec netshoot -- curl -s http://$NGINX_IP/없는페이지
kubectl exec netshoot -- curl -s http://$NGINX_IP/index.html

# L7 HTTP 플로우만 필터링
hubble observe --follow --protocol http
```

```bash
default/netshoot -> default/nginx:80  
  HTTP/1.1 GET / → 200 OK
default/netshoot -> default/nginx:80  
  HTTP/1.1 GET /없는페이지 → 404 Not Found
```

```bash
# 필터 없이 전체 플로우 확인
hubble observe --follow --pod nginx

Apr 25 17:01:07.857: default/netshoot:51742 (ID:25023) <- default/nginx:80 (ID:1688) to-endpoint FORWARDED (TCP Flags: SYN, ACK)
Apr 25 17:01:07.857: default/netshoot:51742 (ID:25023) -> default/nginx:80 (ID:1688) to-endpoint FORWARDED (TCP Flags: ACK)
Apr 25 17:01:07.857: default/netshoot:51742 (ID:25023) -> default/nginx:80 (ID:1688) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Apr 25 17:01:07.858: default/netshoot:51742 (ID:25023) <- default/nginx:80 (ID:1688) to-endpoint FORWARDED (TCP Flags: ACK, PSH)
Apr 25 17:01:07.858: default/netshoot:51742 (ID:25023) -> default/nginx:80 (ID:1688) to-endpoint FORWARDED (TCP Flags: ACK, FIN)
Apr 25 17:01:07.858: default/netshoot:51742 (ID:25023) <- default/nginx:80 (ID:1688) to-endpoint FORWARDED (TCP Flags: ACK, FIN)
Apr 25 17:01:07.858: default/netshoot:51742 (ID:25023) -> default/nginx:80 (ID:1688) to-endpoint FORWARDED (TCP Flags: ACK)
```
