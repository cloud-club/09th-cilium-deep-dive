# Linux Network — K8S On-Premise 설치 시 네트워크 설정 상세 설명

On-premise에 Kubernetes를 설치할 때 반드시 수행해야 하는 Linux 네트워크 관련 설정을 각 항목별로 다룬다.

### 설치 가이드

#### 1. SELinux 비활성화
Kubernetes는 SELinux가 Enforcing 상태일 경우 POD 네트워크 설정에 문제가 발생할 수 있습니다.

```
$ sudo setenforce 0
$ sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

#### 2. 방화벽 비활성화
kubelet, calico, API server 등 포트 충돌을 방지하고, 외부 접근 제약이 없는 내부망이므로 방화벽을 비활성화 합니다.

```
$ sudo systemctl disable --now fiewalld
```

#### 3. Swap 비활성화
Kubernetes는 swap이 활성화된 시스템에서 동작하지 않습니다.

```
$ sudo swapoff -a
$ sudo sed -i '/swap/d' /etc/fstab
```

#### 4. 커널 모듈 설정 및 sysctl
네트워크 패킷 전달 및 IP 포워딩 기능 활성화 합니다.

```
$ cat << EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF

$ sudo modprobe br_netfilter
$ sudo modeprobe overlay

$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
EOF

$ sudo sysctl --system
```

---

## 1. SELinux 비활성화

### 1.1 SELinux란?

SELinux(Security-Enhanced Linux)는 **미국 NSA(국가안보국)가 개발한 리눅스 커널 보안 모듈**이다. 기존 리눅스의 임의 접근 제어(DAC, Discretionary Access Control) 방식을 보완하여 **강제 접근 제어(MAC, Mandatory Access Control)** 를 구현한다.

일반적인 리눅스 파일 권한(`rwxr-xr-x`)은 파일 소유자가 권한을 설정하는 DAC 방식이다. 반면 SELinux는 소유자와 무관하게 **커널이 직접 모든 프로세스와 파일에 보안 레이블(label)을 부여**하고, 미리 정의된 정책(policy)에 따라 접근을 허용하거나 차단한다.

```
[DAC 방식]
프로세스 A → 파일 B 접근 요청 → 파일의 owner/group/mode 확인 → 허용/거부

[MAC 방식 — SELinux]
프로세스 A → 파일 B 접근 요청 → 커널이 정책 테이블 조회
                                  프로세스 레이블(context) 확인
                                  파일 레이블(context) 확인
                                  정책에 해당 조합이 있으면 허용, 없으면 거부
```

#### SELinux의 3가지 동작 모드

| 모드 | 설명 |
|------|------|
| **Enforcing** | 정책을 위반하는 모든 접근을 차단하고 로그를 남긴다. 기본값. |
| **Permissive** | 차단하지는 않지만 정책 위반 사항을 로그로 기록한다. 디버깅 용도. |
| **Disabled** | SELinux를 완전히 끈다. |

#### SELinux Context(레이블)란?

SELinux는 모든 파일, 프로세스, 소켓, 포트에 아래와 같은 형식의 레이블을 붙인다.

```bash
# 파일의 SELinux context 확인
ls -Z /etc/passwd
# system_u:object_r:passwd_file_t:s0 /etc/passwd
#  ^user     ^role    ^type       ^level

# 프로세스의 SELinux context 확인
ps -eZ | grep nginx
# system_u:system_r:httpd_t:s0  nginx
```

이 중 가장 중요한 것은 **type**(`httpd_t`, `passwd_file_t` 등)이며, SELinux 정책의 핵심은 "어떤 type의 프로세스가 어떤 type의 파일/소켓에 접근할 수 있는가"를 정의하는 것이다.

---

### 1.2 명령어 설명

```bash
# 현재 세션에서 즉시 Permissive 모드로 전환 (재부팅 후 원래대로 돌아옴)
$ sudo setenforce 0

