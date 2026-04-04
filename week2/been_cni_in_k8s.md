## 01. CNI?
- CNI (Container Network Interface)는 컨테이너 런타임(contianerd 등)과 네트워크 플러그인(Calico, Cilium 등) 사이의 표준 인터페이스다.
	- ==인터페이스라고 해서 어떠한 매개체를 뜻하는 것이 아니라, IT 통신에서의 프로토콜처럼 '컨테이너 런타임'과 '네트워크 플러그인' 간 통신 시 사용되는 규약/약속을 말한다. 이 때 주고받는 데이터의 형식은 JSON을 사용한다.==
- 쿠버네티스에 종속돼있지 않으며, 어떤 컨테이너 런타임에도 사용할 수 있는 공통 인터페이스로 설계됐다.
- 네트워크 인터페이스가 없는 컨테이너에 네트워크를 붙이는 역할을 한다.

### 01.01. CNI in K8s
- 쿠버네티스 자체는 네트워킹 구현체를 포함하지 않고, CNI 스펙만 정의한다.
	- CNI 스펙
		- CNI 표준에 따라 플러그인에 전달하는 약속된 형식의 JSON 파일
		- 환경변수
			- CNI 명령(ADD, CHECK, DEL)
			- 컨테이너 ID
			- 네트워크 네임스페이스 등
- 실질적인 네트워킹 구현체는 `/opt/cni/bin` 에 위치한 바이너리 파일들이 가지고 있고, 파드 생성 시 네트워크를 붙이는 역할을 수행한다.
- 그렇기 때문에 쿠버네티스는 네트워크 관련된 실질적인 워크로드를 수행하지 않고, 표준으로 정해진 형식에 따라 CNI 스펙만 정의해서 네트워크 구현체에 전달하면 된다.
- 이러한 형태 덕분에 CNI 'Plugin'이라고 부른다.

#### 01.01.01. 흐름
1. kubelet : 파드 생성 동작 수행
2. kubelet 이 containerd 호출
3. containerd : sandbox 컨테이너 및 netns 생성 후, netns 경로를 kubelet에게 반환
4. kubelet이 CNI 호출
	1. `/opt/cni/bin` 내의 CNI 바이너리를 exec 실행하며, 환경변수와 stdin JSON을 전달
5. CNI 바이너리 실행
	1. IP 할당 요청
	2. veth pair 생성
	3. pod 쪽 veth를 pause의 netns에 배치, IP 및 라우팅 설정
	4. host 쪽 veth를 attach
	5. 결과 JSON을 stdout으로 kubelet에게 반환
6. kubelet이 contianerd 호출
	1. 애플리케이션 컨테이너를 puase와 같은 netns에 join시켜 생성
7. 완료되면 pod를 running 상태로 업데이트

#### 01.01.02. JSON
```bash
# etc/cni/net.d/05-cilium.conflist 
{
  "cniVersion": "0.3.1",
  "name": "cilium",
  "plugins": [
    {
       "type": "cilium-cni",
       "enable-debug": false,
       "log-file": "/var/run/cilium/cilium-cni.log"
    }
  ]
}
```

- 환경변수 목록은 kubelet에 하드코딩

### Sandbox 컨테이너 Pause
- pause가 보호하는 것은 파드 내부의 컨테이너 크래시로부터 네트워크 네임스페이스이다. 이를 통해 파드 내에서 컨테이너가 재시작되더라도, 동일한 IP를 유지할 수 있다. 
	- 파드 재생성과는 별개이다.
```bash
# 테스트 파드 배포
kubectl run -n been test-pod --image=busybox --command -- sleep 3600

# 노드 확인
kubectl get po -n been -o wide

# 노드 이동 후 확인 (컨테이너)
crictl ps -a

# container id
crictl inspect <containerd-id> | grep pid

# sandbox container id
crictl inspectp <sandbox-id> | grep pid

# 동일 네임스페이스인지 확인
cd /etc/<container-pid>/ns/net
cd /etc/<pause-pid>/ns/net
```