---
layout: post
title:  "사설 클라우드 개발 관리 서버 구축 가이드 (Gitea/Wiki.js 설치 및 클라우드 구성)"
date:   2026-04-17 01:00:00 +0900
---

# 📘 Phase 2 — VM 1 개발 관리 서버 구축 가이드
## Gitea (소스 관리) + Wiki.js (기술 문서) 통합 구축

> **Proxmox VE 9.1 · Ubuntu 22.04 · k3s · Gitea · Wiki.js**

| 항목 | 내용 |
|------|------|
| **버전** | v1.1 (2026-04-20 업데이트) |
| **대상 VM** | VM 100 — `vm-dev` (<VM1_IP>) |
| **사전 조건** | Phase 1 완료 — Ubuntu 22.04 + k3s 설치 및 `dev-tools` 네임스페이스 생성 |
| **예상 소요 시간** | 2 ~ 4 시간 |
| **난이도** | ⭐⭐☆☆☆ (초보자 가능) |

> 📝 **v1.1 변경사항**
> - Gitea 조직(`infra-lab`) 개념 및 생성 절차 추가
> - Wiki.js 동기화 시 `ko/` 디렉토리가 생성되지 않는 실제 동작 반영
> - Gitea 저장소 경로를 조직 기준(`infra-lab/wiki-storage`)으로 전면 수정

---

## 📋 목차