# 설정 파일을 수정하여 재부팅 후에도 Permissive 모드 유지
$ sudo sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config
```

**`setenforce 0`**

`setenforce`는 SELinux의 동작 모드를 런타임에서 즉시 전환하는 명령어다.

- `setenforce 0` → Permissive 모드 (정책 위반을 차단하지 않고 로그만 남김)
- `setenforce 1` → Enforcing 모드 (정책 위반 차단)

이 명령어는 **현재 실행 중인 커널에만 적용**되며, 재부팅하면 `/etc/selinux/config` 파일의 설정으로 되돌아간다.

**`sed -i 's/^SELINUX=enforcing/SELINUX=permissive/' /etc/selinux/config`**

`/etc/selinux/config` 파일에서 `SELINUX=enforcing` 라인을 `SELINUX=permissive`로 영구 변경한다.

```
# /etc/selinux/config 파일 내용 (변경 전)
SELINUX=enforcing     ← 이 줄을

# 변경 후
SELINUX=permissive    ← 이렇게 바꿈
```

현재 모드 확인:

```bash
$ getenforce
Permissive

$ sestatus
SELinux status:                 enabled
SELinuxfs mount:                /sys/fs/selinux
SELinux mount point:            /sys/fs/selinux
Loaded policy name:             targeted
Current mode:                   permissive
Mode from config file:          permissive
```

---

### 1.3 K8S에서 SELinux를 비활성화해야 하는 이유

SELinux는 강력한 보안 기능이지만, Kubernetes가 동작하면서 수행하는 여러 작업들이 기본 SELinux 정책에 정의되어 있지 않아 차단될 수 있다.

#### 문제 1 — CNI 플러그인의 네트워크 소켓 접근 차단

Calico, Flannel 등 CNI 플러그인은 Pod 네트워크를 구성하기 위해 아래 작업을 수행한다.

- 네트워크 인터페이스(veth) 생성
- iptables 규칙 직접 수정
- `/proc/sys/net/` 경로의 커널 파라미터 변경
- 특정 네트워크 소켓 생성 및 바인딩

SELinux Enforcing 모드에서는 CNI 플러그인의 SELinux context(`container_runtime_t`)가 이 작업들에 필요한 권한을 갖지 못해 **AVC(Access Vector Cache) denied 에러**가 발생한다.

```
# /var/log/audit/audit.log 에 쌓이는 SELinux 차단 로그 예시
type=AVC msg=audit: avc: denied { create } for pid=1234
comm="calico-node" scontext=system_u:system_r:container_runtime_t:s0
tcontext=system_u:object_r:net_socket_t:s0 tclass=rawip_socket
```

#### 문제 2 — kubelet의 컨테이너 볼륨 마운트 차단

kubelet이 Pod에 볼륨을 마운트할 때 호스트의 디렉토리를 컨테이너에 바인드 마운트하는데, SELinux는 컨테이너 프로세스가 호스트 파일시스템 레이블의 파일에 접근하는 것을 차단할 수 있다.

#### 문제 3 — containerd/CRI-O의 컨테이너 실행 제한

컨테이너 런타임이 새로운 네임스페이스를 생성하거나, Overlay 파일시스템을 마운트하거나, 프로세스 권한을 조정할 때 SELinux 정책이 이를 차단하는 경우가 있다.

#### 왜 Disabled가 아닌 Permissive인가?

`Disabled`로 설정하면 SELinux 모듈 자체가 꺼지므로, 이후 다시 `Enforcing`으로 전환할 때 **파일시스템 전체의 레이블을 재생성(relabeling)** 해야 하는 부담이 생긴다. 반면 `Permissive`는 SELinux 모듈은 활성 상태로 유지하면서 차단만 하지 않으므로 운영 중에도 로그로 정책 위반 사항을 확인할 수 있고, 나중에 `Enforcing`으로 전환하기도 쉽다.

---

## 2. 방화벽 비활성화

### 2.1 방화벽(firewalld)이란?

#### 2.1.1 리눅스 방화벽의 계층 구조

리눅스의 방화벽 체계는 여러 계층으로 이루어져 있다. 이를 이해하려면 아래 구조를 먼저 파악해야 한다.

```
[사용자 공간]
  firewalld (RHEL/CentOS)
  ufw       (Ubuntu)
      ↓ 규칙 번역
  iptables / nftables (명령어 도구)
      ↓ 규칙 전달
[커널 공간]
  Netfilter (커널 패킷 필터링 프레임워크)
      ↓
  실제 패킷 처리 (허용 / 차단 / 변환)
