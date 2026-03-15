# Kubernetes Networking Deep Dive Study

> **쿠버네티스 네트워크가 커널에서 실제로 어떻게 동작하는지 이해하고  
eBPF 기반 CNI인 Cilium의 datapath를 코드 레벨까지 분석하는 8주 스터디**
---
# 🧑‍💻 Study Workflow

각 주차마다 다음 방식으로 진행합니다.

1. 주차 주제에 맞는 내용을 **자유롭게 학습**
2. 학습 내용을 정리하여 **GitHub에 업로드**
3. 스터디 시간에 **발표 및 토론**
---
# 📚 Study Materials

자료는 제한하지 않습니다.

가능한 다양한 자료를 활용합니다.

- Official documentation
- Technical blogs
- Linux kernel source
- Papers
- Books
- Conference talks

각 주차 주제에 맞는 자료를 **자유롭게 탐색 후 공유**합니다.
---
# 📂 Repository Structure

```
study-repo
 ├─ week1
 │   ├─ member1.md
 │   ├─ member2.md
 │
 ├─ week2
 │   ├─ member1.md
 │   ├─ member2.md
 │
 └─ README.md
```
---
# 🚀 How to Contribute

1. 본인 이름으로 **branch를 생성**합니다.

```bash
git checkout -b haejun
```

2. 해당 주차 폴더에 학습 내용을 정리합니다.

예시

```
week1/
  haejun.md
```

3. commit 후 push 합니다.

```bash
git add .
git commit -m "week1: linux network study"
git push origin haejun
```

4. **Pull Request(PR)** 를 생성합니다.
