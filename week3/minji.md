# Calico CNI 분석

## 목차
- [Calico란?](#calico란)
- [BGP](#bgp)
- [Overlay vs Underlay](#overlay-vs-underlay)
- [Policy Enforcement](#policy-enforcement)
- [Felix 내부 동작](#felix-내부-동작)


## Calico란?

Kubernetes에서 가장 많이 쓰이는 CNI 플러그인으로 **네트워크 연결 + 네트워크 정책(보안)** 을 모두 담당한다.

### 다른 CNI(Flannel 등)와의 차별점

| 특징 | 설명 |
|---|---|
| BGP 기반 순수 L3 라우팅 | 오버레이 없이 직접 라우팅 지원 |
| 강력한 NetworkPolicy 엔진 | GlobalNetworkPolicy 등 세밀한 제어 |
| 대규모 클러스터 성능 | Route Reflector로 확장성 확보 |


## BGP

원래 인터넷에서 AS(자율 시스템) 간 라우팅 경로를 교환하는 프로토콜로 Calico는 이를 클러스터 내부에 적용한다.

- 각 노드 = BGP 피어(speaker)
- Pod CIDR 정보를 노드끼리 서로 교환
- 모든 노드가 전체 라우팅 테이블 보유

### 동작 방식

```
Node1 (Pod: 10.0.1.0/24)
Node2 (Pod: 10.0.2.0/24)
Node3 (Pod: 10.0.3.0/24)

Node1 → Node2, Node3: "10.0.1.0/24는 나한테 있어" 광고
Node2 → Node1, Node3: "10.0.2.0/24는 나한테 있어" 광고

결과: 모든 노드가 전체 Pod 네트워크 경로 파악
     → Pod 간 통신 시 오버레이 없이 직접 라우팅
```

### BGP 모드 종류

| 모드 | 설명 | 적합한 환경 |
|---|---|---|
| Full Mesh | 모든 노드가 서로 BGP 피어링 | 소규모 클러스터 |
| Route Reflector | 특정 노드가 라우팅 정보 중계 | 대규모 클러스터 |

```
Full Mesh          Route Reflector

N1──N2──N3         N1    N3
 \  │  /             \  / \
  N4─N5               RR   N4
                     /  \
                   N2    N5
```


## Overlay vs Underlay

| 구분 | 설명 |
|---|---|
| **Underlay** | 물리적 네트워크 인프라 그 자체 (스위치, 라우터, 실제 NIC) |
| **Overlay** | Underlay 위에 가상으로 만든 네트워크, 패킷을 캡슐화해서 전송 |

### Calico 네트워크 모드

#### IPIP 모드 (Overlay)
Pod 패킷을 IP-in-IP로 캡슐화한다.

```
┌──────────────────────────────────┐
│ Outer IP  │ Inner IP  │  Data    │
│ (노드 IP) │ (Pod IP)  │          │
└──────────────────────────────────┘
```

- 어떤 네트워크 환경에서도 동작
- 노드 간 라우팅 설정 불필요
- 캡슐화 오버헤드로 성능 약간 저하

#### BGP Native 모드 (Underlay)
캡슐화 없이 Pod IP를 직접 라우팅한다.

```
┌──────────────────┐
│  IP    │  Data   │
│(Pod IP)│         │
└──────────────────┘
```

- 캡슐화 오버헤드 없음 → 성능 최고
- 네트워크 인프라가 Pod CIDR 라우팅을 지원해야 함
- 온프레미스, BGP 지원 라우터 환경에서 사용

#### VXLAN 모드 (Overlay)
BGP 없이 VXLAN으로 캡슐화한다.

- BGP 설정이 불가한 환경(퍼블릭 클라우드 등)에서 사용
- IPIP보다 호환성 높음

### 성능 비교

```
BGP Native > IPIP > VXLAN
```


## Policy Enforcement

Pod 간 트래픽을 허용/차단하는 방화벽 규칙으로 **iptables 또는 eBPF**로 커널 레벨에서 강제 적용한다.

### k8s 기본 NetworkPolicy 대비 장점

- 더 세밀한 규칙 설정 가능
- `GlobalNetworkPolicy` (네임스페이스 초월 적용)
- 노드 레벨 정책도 설정 가능

### 동작 위치

```
[Pod A] → [veth] → [iptables/eBPF] → [veth] → [Pod B]
                          ↑
               여기서 정책 적용 (허용 or 차단)
```

### Default Deny 정책

Calico 권장 보안 패턴: 기본적으로 모든 트래픽을 차단하고 필요한 것만 명시적으로 허용한다.

```yaml
apiVersion: projectcalico.org/v3
kind: GlobalNetworkPolicy
metadata:
  name: default-deny
spec:
  selector: all()
  types:
    - Ingress
    - Egress
```

### Policy 적용 예시

> order 숫자가 낮을수록 먼저 적용된다.

**시나리오**: frontend Pod → backend Pod 통신만 허용, 나머지는 차단

```yaml
apiVersion: projectcalico.org/v3
kind: NetworkPolicy
metadata:
  name: allow-frontend-to-backend
spec:
  selector: app == 'backend'
  ingress:
    - action: Allow
      source:
        selector: app == 'frontend'
    - action: Deny
```


# Felix — Calico 네트워크 정책 에이전트 내부 동작

## Calico의 보안이란?

Calico의 보안은 **네트워크 수준(L3/L4)의 접근 제어**다.  
"패킷이 어디서 어디로 갈 수 있는가"를 정책으로 강제하는 것이 핵심이다.


## Felix란?

Felix는 각 노드에서 실행되는 Calico의 핵심 에이전트로 **NetworkPolicy를 실제 커널 규칙으로 변환**하는 역할을 한다.

- Kubernetes API에서 `NetworkPolicy` 변경을 감지
- `iptables` / `nftables` / `eBPF` 규칙으로 변환하여 커널에 적용
- 노드 네트워크 상태를 data-store에 기록
- **DaemonSet**으로 배포 → 각 노드마다 1개씩 실행

### 전체 흐름

```
누가, 어디서 어디로, 어떤 프로토콜·포트로 통신할지 선언
    ↓
Felix가 데이터플레인(iptables / nftables / eBPF)에 ACL로 프로그래밍
    ↓
계산 그래프(calc)에서 protobuf 메시지 생성
    ↓
EventSequencer가 순서에 맞춰 전달
    ↓
데이터플레인 매니저가 iptables/eBPF에 반영
```


## 파일: `/felix/calc/active_rules_calculator.go`

### 1. 데이터 캐시 (line 64)

**"현재 이 노드에서 어떤 정책이 활성화됐는지"** 계산하기 위한 메모리 캐시.

```go
allPolicies     packedmap.Map[model.PolicyKey, *model.Policy]
allProfileRules packedmap.Deduped[string, *model.ProfileRules]
allTiers        map[string]*model.Tier
```

- data-store 스냅샷을 메모리에 유지
- 변경사항 수신 시 캐시만 갱신 → data-store에 재조회 불필요
- **전체 재계산 없이 델타만 처리** → 성능 최적화



### 2. 변경사항 수신 (line 146)

data-store에서 오는 변경 이벤트를 `switch`문으로 종류별로 처리한다.  
변경이 없으면 `DeepEqual`로 감지하여 처리를 스킵한다.

| 이벤트 종류 | 처리 방식 |
|---|---|
| `WorkloadEndpointKey` | 프로파일 재계산 + 정책 셀렉터 재매칭 |
| `PolicyKey` | 변경 없으면 스킵 → 캐시 갱신 → 재매칭 → dataplane 전달 |
| `ProfileRulesKey` | 변경 없으면 스킵 → 캐시 갱신 → 활성화된 것만 전달 |
| `TierKey` | 캐시만 갱신 |

```
datastore 변경
    ↓
OnUpdate() 호출
    ↓
switch로 종류 판별
    ↓
DeepEqual로 실제 변경인지 확인
    변경 없음 → return (스킵)
    변경 있음
        ↓
    캐시 갱신 (allPolicies 등)
        ↓
    labelIndex 재매칭
        ↓
    활성화된 것만 sendPolicyUpdate
        ↓
    dataplane 적용
```


### 3. 셀렉터 매칭 (line 121)

**label-index**: 정책 selector와 endpoint label을 실시간으로 매칭하는 인덱스.

```go
arc.labelIndex = labelindex.NewInheritIndex(arc.onMatchStarted, arc.onMatchStopped)
```

- `"app == frontend"` 같은 셀렉터가 어떤 endpoint에 매칭되는지 계산
- endpoint 또는 정책 셀렉터가 변경되면 자동으로 재매칭


### 4. 활성 정책 결정 (line 367)

정책이 **처음 활성화되는 순간에만** dataplane에 알린다. 매칭된 endpoint가 모두 사라지면 비활성화한다.

호출 시점: labelIndex가 셀렉터 ↔ endpoint 매칭 감지하면 onMatchStarted() 자동 호출
즉 "app == frontend" 셀렉터가 어떤 Pod에 매칭됐을 때 실행됨

```go
policyWasActive := arc.policyIDToEndpointKeys.ContainsKey(polKey)
arc.policyIDToEndpointKeys.Put(selID, labelId)

if !policyWasActive {
    // 처음 활성화된 경우에만 dataplane에 알림
    arc.sendPolicyUpdate(polKey, policy)
}
```

> **왜 `policyWasActive` 체크가 중요한가?**  
> endpoint 100개가 동일 정책에 매칭될 때, 1번째 endpoint 매칭 시에만 dataplane에 알리고  
> 이후 99개는 알리지 않는다 → 불필요한 dataplane 업데이트 방지

`PolicyMatchListener`는 처음 활성화 여부와 무관하게 **항상** 알림을 받는다.  
"어떤 endpoint에 어떤 정책이 매칭됐는지" 전부 추적해야 하기 때문이다.



### 5. 데이터플레인 전달 (line 468, 429)

계산된 활성/비활성 결과를 `RuleScanner`로 전달하고 이는 최종적으로 dataplane 드라이버로 연결된다.

```go
known  := policy != nil                                      // 정책 내용이 캐시에 있는가?
active := arc.policyIDToEndpointKeys.ContainsKey(policyKey) // 매칭된 endpoint가 있는가?
```

| 상태 | 동작 |
|---|---|
| `active && known` | 활성화 → dataplane에 정책 규칙 적용 |
| `active && !known` | 발생 불가 → Panic |
| `!active` | 비활성화 → dataplane에서 정책 규칙 제거 |

```go
if active {
    arc.RuleScanner.OnPolicyActive(policyKey, policy)   // dataplane에 iptables/eBPF 규칙 생성
} else {
    arc.RuleScanner.OnPolicyInactive(policyKey)         // dataplane에서 iptables/eBPF 규칙 삭제
}
```

#### 프로파일이 없을 때 — 자동 Deny All (line 429)

```go
if active {
    if !known {
        ...
        rules = &DummyDropRules  // InboundRules: deny, OutboundRules: deny
    }
    arc.RuleScanner.OnProfileActive(key, rules)
}
```

> 프로파일이 없는 endpoint는 어떤 트래픽을 허용할지 알 수 없다  
> → **안전하게 전부 차단**, 명시적 허용이 있을 때만 통과

| 함수 | 역할 |
|---|---|
| `sendPolicyUpdate` (468) | 정책 활성/비활성을 RuleScanner에 전달 |
| `sendProfileUpdate` (429) | 프로파일 활성/비활성 전달, 없으면 자동 deny all 적용 |


### 6. 프로파일 ID 델타 계산 (line 520)

endpoint의 profile 목록이 바뀔 때, **전체 재계산 없이 추가·제거된 것만 계산**한다.

**알고리즘:**
1. 기존 ID를 전부 `removedIDs`에 넣는다 (일단 제거됐다고 가정)
2. 새 ID 목록을 순회하며:
   - 기존에도 있으면 → `removedIDs`에서 삭제 (유지)
   - 기존에 없으면 → `addedIDs`에 추가 (신규)

```
변경 전: [A, B, C]
변경 후: [B, C, D]

1단계: removedIDs = {A, B, C}
2단계 순회 (B, C, D):
  B → removedIDs에 있음 → 제거 취소
  C → removedIDs에 있음 → 제거 취소
  D → removedIDs에 없음 → addedIDs에 추가

최종:
  removedIDs = {A}     ← 제거된 것
  addedIDs   = {D}     ← 추가된 것
  유지        = {B, C} ← 아무것도 안 해도 됨
```


### 7. Force Programmed 정책 처리 (line 306)

**일반 정책**: 셀렉터가 endpoint에 매칭돼야 활성화됨, 매칭된 endpoint가 없으면 비활성.  
**Force Programmed 정책**: 매칭되는 endpoint가 없어도 **모든 노드에 강제로 적용**.

클러스터 전체 보안 정책 등에 사용한다.

```go
func policyForceProgrammed(policy *model.Policy) bool {
	if policy == nil {
		return false
	}
	return slices.Contains(policy.PerformanceHints, 
	v3.PerfHintAssumeNeededOnEveryNode) // Force Programmed
}
```

```yaml
# 정책 yaml에서 설정
metadata:
  annotations:
    projectcalico.org/assume-needed-on-every-node: "true"
```

#### 구현 방식 — 더미 키 시뮬레이션

실제 endpoint 대신 **가짜(dummy) endpoint 키**를 삽입하여 항상 매칭된 것처럼 시뮬레이션한다.

```go
const forceProgrammedDummyKey = "PolicyAlwaysProgrammed"

// Force Programmed로 전환될 때
if !oldPolicyWasForceProgrammed && newPolicyForceProgrammed {
    arc.onMatchStarted(key, forceProgrammedDummyKey)  // 더미 키 삽입
}

// Force Programmed 해제될 때
if oldPolicyWasForceProgrammed && !newPolicyForceProgrammed {
    arc.onMatchStopped(key, forceProgrammedDummyKey)  // 더미 키 제거
}
```

#### 상태 전환 시나리오

**일반 → Force Programmed:**
```
정책 업데이트 수신
    ↓
oldPolicyWasForceProgrammed = false
newPolicyForceProgrammed    = true
    ↓
onMatchStarted(key, dummyKey) 호출
    ↓
더미 키 추가됨
    ↓
endpoint 없어도 활성 상태 유지
```

**Force Programmed → 일반:**
```
정책 업데이트 수신
    ↓
labelIndex.UpdateSelector() 먼저 실행 (실제 endpoint 매칭 확보)
    ↓
onMatchStopped(key, dummyKey) 호출 (더미 키 제거)
    ↓
실제 매칭된 endpoint 있으면 활성 유지, 없으면 비활성화
```


## 참고

- [Calico 공식 문서](https://docs.tigera.io/calico/latest/about)
- [Felix GitHub](https://github.com/projectcalico/calico/tree/master/felix)
- [BGP in Calico](https://docs.tigera.io/calico/latest/networking/bgp)