```

**Netfilter**는 리눅스 커널 안에 내장된 패킷 처리 프레임워크다. 네트워크 패킷이 커널을 통과할 때 특정 지점(hook point)마다 등록된 규칙을 실행한다.

**iptables**는 이 Netfilter에 규칙을 추가/삭제/조회하는 **사용자 공간 도구(userspace tool)** 다. 커널 4.x 이후부터는 `nftables`가 후계자로 등장했지만, 하위 호환성을 위해 `iptables` 명령어는 여전히 동작한다.

**firewalld**는 RHEL/CentOS 계열에서 iptables/nftables를 더 편리하게 관리하기 위한 **데몬(daemon)** 이다. Zone, Service, Port 개념으로 규칙을 관리하며, 내부적으로는 결국 iptables 또는 nftables 규칙으로 변환되어 Netfilter에 적용된다.

```
firewalld → (변환) → iptables/nftables 규칙 → Netfilter → 패킷 처리
```

#### 2.1.2 Netfilter의 Hook Point와 iptables 체인

Netfilter는 패킷 흐름의 5개 지점에 Hook을 걸어 규칙을 적용한다. iptables는 이 Hook들을 **체인(Chain)** 이라는 이름으로 노출한다.

```
외부에서 들어오는 패킷
        │
   PREROUTING ← (DNAT, 포트 포워딩 처리)
        │
        ├─── [목적지가 로컬 호스트?] ─── INPUT → 로컬 프로세스
        │
        └─── [다른 인터페이스로 전달?] ─── FORWARD → POSTROUTING → 외부
                                                              ↑
                                         로컬 프로세스의 출력 패킷도 여기
                                         (OUTPUT → POSTROUTING)
```

| 체인 | 동작 시점 | K8S에서의 역할 |
|------|----------|--------------|
| PREROUTING | 패킷이 도착하자마자 | Service IP → Pod IP DNAT 변환 |
| INPUT | 로컬 프로세스로 전달되기 전 | 외부에서 API Server(6443) 접근 제어 |
| FORWARD | 다른 인터페이스로 넘어가기 전 | Pod 간 트래픽 필터링 |
| OUTPUT | 로컬 프로세스가 패킷 생성 후 | 로컬에서 나가는 트래픽 제어 |
| POSTROUTING | 패킷이 인터페이스를 떠나기 전 | SNAT, Masquerade |

#### 2.1.3 iptables의 테이블(Table) 구조

iptables는 체인 외에도 **테이블**이라는 개념으로 규칙을 분류한다.

| 테이블 | 역할 | 주요 사용 주체 |
|--------|------|--------------|
| **filter** | 패킷 허용/차단 | firewalld, 일반 방화벽 규칙 |
| **nat** | IP/포트 주소 변환 | kube-proxy (Service IP 변환) |
| **mangle** | 패킷 헤더 수정 | 고급 라우팅, QoS |
| **raw** | conntrack 예외 처리 | 성능 최적화 |

K8S의 kube-proxy는 주로 `nat` 테이블과 `filter` 테이블에 규칙을 추가한다.

---

### 2.2 명령어 설명

```bash
$ sudo systemctl disable --now firewalld
```

`systemctl`은 systemd를 통해 서비스를 관리하는 명령어다.

- `disable` : 부팅 시 자동 시작을 비활성화한다 (심볼릭 링크 제거)
- `--now` : `disable`과 동시에 현재 실행 중인 서비스를 즉시 중지한다 (`stop`을 함께 수행)

```bash
# 아래 두 명령어와 동일한 효과
$ sudo systemctl stop firewalld
$ sudo systemctl disable firewalld
```

상태 확인:

```bash
$ sudo systemctl status firewalld
● firewalld.service - firewalld - dynamic firewall daemon
     Loaded: loaded (/usr/lib/systemd/system/firewalld.service; disabled)
     Active: inactive (dead)
