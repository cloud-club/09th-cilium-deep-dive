# 📝 Week 2 Submission

## 📌 주제  
**Kubernetes Networking**

---

## 📚 학습 범위

이번 주차에서는 Kubernetes에서의 네트워크 동작을 전반적으로 살펴봅니다.

- **CNI 구조**
  - CNI plugin의 역할
  - Pod 네트워크가 구성되는 흐름 이해

- **kube-proxy**
  - iptables vs IPVS 방식 비교
  - Service 트래픽 처리 방식

- **Service → Pod 흐름 추적**
  - ClusterIP 접근 시 패킷 흐름
  - kube-proxy가 개입하는 지점 이해

---

## 🎯 학습 목표

- Kubernetes에서 Pod 간 통신이 어떻게 이루어지는지 이해합니다.
- Service가 실제 Pod로 트래픽을 전달하는 과정을 설명할 수 있습니다.
- kube-proxy의 동작 방식(iptables / IPVS)의 차이를 이해합니다.

---

## 📦 제출 방식

- 본인 이름 branch 생성  
- `week2` 폴더에 정리 파일 업로드  
- PR 생성  

---

## 💡 참고

이번 주는 "네트워크 흐름을 그림으로 그릴 수 있는 수준"까지 이해하는 것이 목표입니다.