1. [이 가이드에서 만들 것](#1-이-가이드에서-만들-것)
2. [사전 조건 확인](#2-사전-조건-확인)
3. [작업 디렉토리 구성](#3-작업-디렉토리-구성)
4. [Step 1 — Gitea 배포 (Pod A)](#4-step-1--gitea-배포-pod-a)
5. [Step 2 — Wiki.js 배포 (Pod B)](#5-step-2--wikijs-배포-pod-b)
6. [Step 3 — Gitea 초기 설정 및 조직 생성](#6-step-3--gitea-초기-설정-및-조직-생성)
7. [Step 4 — Wiki.js 초기 설정 및 한글화](#7-step-4--wikijs-초기-설정-및-한글화)
8. [Step 5 — Wiki.js ↔ Gitea 저장소 연동 (핵심)](#8-step-5--wikijs--gitea-저장소-연동-핵심)
9. [Step 6 — 첫 번째 문서 home.md 생성](#9-step-6--첫-번째-문서-homemd-생성)
10. [Step 7 — 폴더 계층 구조 생성 실습](#10-step-7--폴더-계층-구조-생성-실습)
11. [Step 8 — Gitea 동기화 검증](#11-step-8--gitea-동기화-검증)
12. [트러블슈팅 가이드](#12-트러블슈팅-가이드)
13. [Phase 2 완료 체크리스트](#13-phase-2-완료-체크리스트)
14. [다음 단계](#14-다음-단계)

---

## 1. 이 가이드에서 만들 것

### 최종 완성 구조

```
VM 1 (vm-dev, <VM1_IP>)
│
├─── k3s Namespace: dev-tools
│     │
│     ├── Pod A: Gitea        → http://<VM1_IP>:30000
│     │         (마크다운 .md 파일이 실제로 저장되는 Git 서버)
│     │
│     └── Pod B: Wiki.js      → http://<VM1_IP>:30005
│               (웹 브라우저에서 문서를 작성·편집하는 인터페이스)
│
└─── Gitea 조직: infra-lab
      └── 저장소: wiki-storage
            ├── home.md                  ← Wiki.js 메인 페이지
            └── dev/
                 └── golang/
                      └── architecture.md  ← 폴더 계층 문서
```

> ⚠️ **중요: `ko/` 디렉토리는 생성되지 않습니다**
>
> Wiki.js에서 한국어(`ko`) 언어를 설정해도, Gitea에 동기화될 때는 `ko/` 상위 폴더 없이
> 입력한 경로 그대로 저장됩니다.
>
> ```
> Wiki.js 경로 입력: dev/golang/architecture
>                     ↓ 실제 Gitea 저장 경로
> Gitea 파일 위치:   dev/golang/architecture.md   ← ko/ 없음!
> ```

### Gitea 조직(Organization) vs 개인 계정 이해

이 가이드에서는 **개인 계정**이 아닌 **조직(Organization) 계정** 아래에 저장소를 만듭니다.

| 구분 | 개인 계정 | 조직 (이 가이드) |
|------|-----------|-----------------|
| URL 형식 | `/rt/wiki-storage` | `/infra-lab/wiki-storage` |
| 용도 | 개인용 저장소 | 팀/프로젝트 단위 저장소 그룹 |
| 권장 이유 | - | 나중에 팀원 추가·저장소 분류가 쉬움 |

**실제 확인된 Gitea 조직 구조:**
```
Gitea 조직: infra-lab
  ├── wiki-storage   ← Wiki.js 문서 저장소 (이 가이드에서 생성)
  └── k3s-iac        ← k3s YAML 매니페스트 저장소 (별도 생성)
```

### 완성 화면 미리보기

| 서비스 | 접속 주소 | 역할 |
|--------|-----------|------|
| Gitea | `http://<VM1_IP>:30000` | .md 파일 물리 저장소 |
| Wiki.js | `http://<VM1_IP>:30005` | 문서 웹 편집기 |

---

## 2. 사전 조건 확인

> ⚠️ **이 단계를 건너뛰면 이후 모든 작업이 실패할 수 있습니다.** 반드시 확인 후 진행하세요.

### 2-1. VM 1 접속

```bash
# Windows에서 SSH 접속
ssh rt@<VM1_IP>
```

### 2-2. k3s 동작 확인

```bash
kubectl get nodes
```

**정상 출력 예시:**
```
NAME     STATUS   ROLES                  AGE   VERSION
vm-dev   Ready    control-plane,master   5d    v1.29.x+k3s1
```

> ❌ `NotReady` 또는 명령어 오류 → `sudo systemctl restart k3s` 실행 후 재확인

### 2-3. 네임스페이스 확인 및 생성

```bash
# 네임스페이스 목록 확인
kubectl get namespaces
```

`dev-tools`가 없으면 생성:

```bash
kubectl create namespace dev-tools
```

**확인:**
```bash
kubectl get namespaces | grep dev-tools
# 출력: dev-tools   Active   ...
```

### 2-4. 방화벽 포트 개방

```bash
sudo ufw allow 30000/tcp   # Gitea
sudo ufw allow 30005/tcp   # Wiki.js
sudo ufw status            # 개방 포트 확인
```

---

## 3. 작업 디렉토리 구성

YAML 파일들을 서비스별로 분리해서 관리합니다.

```bash
mkdir -p ~/gitea
mkdir -p ~/wikijs

ls ~
# 출력: gitea  wikijs  ...
```

---

## 4. Step 1 — Gitea 배포 (Pod A)

Gitea는 마크다운 파일이 실제로 저장되는 **물리 저장소**입니다. GitHub의 셀프호스팅 버전이라고 생각하면 됩니다.

### 4-1. Gitea PVC YAML 작성

```bash
cd ~/gitea
nano gitea-pvc.yaml
```

아래 내용을 붙여넣고 저장 (`Ctrl+X` → `Y` → `Enter`):

```yaml
# gitea-pvc.yaml
# Gitea 데이터를 영구 저장하기 위한 볼륨
# 컨테이너가 재시작되거나 재배포되어도 데이터가 삭제되지 않음
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: gitea-pvc
  namespace: dev-tools
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 20Gi     # Gitea 저장소 + 위키 문서 용량
```

### 4-2. Gitea Deployment + Service YAML 작성

```bash
nano gitea-deployment.yaml
```

아래 내용을 붙여넣고 저장:

```yaml
# gitea-deployment.yaml
# Gitea 서버 Pod 배포 + 외부 접근 Service 설정
apiVersion: apps/v1
kind: Deployment
metadata:
  name: gitea
  namespace: dev-tools
  labels:
    app: gitea
spec:
  replicas: 1
  selector:
    matchLabels:
      app: gitea
  template:
    metadata:
      labels:
        app: gitea
    spec:
      containers:
      - name: gitea
        image: gitea/gitea:latest
        env:
        - name: USER_UID
          value: "1000"
        - name: USER_GID
          value: "1000"
        ports:
        - containerPort: 3000    # Gitea 웹 UI 포트
        - containerPort: 22      # Git SSH 포트
        volumeMounts:
        - name: gitea-storage
          mountPath: /data        # Gitea 모든 데이터 저장 경로
      volumes:
      - name: gitea-storage
        persistentVolumeClaim:
          claimName: gitea-pvc   # 위에서 만든 PVC 연결
---
# NodePort Service: VM 외부(브라우저)에서 접근 가능하게 노출
apiVersion: v1
kind: Service
metadata:
  name: gitea
  namespace: dev-tools
spec:
  type: NodePort
  selector:
    app: gitea
  ports:
  - name: web
    port: 3000
    targetPort: 3000
    nodePort: 30000       # 브라우저 접속: http://<VM1_IP>:30000
  - name: ssh
    port: 22
    targetPort: 22
    nodePort: 30022       # git clone/push: ssh://<VM1_IP>:30022
```

### 4-3. Gitea 배포 실행

```bash
# 1. PVC 먼저 생성 (저장 공간 확보)
kubectl apply -f gitea-pvc.yaml

# 2. Deployment + Service 배포
kubectl apply -f gitea-deployment.yaml
```

### 4-4. Gitea Pod 동작 확인

```bash
# Pod 상태 확인 (Running이 될 때까지 대기)
kubectl get pods -n dev-tools

# 실시간 상태 모니터링 (Running 확인 후 Ctrl+C)
kubectl get pods -n dev-tools -w
```

**정상 출력 예시:**
```
NAME                     READY   STATUS    RESTARTS   AGE
gitea-7d9f8b6c4-xk2p9   1/1     Running   0          45s
```

> 💡 `ContainerCreating` → `Running` 으로 바뀌는 데 1~2분 소요됩니다.

```bash
# 서비스 포트 확인
kubectl get svc -n dev-tools
# 출력: gitea   NodePort   ...   3000:30000/TCP,22:30022/TCP
```

**브라우저 접속 테스트:** `http://<VM1_IP>:30000`

> ✅ Gitea 초기 설정 화면이 보이면 성공! (설정은 Step 3에서 진행)

---

## 5. Step 2 — Wiki.js 배포 (Pod B)

Wiki.js는 웹에서 문서를 작성하는 **편집 인터페이스**입니다. 작성한 내용을 Gitea에 자동으로 저장합니다.  
Wiki.js는 데이터 저장을 위해 **PostgreSQL 데이터베이스**가 필요합니다.

### 5-1. PostgreSQL PVC YAML 작성

```bash
cd ~/wikijs
nano postgres-pvc.yaml
```

아래 내용 입력 후 저장:

```yaml
# postgres-pvc.yaml
# Wiki.js가 사용하는 PostgreSQL 데이터베이스 영구 볼륨
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: postgres-pvc
  namespace: dev-tools
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 10Gi
```

### 5-2. PostgreSQL Deployment + Service YAML 작성

```bash
nano postgres-deployment.yaml
```

아래 내용 입력 후 저장:

```yaml
# postgres-deployment.yaml
# Wiki.js 전용 PostgreSQL 데이터베이스 Pod
apiVersion: apps/v1
kind: Deployment
metadata:
  name: postgres
  namespace: dev-tools
spec:
  replicas: 1
  selector:
    matchLabels:
      app: postgres
  template:
    metadata:
      labels:
        app: postgres
    spec:
      containers:
      - name: postgres
        image: postgres:15-alpine
        env:
        - name: POSTGRES_DB
          value: "wiki"
        - name: POSTGRES_USER
          value: "wikijs"
        - name: POSTGRES_PASSWORD
          value: "wikijs_password"  # ⚠️ 운영 환경에서는 강력한 비밀번호로 변경!
        ports:
        - containerPort: 5432
        volumeMounts:
        - name: db-storage
          mountPath: /var/lib/postgresql/data
      volumes:
      - name: db-storage
        persistentVolumeClaim:
          claimName: postgres-pvc
---
# ClusterIP Service: k3s 내부 네트워크에서만 접근 가능 (외부 노출 없음)
# Wiki.js Pod에서 서비스명 "postgres"로 직접 통신
apiVersion: v1
kind: Service
metadata:
  name: postgres
  namespace: dev-tools
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

### 5-3. Wiki.js Deployment + Service YAML 작성

```bash
nano wikijs-deployment.yaml
```

아래 내용 입력 후 저장:

```yaml
# wikijs-deployment.yaml
# Wiki.js 웹 애플리케이션 Pod + 외부 접근 Service
apiVersion: apps/v1
kind: Deployment
metadata:
  name: wikijs
  namespace: dev-tools
  labels:
    app: wikijs
spec:
  replicas: 1
  selector:
    matchLabels:
      app: wikijs
  template:
    metadata:
      labels:
        app: wikijs
    spec:
      containers:
      - name: wikijs
        image: ghcr.io/requarks/wiki:2       # Wiki.js v2 공식 이미지
        env:
        - name: DB_TYPE
          value: "postgres"
        - name: DB_HOST
          value: "postgres"                  # k3s 서비스명으로 자동 연결 (IP 불필요)
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "wiki"
        - name: DB_USER
          value: "wikijs"
        - name: DB_PASS
          value: "wikijs_password"           # postgres-deployment.yaml 과 동일하게
        ports:
        - containerPort: 3000
---
# NodePort Service: 브라우저에서 Wiki.js 접근 허용
apiVersion: v1
kind: Service
metadata:
  name: wikijs
  namespace: dev-tools
spec:
  type: NodePort
  selector:
    app: wikijs
  ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30005       # 브라우저 접속: http://<VM1_IP>:30005
```

### 5-4. Wiki.js 배포 실행

```bash
# 1. PostgreSQL DB 먼저 배포 (Wiki.js가 DB에 의존)
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres-deployment.yaml

# 2. postgres Pod가 Running인지 확인 후 Wiki.js 배포
kubectl get pods -n dev-tools
```

postgres가 `Running` 이면:

```bash
kubectl apply -f wikijs-deployment.yaml
```

### 5-5. 전체 Pod 동작 확인

```bash
kubectl get pods -n dev-tools
```

**정상 출력 예시 (3개 모두 Running 이어야 함):**
```
NAME                        READY   STATUS    RESTARTS   AGE
gitea-7d9f8b6c4-xk2p9      1/1     Running   0          5m
postgres-6c8f9d7b5-mn3q4   1/1     Running   0          2m
wikijs-5b7c4d8a9-pq7r1     1/1     Running   0          1m
```

```bash
kubectl get svc -n dev-tools
```

**정상 출력 예시:**
```
NAME       TYPE        CLUSTER-IP   PORT(S)
gitea      NodePort    10.43.x.x    3000:30000/TCP,22:30022/TCP
postgres   ClusterIP   10.43.x.x    5432/TCP
wikijs     NodePort    10.43.x.x    3000:30005/TCP
```

**Wiki.js 브라우저 접속 테스트:** `http://<VM1_IP>:30005`

> ✅ Wiki.js 관리자 계정 생성 화면이 보이면 성공!

---

## 6. Step 3 — Gitea 초기 설정 및 조직 생성

> 브라우저에서 `http://<VM1_IP>:30000` 접속

### 6-1. 최초 설치 화면 설정

Gitea에 처음 접속하면 초기 설정 페이지가 나타납니다.

| 항목 | 입력값 | 주의사항 |
|------|--------|---------|
| 데이터베이스 유형 | `SQLite3` | 간단 설치. 소규모 팀에 충분 |
| 사이트 제목 | 원하는 이름 (예: `My Dev Lab`) | |
| 루트 URL | `http://<VM1_IP>:30000/` | **반드시 VM의 실제 IP로 변경** |

하단 **관리자 계정 설정:**

| 항목 | 입력값 |
|------|--------|
| 관리자 사용자명 | 원하는 아이디 (예: `rt`) |
| 비밀번호 | 안전한 비밀번호 |
| 이메일 | 이메일 주소 |

**[Gitea 설치]** 클릭 → 완료 후 로그인

> ⚠️ **루트 URL이 `127.0.0.1` 이나 `localhost`로 되어 있으면 외부 브라우저에서 접근 불가.**  
> 반드시 `<VM1_IP>`(실제 VM IP)으로 입력하세요.

### 6-2. 조직(Organization) 생성 — `infra-lab`

> 💡 **왜 조직을 만드나요?**
>
> 개인 계정 아래에 저장소를 만들 수도 있지만, **조직** 아래에 저장소를 두면:
> - 저장소를 프로젝트 단위로 그룹화 가능 (`infra-lab/wiki-storage`, `infra-lab/k3s-iac` 등)
> - 나중에 팀원을 조직에 추가하면 여러 저장소에 한 번에 권한 부여 가능
> - GitHub의 GitHub Organization과 동일한 개념

**① 조직 생성 진입:**
우측 상단 **[+]** 버튼 → **[새 조직]** 클릭

**② 조직 정보 입력:**

| 항목 | 입력값 | 설명 |
|------|--------|------|
| 조직명 | `infra-lab` | URL에 사용됨. 소문자+하이픈 권장 |
| 가시성 | `비공개` | 권장 |

**[조직 만들기]** 클릭

**③ 조직 생성 확인:**
Gitea 대시보드에서 좌측 상단 계정 드롭다운을 누르면 `rt` (개인)과 `infra-lab` (조직)이 모두 보여야 합니다.

```
대시보드 컨텍스트 전환:
  ├── rt          ← 개인 계정
  └── infra-lab   ← 조직 (이 가이드에서 사용)
```

### 6-3. `wiki-storage` 저장소 생성 (조직 소유)

> ⚠️ **반드시 조직(`infra-lab`) 소유로 저장소를 만들어야 합니다.**  
> 개인 계정(`rt`) 소유로 만들면 URL이 달라지므로 이후 Wiki.js 연동 설정이 틀려집니다.

**① 저장소 생성:**
우측 상단 **[+]** 버튼 → **[새 저장소]** 클릭

**② 소유자 확인 (핵심):**

페이지 상단 **소유자(Owner)** 드롭다운에서 **`infra-lab`** 선택

```
소유자 선택:
  ○ rt          ← 개인 계정 (선택 금지)
  ● infra-lab   ← 조직 계정 (반드시 여기 선택!)
```

**③ 저장소 정보 입력:**

| 항목 | 입력값 | 주의사항 |
|------|--------|---------|
| 저장소 이름 | `wiki-storage` | 오타 금지. 대소문자 구분 |
| 가시성 | `비공개 (Private)` | 권장 |
| 초기화 옵션 | **아무것도 체크하지 않음** | README, .gitignore, License 추가 금지 |

> ⚠️ **초기화 파일을 추가하면 Wiki.js 연동 시 충돌이 발생합니다. 반드시 빈 저장소로 생성하세요.**

**[저장소 만들기]** 클릭

**④ 저장소 URL 복사:**
생성 완료 화면 중앙의 HTTP 주소를 복사합니다.

```
복사할 주소 형식:
http://<VM1_IP>:30000/infra-lab/wiki-storage.git
         ↑ VM IP         ↑ 조직명     ↑ 저장소명
```

> 💡 이 주소를 메모장에 붙여두세요. Step 5 Wiki.js 연동에서 사용합니다.

---

## 7. Step 4 — Wiki.js 초기 설정 및 한글화

> 브라우저에서 `http://<VM1_IP>:30005` 접속

### 7-1. 관리자 계정 생성

Wiki.js 최초 접속 시 관리자 계정 생성 화면이 나타납니다.

| 항목 | 입력값 |
|------|--------|
| 이메일 | 관리자 이메일 |
| 비밀번호 | 안전한 비밀번호 (8자 이상) |

**[Create Account]** 클릭 → Wiki.js 메인 화면 진입

### 7-2. 한글 언어 팩 설치 (선택 권장)

> ⚠️ **언어를 한국어로 설정해도 Gitea 저장소에 `ko/` 폴더가 생기지는 않습니다.**
> 언어 설정은 Wiki.js **UI 인터페이스**만 한글로 바꿔줄 뿐, 파일 경로와는 무관합니다.

**① 설정 진입:**
우측 상단 **톱니바퀴 아이콘(⚙)** 클릭 → 좌측 메뉴 **[Locales]** 클릭

**② 한국어 팩 다운로드:**
목록에서 `Korean` 항목을 찾아 **구름 모양(☁ Download) 아이콘** 클릭 → 다운로드 완료 대기

**③ 언어 적용:**
- 상단 `Select language` 드롭다운 → `Korean` 선택
- 우측 상단 **[APPLY]** 클릭

> ✅ 메뉴 전체가 한글로 바뀌면 성공!

---

## 8. Step 5 — Wiki.js ↔ Gitea 저장소 연동 (핵심)

> 🔑 **이 단계가 가장 중요합니다.** 이 설정이 완료되어야 Wiki.js에서 작성한 글이 Gitea에 자동으로 저장됩니다.

### 8-1. 보관소 설정 진입

Wiki.js 관리 메뉴(⚙) → 좌측 **[보관소(Storage)]** 클릭

### 8-2. Git 스토리지 활성화

목록에서 **Git** 항목을 찾아 **[활성화]** 버튼 클릭

### 8-3. 연동 파라미터 입력

| 설정 항목 | 입력값 | 설명 |
|-----------|--------|------|
| Repository URL | `http://<VM1_IP>:30000/infra-lab/wiki-storage.git` | Step 6-3에서 복사한 조직 저장소 주소 |
| Branch | `main` | Gitea 기본 브랜치명 |
| 인증 방법 | `User / Password` | SSH 키 방식 대신 계정/비밀번호 사용 |
| 사용자명 | Gitea 개인 계정 아이디 (예: `rt`) | 조직명이 아닌 **개인 계정** 아이디 입력 |
| 비밀번호 | Gitea 로그인 비밀번호 | |
| 동기화 방향 | **[바꾼 내용만 보관소에 저장하기]** | Wiki.js → Gitea 단방향 저장 |

> ⚠️ **사용자명 주의**: 조직명 `infra-lab`이 아니라 **개인 계정 아이디** (`rt` 등)를 입력해야 합니다.  
> Gitea의 인증은 개인 계정 기준으로 동작하며, 개인 계정이 조직의 멤버이면 조직 저장소에 접근할 수 있습니다.

### 8-4. 적용

우측 상단 **[적용]** 클릭

> ✅ 상단에 **"Success"** 또는 **"성공"** 알림이 뜨면 연동 완료!

> ❌ 오류가 뜨면 → [트러블슈팅 가이드](#12-트러블슈팅-가이드) 참고

---

## 9. Step 6 — 첫 번째 문서 home.md 생성

> ⚠️ **이 단계를 건너뛰면 Wiki.js 접속 시 계속 오류 화면이 뜹니다.** 반드시 진행하세요.

`home.md`는 Wiki.js의 대문(메인) 페이지입니다. 설치 직후 이 파일이 없으면 모든 접속 요청이 오류 처리됩니다.

### 9-1. 홈 문서 생성

Wiki.js 메인 화면에 접속하면 아래 메시지가 뜹니다:

```
홈 문서를 만들고 시작해 봅시다
```

**[홈 문서 만들기]** 버튼 클릭

### 9-2. 에디터 및 페이지 정보 입력

**① 에디터 선택:** `Markdown` 선택

**② 페이지 정보 입력:**

| 항목 | 입력값 | 주의사항 |
|------|--------|---------|
| 제목 | `나의 개발 위키` | 자유롭게 입력 가능 |
| 경로 (Path) | `home` | **절대 수정 금지!** 반드시 `home`이어야 함 |
| 언어 | `ko` | 한국어 페이지임을 표시 (폴더 생성과 무관) |

> ⚠️ **경로가 `home`이 아니면 메인 페이지로 인식되지 않습니다.**

**③ 내용 입력:**

```markdown
# 환영합니다!

이 페이지는 Wiki.js에서 작성되어 Gitea로 자동 백업되는 첫 번째 페이지입니다.

## 위키 구조
- dev/ : 개발 관련 문서
  - golang/ : Golang 개발 가이드
- infra/ : 인프라 관련 문서
```

**④ 저장:**
우측 상단 **[생성]** 버튼 클릭

> ✅ 홈 페이지가 생성되고 Wiki.js 메인으로 이동하면 성공!

### 9-3. Gitea에 home.md 동기화 확인

강제 동기화 실행:
**Wiki.js 관리(⚙) → [보관소] → [Force Sync] → [실행]**

Gitea `wiki-storage` 저장소에서 `home.md` 파일이 루트에 생성되었는지 확인:

```
wiki-storage/
└── home.md    ← 여기에 생성됨 (ko/ 폴더 없음)
```

---

## 10. Step 7 — 폴더 계층 구조 생성 실습

Wiki.js에서 경로에 슬래시(`/`)를 넣으면 Gitea에 자동으로 폴더가 생성됩니다. 이것이 **가상-물리 동기화 구조**의 핵심입니다.

### 경로 동작 원리 (실제 검증 기준)

```
Wiki.js 경로 입력:   dev/golang/architecture
                              ↓ 동기화 후 실제 Gitea 경로
Gitea 저장 위치:     infra-lab/wiki-storage/dev/golang/architecture.md

                 ← ko/ 폴더 없이 dev/ 부터 바로 시작 →
```

**실제 확인된 Gitea URL:**
```
<VM1_IP>:30000/infra-lab/wiki-storage/src/branch/main/dev/golang/architecture.md
```

### 10-1. 새 문서 추가

Wiki.js 메인 화면 우측 상단 **[문서+ 아이콘(📄+)]** 클릭

### 10-2. 경로 설정 (매우 중요)

경로 입력 창에서:

1. 기본으로 입력된 `/new-page` 를 **전부 지웁니다**
2. `dev/golang/architecture` 를 입력합니다

```
입력값:  dev/golang/architecture
          ↑      ↑        ↑
        폴더1  폴더2    파일명
           ↓      ↓        ↓
Gitea:  dev/  golang/  architecture.md
```

> 💡 슬래시(`/`)가 Gitea에서 폴더 계층으로 변환됩니다.  
> 언어(`ko`)는 폴더 경로에 영향을 주지 않습니다.

**[선택]** 버튼 클릭

### 10-3. 문서 작성 및 저장

| 항목 | 입력값 |
|------|--------|
| 제목 | `Golang 아키텍처` |
| 에디터 | `Markdown` |

```markdown
# Golang 아키텍처

Golang 서버 아키텍처 학습 문서입니다.

## 주요 컴포넌트
- API Server
- Worker
- DB Layer
```

우측 상단 **[생성]** 클릭

---

## 11. Step 8 — Gitea 동기화 검증

### 11-1. 강제 동기화 실행

Wiki.js는 기본 5분 주기로 자동 동기화됩니다. 즉시 확인하려면 강제 동기화를 실행합니다.

**Wiki.js 관리(⚙) → [보관소(Storage)] → 맨 아래 [Force Sync (강제 동기화)] 옆 [실행] 버튼 클릭**

### 11-2. Gitea에서 폴더 구조 확인

`http://<VM1_IP>:30000` 접속 → 좌측 대시보드의 **`infra-lab/wiki-storage`** 저장소 클릭

**실제 생성되어야 하는 폴더 구조:**

```
infra-lab/wiki-storage (main 브랜치)
│
├── home.md                     ← Step 6에서 만든 홈 페이지 (루트에 위치)
└── dev/                        ← ko/ 없이 dev/ 부터 시작
     └── golang/
          └── architecture.md   ← Step 7에서 만든 문서
```

> ✅ `dev/golang/architecture.md` 파일이 `ko/` 없이 `dev/` 부터 바로 존재하면 **Phase 2 핵심 구축 완료!**

### 11-3. Gitea 커밋 로그 확인

Gitea `wiki-storage` 저장소 → **[코드]** 탭 → 최근 커밋 목록에서 아래와 같은 커밋이 보이면 정상:

```
fa17a640eb   docs: create dev/golang/architecture   1시간 전
fc06029816   docs: create home                       1시간 전
```

### 11-4. Wiki.js 접속 최종 확인

`http://<VM1_IP>:30005/en/home` 접속:

```
타이틀: Home | 나의 개발 위키
내용:   환영합니다!
        이 페이지는 Wiki.js에서 작성되어 Gitea로 자동 백업되는 첫 번째 페이지입니다.
```

---

## 12. 트러블슈팅 가이드

### ❌ Pod이 Running 상태가 안 될 때

```bash
# 문제 Pod 확인
kubectl get pods -n dev-tools

# 상세 오류 확인 (Events 섹션에서 원인 파악)
kubectl describe pod <Pod_이름> -n dev-tools

# 컨테이너 로그 직접 확인
kubectl logs <Pod_이름> -n dev-tools
```

| 증상 | 원인 | 해결 |
|------|------|------|
| `ImagePullBackOff` | 이미지 다운로드 실패 | 인터넷 연결 확인 후 재시도 |
| `CrashLoopBackOff` | 컨테이너 실행 오류 | `kubectl logs`로 오류 메시지 확인 |
| `Pending` | 리소스 부족 | `kubectl describe pod`의 Events 확인 |

### ❌ Wiki.js → Gitea 연동 시 "Success"가 안 뜰 때

**체크 1: Repository URL 형식 확인**
```
✅ 올바름: http://<VM1_IP>:30000/infra-lab/wiki-storage.git
❌ 틀림 1: http://<VM1_IP>:30000/infra-lab/wiki-storage      (.git 누락)
❌ 틀림 2: http://<VM1_IP>:30000/rt/wiki-storage.git          (개인계정 경로)
```

**체크 2: 사용자명 확인**
```
✅ 올바름: rt           (Gitea 개인 계정 아이디)
❌ 틀림:   infra-lab    (조직명은 사용자명이 아님)
```

**체크 3: 저장소가 완전히 비어있는지 확인**
- Gitea `wiki-storage` 저장소에 접속해서 파일이 하나도 없는지 확인
- README.md나 다른 파일이 있으면 저장소 삭제 후 빈 상태로 재생성

**체크 4: 개인 계정이 조직 멤버인지 확인**
- Gitea → `infra-lab` 조직 페이지 → [멤버] 탭
- `rt` (개인 계정)이 멤버 목록에 있는지 확인
- 없으면 멤버로 추가

**체크 5: Branch 이름 확인**
- Gitea 저장소 기본 브랜치가 `main`인지 `master`인지 확인
- Wiki.js 설정에서 Branch 항목을 Gitea와 동일하게 입력

### ❌ `ko/` 폴더가 생기거나 경로가 이상할 때

Wiki.js 언어 설정(`ko`)은 UI 표시 언어일 뿐, **Gitea 저장 경로와 무관**합니다.

```
정상 동작:
  Wiki.js 경로 dev/golang/architecture → Gitea: dev/golang/architecture.md  ✅

비정상 (과거 버전 이슈):
  Wiki.js 경로 dev/golang/architecture → Gitea: ko/dev/golang/architecture.md  ❌
```

만약 `ko/` 폴더가 생성된다면:
- Wiki.js 설정에서 **언어(Locale)를 `en`으로 변경** 후 재테스트
- 또는 `ko/` 경로를 그대로 사용하고 문서 구조를 그에 맞게 정리

### ❌ Gitea에 폴더가 생기지 않을 때

1. 경로에 슬래시(`/`)를 올바르게 넣었는지 확인
   ```
   ✅ 올바름: dev/golang/architecture
   ❌ 틀림:   dev golang architecture   (공백 사용 금지)
   ❌틀림:    /dev/golang/architecture  (앞에 / 넣으면 안 됨)
   ```
2. 강제 동기화를 실행했는지 확인 (보관소 → Force Sync)
3. Wiki.js 동기화 방향이 **"바꾼 내용만 보관소에 저장하기 (Push)"** 인지 확인

### ❌ home.md를 만들었는데 오류 화면이 뜰 때

- 경로가 정확히 `home` 인지 확인 (대소문자: `Home`은 안 됨)
- 브라우저 캐시 삭제 후 재접속 (`Ctrl+Shift+Delete`)
- 강제 동기화 후 Gitea에 `home.md`가 루트에 생성되었는지 확인

### ❌ Wiki.js Pod이 postgres에 연결 못할 때

```bash
# postgres Pod 상태 확인
kubectl get pods -n dev-tools | grep postgres

# postgres 서비스 확인
kubectl get svc -n dev-tools | grep postgres

# postgres 로그 확인
kubectl logs <postgres Pod명> -n dev-tools
```

`postgres` 서비스가 없으면 → `kubectl apply -f postgres-deployment.yaml` 재실행

---

## 13. Phase 2 완료 체크리스트

아래 항목을 모두 확인한 후 Phase 3으로 진행하세요.

### 인프라 배포

- ☐ `kubectl get pods -n dev-tools` → Gitea, postgres, wikijs 모두 `Running`
- ☐ `kubectl get svc -n dev-tools` → Gitea(30000), wikijs(30005) NodePort 확인
- ☐ 방화벽 포트 30000, 30005 개방 확인 (`sudo ufw status`)

### Gitea 설정

- ☐ `http://<VM1_IP>:30000` 브라우저 접속 성공
- ☐ Gitea 개인 관리자 계정 생성 및 로그인 성공
- ☐ `infra-lab` 조직 생성 완료
- ☐ 조직 소유의 `wiki-storage` 저장소 생성 완료 (빈 저장소, 초기화 파일 없음)
- ☐ HTTP Clone URL 복사 완료 (`/infra-lab/wiki-storage.git` 형식)

### Wiki.js 설정

- ☐ `http://<VM1_IP>:30005` 브라우저 접속 성공
- ☐ Wiki.js 관리자 계정 생성 및 로그인 성공
- ☐ 한국어 팩 설치 및 적용 완료 (선택)

### 연동 검증

- ☐ Wiki.js 보관소 → Git 연동 설정 → "Success" 확인
  - Repository URL: `http://<VM1_IP>:30000/infra-lab/wiki-storage.git`
  - 사용자명: 개인 계정 아이디 (예: `rt`)
- ☐ `home.md` 생성 완료 (경로: `home`, 언어: `ko`)
- ☐ `dev/golang/architecture` 문서 생성 완료
- ☐ 강제 동기화 실행 완료
- ☐ Gitea `infra-lab/wiki-storage` → `dev/golang/architecture.md` 파일 확인 (`ko/` 없이 `dev/` 부터 시작)

### 스냅샷

- ☐ Proxmox 웹 UI → VM 100 (vm-dev) → 스냅샷 → `Phase2_Complete` 스냅샷 생성

---

## 14. 다음 단계

### Phase 2 완성 축하! 🎉

이제 여러분의 로컬 서버에 다음이 구축되었습니다:

```
✅ Gitea (조직: infra-lab)  — Self-Hosted GitHub
   ├── infra-lab/wiki-storage  ← Wiki.js 문서 자동 저장
   └── infra-lab/k3s-iac       ← k3s YAML 매니페스트 관리

✅ Wiki.js  — Self-Hosted Notion/Confluence
   └── 웹에서 글 쓰면 → Gitea에 .md 파일로 자동 동기화

✅ Docs as Code — 문서가 코드처럼 Git으로 버전 관리됨
```

### 다음 Phase 안내

| Phase | 내용 | 전제 조건 |
|-------|------|-----------|
| **Phase 2-C** | Gitea Actions Runner 배포 (CI/CD 자동화) | Phase 2 완료 |
| **Phase 3** | VM 2 — Ollama + Open-WebUI (로컬 AI 서버) | Phase 1 VM 2 준비 |
| **Phase 4** | VM 3 — Mattermost (팀 채팅 서버) | Phase 1 VM 3 준비 |

---

## 📌 자주 쓰는 명령어 모음

```bash
# 전체 Pod 상태 확인
kubectl get pods -n dev-tools

# 전체 리소스 한번에 확인
kubectl get all -n dev-tools

# 특정 Pod 실시간 로그
kubectl logs -f <Pod이름> -n dev-tools

# Pod 상세 정보 (오류 원인 파악)
kubectl describe pod <Pod이름> -n dev-tools

# Deployment 재시작 (Pod 강제 재생성)
kubectl rollout restart deployment/gitea -n dev-tools
kubectl rollout restart deployment/wikijs -n dev-tools
kubectl rollout restart deployment/postgres -n dev-tools

# 서비스 목록 확인
kubectl get svc -n dev-tools

# 네임스페이스 전체 삭제 (초기화 시 - 주의!)
# kubectl delete namespace dev-tools
```

---

## 📡 서비스 접속 정보 요약

| 서비스 | 접속 URL | 역할 |
|--------|----------|------|
| **Gitea** | `http://<VM1_IP>:30000` | 소스코드 + 마크다운 저장소 |
| **Wiki.js** | `http://<VM1_IP>:30005` | 기술 문서 웹 편집기 |
| **Gitea 조직** | `http://<VM1_IP>:30000/infra-lab` | infra-lab 조직 페이지 |
| **위키 저장소** | `http://<VM1_IP>:30000/infra-lab/wiki-storage` | 위키 파일 저장소 |

---

> 💡 **핵심 포인트 요약**
>
> 1. **Gitea 저장소는 조직 소유로 생성** — `infra-lab/wiki-storage` (개인 계정 경로 아님)
> 2. **Wiki.js 연동 시 사용자명은 개인 계정** — 조직명(`infra-lab`)이 아닌 개인 아이디(`rt`) 입력
> 3. **`ko/` 디렉토리는 생성되지 않음** — 경로 `dev/golang/architecture` 입력 시 Gitea에 `dev/golang/architecture.md` 로 저장 (`ko/` 없음)
> 4. **초기화 파일 없이 빈 저장소** — `wiki-storage` 생성 시 README, .gitignore 추가 금지
> 5. **백업은 Gitea만** — Gitea `wiki-storage` 저장소를 백업하면 모든 위키 내용이 보존됨

---

*작성일: 2026-04-20 | 버전: v1.1*  
*작성자: Kim Jong-in (Technical Architect)*  
*기반 자료: Full_IT_Infrastructure_Roadmap_v3.md · 서버 명령어 History · SOP · 브라우저 캡처 화면*  
*v1.1 업데이트: infra-lab 조직 개념 추가 · ko/ 디렉토리 미생성 동작 반영*