```

---

### 2.3 K8S에서 방화벽을 비활성화해야 하는 이유

#### 2.3.1 K8S가 사용하는 포트들이 기본적으로 차단됨

firewalld의 기본 Zone은 대부분의 포트를 차단한다. K8S 클러스터가 정상 동작하려면 아래 포트들이 모두 열려 있어야 한다.

**Control Plane 노드**

| 포트 | 프로토콜 | 용도 |
|------|---------|------|
| 6443 | TCP | Kubernetes API Server |
| 2379-2380 | TCP | etcd (클라이언트/피어 통신) |
| 10250 | TCP | kubelet API |
| 10259 | TCP | kube-scheduler |
| 10257 | TCP | kube-controller-manager |

**Worker 노드**

| 포트 | 프로토콜 | 용도 |
|------|---------|------|
| 10250 | TCP | kubelet API |
| 10256 | TCP | kube-proxy |
| 30000-32767 | TCP/UDP | NodePort Service 범위 |

**CNI 플러그인 (Calico 기준)**

| 포트 | 프로토콜 | 용도 |
|------|---------|------|
| 179 | TCP | BGP (노드 간 라우팅 정보 교환) |
| 4789 | UDP | VXLAN (오버레이 네트워크) |
| 5473 | TCP | Typha (대규모 클러스터 최적화) |

#### 2.3.2 firewalld와 kube-proxy의 iptables 규칙 충돌

K8S의 kube-proxy는 Service가 생성될 때마다 iptables에 규칙을 직접 추가한다.

```
# kube-proxy가 추가하는 iptables 규칙 예시 (nat 테이블)
-A KUBE-SERVICES -d 10.96.0.1/32 -p tcp --dport 443 -j KUBE-SVC-XXXXX
-A KUBE-SVC-XXXXX -j KUBE-SEP-YYYYY
-A KUBE-SEP-YYYYY -p tcp -j DNAT --to-destination 10.244.0.5:8080
```

문제는 firewalld도 자체적으로 iptables 규칙을 관리한다는 점이다. firewalld가 재시작되거나 Zone 설정이 변경될 때 **kube-proxy가 추가한 규칙들을 지워버리는 상황**이 발생한다. 이렇게 되면 클러스터 내 Service 통신이 갑자기 끊기는 장애가 발생한다.

```
[정상 상태]
iptables nat 테이블:
  - KUBE-SERVICES 체인 (kube-proxy가 생성)
  - KUBE-SVC-* 체인들
  - firewalld 규칙들

[firewalld reload 후]
iptables nat 테이블:
  - firewalld 규칙들만 남음
  - KUBE-SERVICES 체인 ← 사라짐! → Service 통신 불가
```

이 충돌 문제 때문에 K8S 공식 문서에서도 firewalld와 kube-proxy를 함께 사용하지 말 것을 권고한다.

#### 2.3.3 내부망 on-premise 환경에서의 현실적인 판단

외부 인터넷에 직접 노출되지 않는 **폐쇄망(내부망) 환경**에서는 방화벽보다 네트워크 스위치/라우터 레벨의 ACL(Access Control List)로 접근을 제어하는 경우가 일반적이다. 이 경우 OS 레벨의 firewalld를 끄는 것이 K8S 안정성을 위한 현실적인 선택이다.

> 만약 보안 요구사항으로 firewalld를 유지해야 한다면, `--now firewalld` 대신 필요한 포트를 모두 수동으로 허용하고, `firewall-cmd --reload` 이후에도 kube-proxy 규칙이 유지되도록 별도 설정이 필요하다.

---

## 4. 커널 모듈 설정 및 sysctl

### 4.1 커널 모듈(Kernel Module)이란?

리눅스 커널은 모든 기능을 처음부터 메모리에 올려두지 않는다. **커널 모듈**은 필요할 때만 커널에 동적으로 적재(load)하거나 제거(unload)할 수 있는 **커널 기능의 단위**다.

```bash
# 현재 로드된 모듈 목록 확인
$ lsmod
Module                  Size  Used by
br_netfilter           32768  0
bridge                307200  1 br_netfilter
overlay               151552  0
...

