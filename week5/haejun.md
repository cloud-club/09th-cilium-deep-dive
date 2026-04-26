## 0. 이번 주 포지션

이번 문서는 "기능 소개"가 아니라 "어디서 깨지고 어떻게 추적할지"를 중심으로 정리한다.

DeepWiki 2.1~3.6을 기준으로 운영에서 실제로 의미 있는 지점을 남긴다.

참고: https://deepwiki.com/cilium/cilium

---

## 1. Control Plane Deep-Dive

Cilium Agent를 한 문장으로 요약하면:

**Kubernetes 상태를 받아 endpoint/policy/identity/IPAM/LB 상태를 계산하고, 그 결과를 datapath가 소비 가능한 형태로 동기화하는 조정기**

### 1.1 설정 시스템: "값을 어디서 넣었는가"보다 "최종값이 무엇인가"

표면적으로는 Helm values, ConfigMap, 환경 변수, CLI flag를 쓴다.
실제로 중요한 건 우선순위 충돌 시 최종 런타임 값이다.

핵심 관찰 포인트:
- 우선순위: CLI > Env > ConfigMap/Config file > Default
- 업그레이드 호환성: upgradeCompatibility가 기본값 변화 충격을 완화
- 초기 검증기: 잘못된 고정 ID, zone, map event buffer 문자열을 시작 전에 차단

운영에서 자주 깨지는 지점:
- Helm 값 바꿨는데 Env override가 남아 있어 반영 안 된 것처럼 보이는 경우
- 버전 업 후 기본값 변경을 예상 못해 node behavior가 달라지는 경우

실전 체크:
1. 변경 후 "실제 런타임 config dump"로 최종값 확인
2. 값 소스(Helm/Env/Flag) 추적 로그를 함께 저장

### 1.2 시작 순서: 순서가 맞아야 일관성이 맞는다

Agent 시작은 단순 부팅이 아니라 순서 의존 작업이다.

**Phase 1: Configuration Validation**
- 설정 충돌 감지 (IPsec vs WireGuard, kube-proxy replacement 조건 등)

**Phase 2: Main Daemon Initialization**
- CRD 동기화 대기
- IPAM 준비 (IPAM mode가 cluster pool이면 CiliumNode 리소스 생성)
- Device detection (routing device 등 네트워크 인터페이스 감지)
- K8s Watchers 시작 (Pod, Service, Policy 감시자)

**Phase 3: Endpoint Restoration**
- 디스크에서 endpoint 상태 복원
- 기존 Pod 연결성 보장

**Phase 4: Node Discovery and IPAM**
- NodeDiscovery.StartDiscovery() 호출로 로컬 노드 등록
- IPAM 시작으로 host/health/ingress IP 할당
- background job들 시작 (watcher 루프 등)

이 순서를 어기면 발생하는 증상:
- endpoint 상태 미복원 상태에서 policy 계산이 먼저 일어나 revision mismatch
- CRD 준비 전 watcher가 돌면서 초기 이벤트 누락/지연
- node 정보가 비어 있는 상태에서 identity/policy 문맥이 흔들림

shutdown도 같은 수준으로 중요하다:
- DNS policy unload, endpoint 정리, health responder 종료가 순서대로 끝나야 재기동 시 흔적(stale state)이 줄어든다.

### 1.3 Hive 모듈성: "구조 미학"이 아니라 장애 반경 제어 도구