# 특정 모듈 정보 확인
$ modinfo br_netfilter
filename:       /lib/modules/5.15.0/kernel/net/bridge/br_netfilter.ko
description:    Linux ethernet netfilter firewall bridge
```

모듈 파일은 `/lib/modules/$(uname -r)/` 경로에 `.ko` 확장자로 저장되어 있다.

#### 영구 로드 설정

`modprobe` 명령어는 현재 세션에서만 모듈을 로드한다. 재부팅 후에도 자동으로 로드하려면 `/etc/modules-load.d/` 디렉토리에 설정 파일을 생성해야 한다.

```bash
# /etc/modules-load.d/k8s.conf 생성
$ cat << EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
overlay
EOF
```

systemd가 부팅 시 이 디렉토리의 파일을 읽어 나열된 모듈을 자동으로 로드한다.

---

### 4.2 `modprobe overlay` — OverlayFS 모듈

#### overlay(OverlayFS)란?

OverlayFS는 **여러 개의 디렉토리(레이어)를 하나의 디렉토리처럼 보이게 합치는 리눅스 파일시스템**이다. 읽기 전용 레이어 위에 쓰기 가능한 레이어를 올려 **Copy-on-Write(CoW)** 방식으로 동작한다.

```
[컨테이너가 보는 파일시스템 — merged]
/bin/bash    ← lowerdir에서 읽음 (이미지 레이어)
/etc/hosts   ← upperdir에서 읽음 (컨테이너 실행 중 변경됨)
/app/code    ← lowerdir에서 읽음 (이미지 레이어)
    ↑
    OverlayFS가 upperdir + lowerdir를 합쳐서 보여줌

upperdir (컨테이너 쓰기 레이어) — 실행 중 변경사항 저장
    /etc/hosts         ← 컨테이너가 수정한 파일

lowerdir (이미지 레이어들 — 읽기 전용)
    /bin/bash          ← ubuntu base 이미지
    /app/code          ← 앱 이미지 레이어
```

파일을 수정할 때의 동작:

1. 컨테이너가 `/etc/nginx/nginx.conf` 수정 요청
2. OverlayFS가 `lowerdir`의 파일을 `upperdir`로 복사
3. `upperdir`의 복사본을 수정
4. 이후 읽기는 `upperdir` 버전을 반환

이를 **Copy-on-Write(CoW)** 라고 하며, 덕분에 여러 컨테이너가 동일한 이미지 레이어를 공유하면서도 서로 독립적인 파일시스템을 가질 수 있다.

#### K8S에서 overlay가 필요한 이유

`containerd`(K8S의 기본 컨테이너 런타임)는 컨테이너 이미지를 마운트할 때 OverlayFS를 사용한다. `overlay` 모듈이 없으면 containerd가 이미지를 마운트하지 못해 **Pod 실행 자체가 불가능**해진다.

```bash
# overlay 모듈 없을 때 containerd 에러 로그
ERRO[0000] failed to mount overlay: "overlay" not found
failed to create containerd task: failed to mount rootfs
```

---

### 4.3 `modprobe br_netfilter` — 브리지 네트워크 필터링 모듈

#### 4.3.1 Linux Bridge(브리지)란?

Linux Bridge는 **소프트웨어로 구현된 L2 네트워크 스위치**다. 여러 네트워크 인터페이스를 브리지에 연결하면, 이 인터페이스들은 같은 L2 네트워크(이더넷 세그먼트)에 속하게 된다.

```
# 브리지(cni0) 생성 및 확인
$ ip link show type bridge
4: cni0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1450 qdisc noqueue state UP mode DEFAULT
    link/ether 0a:58:0a:f4:00:01 brd ff:ff:ff:ff:ff:ff

# 브리지에 연결된 인터페이스 확인
$ bridge link show
5: vethXXXXXX@if3: <BROADCAST,MULTICAST,UP> mtu 1450 master cni0
6: vethYYYYYY@if3: <BROADCAST,MULTICAST,UP> mtu 1450 master cni0
```

K8S에서 브리지는 **CNI 플러그인이 Pod 네트워크를 구성할 때 핵심 역할**을 한다. 각 Pod는 `veth pair`(가상 이더넷 쌍)를 통해 브리지에 연결된다.

```
Pod A (네임스페이스 안)             호스트 네임스페이스
┌──────────────────┐            ┌──────────────────────────┐
│  eth0 (10.244.0.2)│←─ veth ──→│  vethXXXXXX              │
└──────────────────┘            │      │                   │
                                │    cni0 (bridge)          │
Pod B (네임스페이스 안)           │    10.244.0.1             │
┌──────────────────┐            │      │                   │
│  eth0 (10.244.0.3)│←─ veth ──→│  vethYYYYYY              │
└──────────────────┘            │      │                   │
                                │    eth0 (물리 NIC)        │
                                └──────────────────────────┘
```

#### 4.3.2 veth pair(가상 이더넷 쌍)란?

veth pair는 **항상 쌍으로 생성되는 가상 네트워크 인터페이스**다. 한쪽에 보낸 패킷은 반드시 반대쪽에서 수신된다. 파이프처럼 동작한다.

```
veth0 ←─────────────── veth1
        패킷을 veth0에    반대쪽 veth1에서
        보내면            패킷이 나옴
```

K8S(CNI)는 Pod를 생성할 때 아래 과정을 수행한다.

1. veth pair 생성 (vethXXXXXX, vethYYYYYY)
2. vethXXXXXX를 Pod의 네트워크 네임스페이스로 이동 → `eth0`으로 이름 변경
3. vethYYYYYY를 호스트의 브리지(cni0)에 연결

이렇게 하면 Pod 내부의 `eth0`과 호스트의 브리지가 연결되어, Pod가 브리지를 통해 다른 Pod나 외부와 통신할 수 있게 된다.

#### 4.3.3 핵심 문제 — 브리지 트래픽은 iptables를 우회한다

리눅스 커널에서 패킷 처리 경로는 계층별로 분리되어 있다.

- **L2 경로** : 브리지 내부에서 MAC 주소 기반으로 직접 스위칭
- **L3 경로** : IP 라우팅 → Netfilter(iptables) 체인 통과

같은 브리지에 연결된 Pod A → Pod B 패킷은 **L2에서 직접 스위칭**되기 때문에 기본적으로 iptables의 FORWARD 체인을 **통과하지 않는다**.

```
[br_netfilter 없을 때]
Pod A → vethXXXXXX → cni0 bridge → vethYYYYYY → Pod B
                        ↑
                    L2 스위칭으로 바로 전달
                    iptables FORWARD 체인을 거치지 않음!

[br_netfilter 있을 때]
Pod A → vethXXXXXX → cni0 bridge → iptables FORWARD 체인 → vethYYYYYY → Pod B
                                          ↑
                                  kube-proxy 규칙 적용됨
                                  Network Policy 적용됨
```

`br_netfilter` 모듈이 하는 일은 정확히 이 부분이다. **브리지를 통과하는 L2 패킷도 Netfilter(iptables) 훅을 거치도록 커널에 연결**한다.

---

### 4.4 sysctl 설정

`sysctl`은 **실행 중인 리눅스 커널의 파라미터를 읽거나 변경**하는 도구다. 설정값은 `/proc/sys/` 경로 아래에 파일로 존재하며, sysctl을 통해 런타임에 변경할 수 있다.

```bash
# 현재 값 확인
$ sysctl net.ipv4.ip_forward
net.ipv4.ip_forward = 0

# 런타임에서 즉시 변경
$ sudo sysctl -w net.ipv4.ip_forward=1

# 파일로 직접 변경도 동일한 효과
$ echo 1 | sudo tee /proc/sys/net/ipv4/ip_forward
```

영구 적용은 `/etc/sysctl.d/` 디렉토리의 `.conf` 파일에 작성하고 `sysctl --system`으로 적용한다.

```bash
$ cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

# /etc/sysctl.d/ 및 /etc/sysctl.conf 의 모든 설정 일괄 적용
$ sudo sysctl --system
```

---

#### 4.4.1 `net.bridge.bridge-nf-call-iptables = 1`

**이 파라미터가 하는 일**

`br_netfilter` 모듈을 로드했다고 해서 자동으로 브리지 트래픽이 iptables를 거치지 않는다. `bridge-nf-call-iptables=1`로 설정해야 실제로 **브리지를 통과하는 IPv4 패킷이 iptables의 체인을 호출**하도록 활성화된다.

```bash
# 대응하는 /proc 경로
/proc/sys/net/bridge/bridge-nf-call-iptables