[hive git](https://github.com/cilium/hive)

#### Hive의 근본 목표

Cilium agent는 거대한 Go 바이너리다. 하지만 내부적으로는 독립적인 기능들(endpoint manager, policy engine, identity backend, IPAM, datapath loader 등)이 뒤엉켜 있다.

**Hive는 이 뒤엉김을 의존성 그래프로 명시화하고, 각 기능의 생명주기를 일관되게 제어하기 위한 프레임워크다.**

구체적으로:
1. **Cell**: 각 기능을 독립 모듈로 정의 (endpoint cell, policy cell, identity cell 등)
2. **의존성 선언**: 어떤 cell이 다른 cell을 필요로 하는지 코드로 명시
3. **OnStart/OnStop hooks**: 각 cell의 시작과 종료 단계를 제어
4. **자동 순서 계산**: 의존성 그래프를 기반으로 초기화 순서를 자동으로 결정

#### 왜 운영에서 중요한가

- **장애 범위 제한**: 특정 cell이 비정상일 때 영향받는 것이 정확히 무엇인지 알 수 있다.
- **디버깅 효율**: "agent가 느려졌다" → "어느 cell의 OnStart에서 지연됐는가"로 빠르게 좁힐 수 있다.
- **테스트 격리**: 테스트 시 일부 cell만 mock해서 나머지 로직을 분리 검증 가능하다.
- **업그레이드 안정성**: 시작 순서가 암묵적 관습이 아니라 그래프로 고정되어, 버전 간 순서 변화로 인한 문제를 방지한다.

#### 실무 추적 포인트

- 장애 시 log에서 "cell X started", "cell Y failed on start"를 먼저 찾는다.
- 기능 토글 시 side-effect를 전사 검색하지 말고, 해당 cell의 의존 경계부터 확인한다.
- 높은 restart 주기나 intermittent failure는 cell 간 순서 의존성 위반일 가능성이 크다.

### 1.4 Endpoint/Policy/Identity: 세 개를 같이 봐야 원인이 보인다

#### Endpoint Management
- endpoint는 단순 객체가 아니라 lifecycle + policy revision + datapath regeneration 상태를 가진다.
- 복원 과정에서 disk 상태, Kubernetes 상태, 현재 identity 상태가 동시에 맞아야 한다.

깨지는 패턴:
- 복원은 되었는데 policy revision sync가 늦어 Ready 전환이 지연
- regeneration queue가 밀리며 policy 반영이 늦게 보이는 현상

#### Policy System
- rule 저장소(Repository) -> selector resolution -> distillation -> map state 반영
- 핵심은 "selector 기반 규칙"을 "숫자 identity 기반 key"로 압축하는 단계

깨지는 패턴:
- selector cache 갱신 지연 시 정책이 늦게 반영된 것처럼 보임
- deny/allow 우선순위 착각으로 기대와 다른 verdict 발생

#### Identity Management
- 정책 기준은 IP가 아니라 numeric identity
- backend는 kvstore/crd/double-write, 모드는 agent/operator/both
- reserved identity(host/world/health/remote-node)는 디버깅 기준점

깨지는 패턴:
- identity GC 타이밍과 endpoint churn이 겹쳐 stale 판단이 어려워짐
- 멀티클러스터에서 cluster ID 비트 문맥을 놓쳐 identity 충돌처럼 보이는 오해

### 1.5 운영 포인트

#### IPAM
- 단순 IP 할당이 아니라 routing/MTU/infra IP(health, ingress, router)와 결합된 기능
- multi-pool은 pool selection 규칙(annotation/selector/default) 이해가 핵심

#### Service LB
- socket LB + TC/XDP LB + BPFReconciler가 함께 작동
- Maglev, affinity, DSR은 "성능"뿐 아니라 "디버깅 난이도"를 같이 바꾼다

#### ClusterMesh
- API Server + KVStore + remote identity sync 조합
- 장애 분석 시 "서비스 발견" 문제와 "identity 동기화" 문제를 분리해서 봐야 함

#### BGP
- 연결 자체보다 운영 조건(노드 선택, ASN 일치, secret 접근성)에서 자주 실패
- 상태 확인은 peers/routes/conditions를 세트로 확인

#### Mutual Auth + SPIRE
- 암호화와 다르게 "누구인지 검증"이 목표
- auth map miss -> signal -> handshake -> map update 흐름이 끊기면 정상 트래픽도 drop

---

## 2. Datapath Deep-Dive

### 2.1 Config-to-BPF 컴파일 파이프라인

#### 온보드 컴파일 vs 사전 빌드

핵심은 런타임 분기 최소화다:
- option.Config -> HeaderfileWriter -> node_config.h -> clang/LLVM -> bpf object
- ENABLE_IPV4/ENABLE_NODEPORT 같은 define이 코드 경로 자체를 제거하거나 활성화

하지만 현실 문제: 각 노드의 커널 버전, BTF 지원, bpftool 가용성이 다르다.

**Cilium의 해결책:**

1. **Portable eBPF (CO-RE 기반)**
   - libbpf CO-RE를 사용해 컴파일 시점의 struct layout을 런타임에 자동 조정
   - vmlinux.h를 미리 생성하거나, 런타임에 커널 BTF에서 추출
   - 덕분에 한 번 빌드한 object가 여러 커널에서 동작 가능

2. **커널 feature detection**
   - agent 시작 시 `BPF_PROG_TEST_RUN`으로 필요 feature 확인
   - 지원하지 않는 기능의 BPF 부분은 로드 단계에서 제외 (pruning)
   - 예: `memcg_acct` 없으면 해당 코드 경로 비활성화

3. **Pre-built object 선택**
   - DaemonSet으로 배포 시 여러 환경용 object를 포함
   - 노드에서 실행 시 "이 커널은 어떤 기능 지원하는가"를 판단해 적절한 object 로드
   - 캐싱: object_cache에서 동일 config라면 재컴파일 스킵

운영 의미:
- "옵션만 바꿨다"가 아니라 "컴파일 결과가 바뀐다"
- 같은 정책이어도 빌드 산출물 차이가 behavior 차이로 나타날 수 있다
- 하지만 설치/업그레이드 시 "모든 노드가 완벽히 같은 커널"이아니어도 각 노드가 자신에게 맞는 object를 선택할 수 있다

예시: EnableIPv6 설정이 false면 IPv6 코드 경로 전체가 object에서 제거됨

``` c
// bpf/bpf_host.c
#ifdef ENABLE_IPV6
static __always_inline int
handle_ipv6(struct __ctx_buff *ctx, ...) {
    // IPv6 logic
}
#endif
```

**체크 포인트:**
1. node_config.h diff (설정 차이 확인)
2. object cache hit/miss (컴파일 재실행 여부)
3. 각 노드의 커널 feature support log (어떤 object가 로드됐는지)
4. attach 대상 인터페이스별 실제 로딩 결과

### 2.2 BPF Map lifecycle: map은 캐시가 아니라 상태 원본

중요 map:
- ipcache, policy, ct, nat, lb 계열

DeepWiki 3.3에서 실무적으로 특히 중요한 것:
- reachability 기반 pruning: 안 쓰는 map/tailcall을 로딩 단계에서 제외
- map registry + pinning 재사용: agent restart 시 상태 연속성 유지
- alignchecker: Go/C struct mismatch를 사전에 차단

운영 리스크:
- map 크기 과소 설정 시 eviction/retry/지연이 누적되어 "가끔 안 됨" 증상 발생
- struct 정렬 불일치가 있으면 증상이 랜덤 드롭처럼 보일 수 있음

### 2.3 CT/NAT: 성능/정합성/디버깅 난이도의 균형점

CT와 NAT는 분리 기능처럼 보이지만 실제로는 강하게 결합돼 있다.

핵심 포인트:
- CT GC 주기는 stale state와 CPU 비용 사이의 트레이드오프
- revNAT index 정합성이 깨지면 reply path가 이상해진다
- CT 삭제와 NAT 삭제의 타이밍 불일치가 잔존 매핑 문제를 만든다

운영에서 보이는 전형적 증상:
- 신규 연결은 되는데 일부 응답이 비정상 경로로 가는 현상
- 부하가 높을수록 재현이 어려운 간헐 문제

### 2.4 Encryption (IPsec/WireGuard)

- IPsec: XFRM state/policy + key rotation 안정성이 핵심
- WireGuard: peer/allowed IP 갱신 누락 시 특정 노드 경로만 실패

실전에서 중요한 질문:
1. 키가 없어서 실패인가, 경로 정책이 어긋나서 실패인가
2. strict mode가 의도된 drop인가, 설정 누락인가

### 2.5 Observability and Metrics: "많이 본다"보다 "필요한 만큼 본다"

3.6의 핵심은 이벤트 생성-집계-파싱 파이프라인의 비용 관리다.

구성:
- kernel trace/drop notify -> events map -> userspace monitor/hubble parser
- aggregation level + rate limit으로 이벤트 폭주 억제
- ipcache metadata로 flow 문맥(누가 누구인지) 보강

운영 리스크:
- 관측을 과도하게 켜면 문제 찾기 전에 시스템 비용이 먼저 올라간다
- aggregation이 너무 강하면 중요한 단서가 사라진다

권장 접근:
- 평시 저비용, 장애 시 증분 확장 방식으로 observability 레벨 운영

---


### 3.0 폴더 구조 설명

* `daemon/`: Cilium Agent
* `operator/`: Cilium Operator
* `bpf/`: eBPF datapath 코드
* `pkg/`: 공용 Go 라이브러리와 핵심 상태 관리 코드
* `install/kubernetes/`: Helm chart와 배포 리소스
* `images/`: Dockerfile과 이미지 빌드 정의
* `cilium-cli/`, `cilium-dbg/`, `cilium-health/`: 사용자용 CLI와 진단 도구
* `hubble/`, `hubble-relay/`: 관측 관련 컴포넌트
* `clustermesh-apiserver/`: 멀티 클러스터 연동
* `test/`, `.github/`: 테스트와 CI


핵심은 "control plane 계산"과 "datapath 집행"을 한 번에 보지 말고,
동기화 경계(map/header/load 단계)에서 먼저 끊어 보는 것이다.