# 0 : 브리지 트래픽이 iptables를 거치지 않음 (기본값)
# 1 : 브리지 트래픽도 iptables를 통과
```

**K8S에서 왜 반드시 필요한가?**

kube-proxy는 Kubernetes Service가 생성될 때마다 iptables의 `nat` 테이블에 규칙을 추가한다. 예를 들어 `ClusterIP: 10.96.0.10`인 Service에 접근하는 패킷을 실제 Pod IP인 `10.244.0.5`로 DNAT 변환하는 규칙이다.

브리지를 통과하는 Pod 간 트래픽이 iptables를 거치지 않으면:

1. Pod A가 Service IP(10.96.0.10)로 요청을 보냄
2. 패킷이 브리지에서 L2 스위칭됨
3. iptables의 DNAT 규칙을 거치지 않음
4. Service IP로 패킷이 전달되지만 실제 Pod가 없으므로 통신 실패

또한 Calico 등 CNI 플러그인의 **Network Policy**(Pod 간 트래픽 제어 규칙)도 iptables를 통해 구현된다. `bridge-nf-call-iptables=0`이면 같은 노드 안의 Pod 간 Network Policy가 무력화된다.

#### 4.4.2 `net.bridge.bridge-nf-call-ip6tables = 1`

`bridge-nf-call-iptables`의 IPv6 버전이다. K8S 클러스터에서 IPv6 또는 IPv4/IPv6 듀얼 스택을 사용하는 경우 브리지를 통과하는 IPv6 패킷도 ip6tables 체인을 거치도록 활성화한다.

IPv4만 사용하더라도 K8S 공식 설치 가이드에서 함께 설정하도록 권고한다.

#### 4.4.3 `net.ipv4.ip_forward = 1`

**IP Forwarding이란?**

기본적으로 리눅스 호스트는 **자신에게 온 패킷만 처리**한다. 목적지 IP가 자신의 IP가 아닌 패킷은 드롭(drop)한다. 이것이 호스트와 라우터의 근본적인 차이다.

`ip_forward=1`로 설정하면 리눅스가 **라우터처럼 동작**한다. 즉, 목적지가 자신이 아닌 패킷을 받았을 때 적절한 인터페이스로 **포워딩(전달)** 한다.

```bash
# 대응하는 /proc 경로
/proc/sys/net/ipv4/ip_forward

# 0 : 목적지가 자신이 아닌 패킷 드롭 (기본값)
# 1 : 다른 인터페이스로 포워딩 허용
```

**K8S에서 왜 반드시 필요한가?**

K8S 노드는 반드시 라우터처럼 패킷을 포워딩해야 한다. 두 가지 상황을 살펴본다.

**상황 1 — 같은 노드 내 Pod → 외부 통신**

```
Pod A (10.244.0.2)
  → 목적지: 8.8.8.8 (Google DNS)
  → cni0 브리지 (10.244.0.1)
  → 노드의 eth0 (192.168.1.10)
  → 인터넷

ip_forward=0 이면: eth0에서 받은 패킷의 출발지가 Pod IP(10.244.0.2)이므로
                   자신의 IP가 아님 → 드롭
ip_forward=1 이면: cni0 → eth0으로 정상 포워딩 후 인터넷 전달
```

**상황 2 — 다른 노드의 Pod와 통신**

```
Node 1의 Pod A (10.244.0.2) → Node 2의 Pod C (10.244.1.2)

Node 1:
  Pod A → veth → cni0 → eth0 (10.0.0.1)

Network:
  10.0.0.1 → 10.0.0.2 (패킷에는 목적지 IP가 10.244.1.2로 캡슐화됨)

Node 2:
  eth0 (10.0.0.2) 수신
  → 목적지 IP: 10.244.1.2 (자신의 IP가 아님)
  → ip_forward=0 이면 드롭
  → ip_forward=1 이면 cni0으로 포워딩 → Pod C 도달
```

### 4.6 각 설정이 없을 때 발생하는 문제 요약

| 설정 | 없을 때 발생하는 문제 |
|------|-------------------|
| `overlay` 모듈 없음 | containerd가 컨테이너 이미지 마운트 실패 → Pod 실행 불가 |
| `br_netfilter` 모듈 없음 | 브리지 트래픽이 iptables 우회 → Service 통신 불가, Network Policy 무력화 |
| `bridge-nf-call-iptables=0` | 모듈 로드했어도 브리지 트래픽이 iptables 미통과 → 동일 문제 |
| `ip_forward=0` | 노드가 패킷 포워딩 거부 → Pod의 외부 통신 불가, 노드 간 Pod 통신 불가 |