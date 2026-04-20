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
| **버전** | v1.0 (2026-04-20 작성) |
| **대상 VM** | VM 100 — `vm-dev` (172.28.136.160) |
| **사전 조건** | Phase 1 완료 — Ubuntu 22.04 + k3s 설치 및 `dev-tools` 네임스페이스 생성 |
| **예상 소요 시간** | 2 ~ 4 시간 |
| **난이도** | ⭐⭐☆☆☆ (초보자 가능) |

---

## 📋 목차

1. [이 가이드에서 만들 것](#1-이-가이드에서-만들-것)
2. [사전 조건 확인](#2-사전-조건-확인)
3. [작업 디렉토리 구성](#3-작업-디렉토리-구성)
4. [Step 1 — Gitea 배포 (Pod A)](#4-step-1--gitea-배포-pod-a)
5. [Step 2 — Wiki.js 배포 (Pod B)](#5-step-2--wikijs-배포-pod-b)
6. [Step 3 — Gitea 초기 설정](#6-step-3--gitea-초기-설정)
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
VM 1 (vm-dev, 172.28.136.160)
│
├─── k3s Namespace: dev-tools
│     │
│     ├── Pod A: Gitea        → http://172.28.136.160:30000
│     │         (마크다운 .md 파일이 실제로 저장되는 Git 서버)
│     │
│     └── Pod B: Wiki.js      → http://172.28.136.160:30005
│               (웹 브라우저에서 문서를 작성·편집하는 인터페이스)
│
└─── Gitea 저장소: wiki-storage
      │
      ├── home.md              ← Wiki.js 메인 페이지
      └── dev/
           └── golang/
                  └── architecture.md  ← 폴더 계층 문서
```

### 작동 원리 (한 줄 요약)
> **Wiki.js에서 글을 쓰면** → 지정한 경로(슬래시 `/` 구분)대로 → **Gitea 저장소에 `.md` 파일로 자동 저장된다**

### 완성 화면 미리보기

| 서비스 | 접속 주소 | 역할 |
|--------|-----------|------|
| Gitea | `http://172.28.136.160:30000` | .md 파일 물리 저장소 |
| Wiki.js | `http://172.28.136.160:30005` | 문서 웹 편집기 |

---

## 2. 사전 조건 확인

> ⚠️ **이 단계를 건너뛰면 이후 모든 작업이 실패할 수 있습니다.** 반드시 확인 후 진행하세요.

### 2-1. VM 1 접속

```bash
# Windows에서 SSH 접속 (vm-dev IP로 변경)
ssh rt@172.28.136.160
```

### 2-2. k3s 동작 확인

```bash
# k3s 상태 확인
kubectl get nodes
```

**정상 출력 예시:**
```
NAME     STATUS   ROLES                  AGE   VERSION
vm-dev   Ready    control-plane,master   5d    v1.29.x+k3s1
```

> ❌ `NotReady` 또는 명령어 오류가 뜨면 → `sudo systemctl restart k3s` 실행 후 재확인

### 2-3. 네임스페이스 확인 및 생성

```bash
# 네임스페이스 목록 확인
kubectl get namespaces
```

`dev-tools`가 없으면 아래 명령어로 생성:

```bash
kubectl create namespace dev-tools
```

**확인:**
```bash
kubectl get namespaces | grep dev-tools
# 출력: dev-tools   Active   ...
```

### 2-4. 방화벽 포트 개방

Gitea와 Wiki.js에 외부 브라우저에서 접속하려면 포트를 미리 열어야 합니다.

```bash
sudo ufw allow 30000/tcp   # Gitea
sudo ufw allow 30005/tcp   # Wiki.js
sudo ufw status            # 개방 포트 확인
```

---

## 3. 작업 디렉토리 구성

YAML 파일들을 체계적으로 관리하기 위한 폴더 구조를 만듭니다.

```bash
# 작업 디렉토리 생성
mkdir -p ~/gitea
mkdir -p ~/wikijs

# 구조 확인
ls ~
# 출력: gitea  wikijs  ...
```

---

## 4. Step 1 — Gitea 배포 (Pod A)

Gitea는 마크다운 파일이 실제로 저장되는 **물리 저장소**입니다. GitHub의 셀프호스팅 버전이라고 생각하면 됩니다.

### 4-1. Gitea PVC + Deployment + Service YAML 작성

```bash
cd ~/gitea
nano gitea-pvc.yaml
```

아래 내용을 붙여넣고 저장 (`Ctrl+X` → `Y` → `Enter`):

```yaml
# gitea-pvc.yaml
# Gitea 데이터를 영구 저장하기 위한 볼륨 (컨테이너 재시작해도 데이터 보존)
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

```bash
nano gitea-deployment.yaml
```

아래 내용을 붙여넣고 저장:

```yaml
# gitea-deployment.yaml
# Gitea 서버 Pod 배포 설정
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
        - containerPort: 3000    # Gitea 웹 포트
        - containerPort: 22      # Gitea SSH 포트
        volumeMounts:
        - name: gitea-storage
          mountPath: /data        # Gitea 데이터 저장 경로
      volumes:
      - name: gitea-storage
        persistentVolumeClaim:
          claimName: gitea-pvc   # 위에서 만든 PVC 연결
---
# Gitea를 외부에서 접근 가능하게 만드는 Service
apiVersion: v1
kind: Service
metadata:
  name: gitea
  namespace: dev-tools
spec:
  type: NodePort          # VM 외부에서 접근 허용
  selector:
    app: gitea
  ports:
  - name: web
    port: 3000
    targetPort: 3000
    nodePort: 30000       # 브라우저 접속 포트: http://VM_IP:30000
  - name: ssh
    port: 22
    targetPort: 22
    nodePort: 30022       # SSH 접속 포트 (git clone/push)
```

### 4-2. Gitea 배포 실행

```bash
# PVC 먼저 생성 (저장소 준비)
kubectl apply -f gitea-pvc.yaml

# Deployment + Service 배포
kubectl apply -f gitea-deployment.yaml
```

### 4-3. Gitea Pod 동작 확인

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

> 💡 `ContainerCreating` → `Running` 으로 바뀌는 데 1~2분 소요됩니다. 기다리세요.

```bash
# 서비스 포트 확인
kubectl get svc -n dev-tools
# 출력: gitea   NodePort   ...   3000:30000/TCP,22:30022/TCP
```

**브라우저 접속 테스트:** `http://172.28.136.160:30000`

> ✅ Gitea 초기 설정 화면이 보이면 성공! (설정은 Step 3에서 진행)

---

## 5. Step 2 — Wiki.js 배포 (Pod B)

Wiki.js는 웹에서 문서를 작성하는 **편집 인터페이스**입니다. Wiki.js가 작성한 내용을 Gitea에 자동으로 저장합니다.

Wiki.js는 데이터를 보관하기 위해 **PostgreSQL 데이터베이스**가 필요합니다.

### 5-1. PostgreSQL DB PVC + Deployment 작성

```bash
cd ~/wikijs
nano postgres-pvc.yaml
```

아래 내용 입력 후 저장:

```yaml
# postgres-pvc.yaml
# Wiki.js가 사용하는 PostgreSQL 데이터베이스 볼륨
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
          value: "wiki"             # 데이터베이스 이름
        - name: POSTGRES_USER
          value: "wikijs"           # DB 접속 계정
        - name: POSTGRES_PASSWORD
          value: "wikijs_password"  # ⚠️ 운영 환경에서는 반드시 강력한 비밀번호로 변경!
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
# PostgreSQL을 k3s 내부 네트워크에서만 접근 가능한 서비스로 등록
# (ClusterIP = 외부 노출 없음, Wiki.js Pod에서만 접근 가능)
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

### 5-2. Wiki.js Deployment 작성

```bash
nano wikijs-deployment.yaml
```

아래 내용 입력 후 저장:

```yaml
# wikijs-deployment.yaml
# Wiki.js 웹 애플리케이션 Pod
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
        image: ghcr.io/requarks/wiki:2     # Wiki.js v2 공식 이미지
        env:
        - name: DB_TYPE
          value: "postgres"
        - name: DB_HOST
          value: "postgres"               # k3s 내부 서비스명으로 연결 (IP 불필요)
        - name: DB_PORT
          value: "5432"
        - name: DB_NAME
          value: "wiki"
        - name: DB_USER
          value: "wikijs"
        - name: DB_PASS
          value: "wikijs_password"        # postgres-deployment.yaml 의 패스워드와 동일
        ports:
        - containerPort: 3000
---
# Wiki.js를 외부 브라우저에서 접근 가능하게 노출
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
    nodePort: 30005       # 브라우저 접속 포트: http://VM_IP:30005
```

### 5-3. Wiki.js 배포 실행

```bash
# PostgreSQL DB 먼저 배포 (Wiki.js가 DB에 의존함)
kubectl apply -f postgres-pvc.yaml
kubectl apply -f postgres-deployment.yaml

# DB Pod가 Running 상태인지 확인 후 Wiki.js 배포
kubectl get pods -n dev-tools
```

**DB Pod가 Running이면** Wiki.js 배포:

```bash
kubectl apply -f wikijs-deployment.yaml
```

### 5-4. 전체 Pod 동작 확인

```bash
kubectl get pods -n dev-tools
```

**정상 출력 예시 (모두 Running 이어야 함):**
```
NAME                        READY   STATUS    RESTARTS   AGE
gitea-7d9f8b6c4-xk2p9      1/1     Running   0          5m
postgres-6c8f9d7b5-mn3q4   1/1     Running   0          2m
wikijs-5b7c4d8a9-pq7r1     1/1     Running   0          1m
```

```bash
# 서비스 목록 확인
kubectl get svc -n dev-tools
```

**정상 출력 예시:**
```
NAME       TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)
gitea      NodePort    10.43.x.x      <none>        3000:30000/TCP
postgres   ClusterIP   10.43.x.x      <none>        5432/TCP
wikijs     NodePort    10.43.x.x      <none>        3000:30005/TCP
```

**브라우저 접속 테스트:** `http://172.28.136.160:30005`

> ✅ Wiki.js 관리자 계정 생성 화면이 보이면 성공!

---

## 6. Step 3 — Gitea 초기 설정

> 브라우저에서 `http://172.28.136.160:30000` 접속

### 6-1. 최초 설치 화면 설정

Gitea에 처음 접속하면 초기 설정 페이지가 나타납니다.

| 항목 | 입력값 | 설명 |
|------|--------|------|
| 데이터베이스 유형 | SQLite3 | 간단한 설치용. 소규모 팀에 충분 |
| 사이트 제목 | 원하는 이름 (예: `My Dev Lab`) | Gitea 메인 페이지 타이틀 |
| HTTP 포트 | 3000 | 기본값 유지 |
| 루트 URL | `http://172.28.136.160:30000/` | **반드시 VM IP로 변경** |

> ⚠️ **루트 URL이 잘못되면 나중에 Wiki.js 연동 시 오류가 발생합니다.**

하단 **[관리자 계정 설정]** 항목:

| 항목 | 입력값 |
|------|--------|
| 관리자 사용자명 | 원하는 아이디 (예: `infra-lab`) |
| 비밀번호 | 안전한 비밀번호 입력 |
| 이메일 | 이메일 주소 입력 |

**[Gitea 설치]** 버튼 클릭 → 설치 완료 후 로그인 화면으로 이동

### 6-2. 관리자 로그인 확인

설정한 아이디/비밀번호로 로그인 → 대시보드가 보이면 정상입니다.

---

## 7. Step 4 — Wiki.js 초기 설정 및 한글화

> 브라우저에서 `http://172.28.136.160:30005` 접속

### 7-1. 관리자 계정 생성

Wiki.js 최초 접속 시 관리자 계정 생성 화면이 나타납니다.

| 항목 | 입력값 |
|------|--------|
| 이메일 | 관리자 이메일 |
| 비밀번호 | 안전한 비밀번호 (8자 이상) |

**[Create Account]** 클릭 → Wiki.js 메인 화면 진입

### 7-2. 한글 언어 팩 설치 (선택 권장)

영문 인터페이스는 메뉴 찾기가 불편할 수 있으므로 한글화를 진행합니다.

**① 설정 진입:**
우측 상단 **톱니바퀴 아이콘(⚙)** 클릭 → 좌측 메뉴 **[Locales]** 클릭

**② 한국어 팩 다운로드:**
목록에서 `Korean` 항목을 찾아 **구름 모양(☁ Download) 아이콘** 클릭

**③ 언어 적용:**
- 상단 `Select language` 드롭다운 → `Korean` 선택
- 우측 상단 **[APPLY]** 버튼 클릭

> ✅ 메뉴 전체가 한글로 바뀌면 성공!

---

## 8. Step 5 — Wiki.js ↔ Gitea 저장소 연동 (핵심)

> 🔑 **이 단계가 가장 중요합니다.** 이 설정이 완료되어야 Wiki.js에서 작성한 글이 Gitea에 자동으로 저장됩니다.

### 8-1. Gitea에서 Wiki용 저장소 생성

먼저 Wiki.js가 문서를 저장할 빈 저장소를 Gitea에 만들어야 합니다.

**① Gitea 접속:** `http://172.28.136.160:30000`

**② 새 저장소 생성:**
우측 상단 **[+]** 버튼 → **[새 저장소]** 클릭

| 항목 | 입력값 | 주의사항 |
|------|--------|---------|
| 저장소 이름 | `wiki-storage` | 오타 금지. 대소문자 구분 |
| 가시성 | 비공개 (Private) | 권장 |
| 초기화 옵션 | **아무것도 체크하지 않음** | README, .gitignore 추가 금지 |

**[저장소 만들기]** 클릭

**③ 저장소 URL 복사:**
생성된 페이지 중앙에 있는 HTTP 주소를 복사합니다.

```
예: http://172.28.136.160:30000/infra-lab/wiki-storage.git
             ↑ VM IP        ↑ Gitea 계정명  ↑ 저장소 이름
```

> 💡 이 주소를 메모장에 붙여두세요. 다음 단계에서 사용합니다.

### 8-2. Wiki.js에서 Git 스토리지 연동 설정

**① 보관소 설정 진입:**
Wiki.js 관리 메뉴(⚙) → 좌측 **[보관소(Storage)]** 클릭

**② Git 활성화:**
목록에서 **Git** 항목을 찾아 **[활성화]** 버튼 클릭

**③ 연동 파라미터 입력:**

| 설정 항목 | 입력값 | 설명 |
|-----------|--------|------|
| Repository URL | `http://172.28.136.160:30000/infra-lab/wiki-storage.git` | 8-1에서 복사한 주소 |
| Branch | `main` | Gitea 기본 브랜치명 |
| 인증 방법 | `User / Password` 선택 | SSH 대신 계정/비밀번호 방식 사용 |
| 사용자명 | Gitea 로그인 아이디 | (예: `infra-lab`) |
| 비밀번호 | Gitea 로그인 비밀번호 | |
| 동기화 방향 | **[바꾼 내용만 보관소에 저장하기]** 선택 | Wiki.js → Gitea 방향으로만 저장 |

**④ 적용:**
우측 상단 **[적용]** 버튼 클릭

> ✅ 상단에 **"Success"** 또는 **"성공"** 알림이 뜨면 연동 완료!

> ❌ 오류가 뜨면 → [트러블슈팅 가이드](#12-트러블슈팅-가이드) 참고

---

## 9. Step 6 — 첫 번째 문서 home.md 생성

> ⚠️ **이 단계를 건너뛰면 Wiki.js 접속 시 계속 오류 화면이 뜹니다.** 반드시 진행하세요.

`home.md`는 Wiki.js의 대문(메인) 페이지입니다. 설치 직후 이 파일이 없으면 모든 경로 요청이 오류 처리됩니다.

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
| 제목 | `나의 개발 위키` | 자유롭게 입력 |
| 경로 (Path) | `home` | **수정 금지!** 반드시 `home`이어야 함 |
| 언어 | `ko` | 한국어로 설정 |

> ⚠️ **경로가 `home`이 아니면 메인 페이지로 인식되지 않습니다.**

**③ 내용 입력:**
본문 에디터에 아래 내용을 입력합니다.

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

> ✅ 홈 페이지가 생성되며 Wiki.js 메인으로 이동하면 성공!

---

## 10. Step 7 — 폴더 계층 구조 생성 실습

Wiki.js에서 경로에 슬래시(`/`)를 넣으면 Gitea에 자동으로 폴더(디렉토리)가 생성됩니다. 이것이 **가상-물리 동기화 구조**의 핵심입니다.

### 원리 이해

```
Wiki.js 경로 입력:   dev/golang/architecture
                      ↓ 자동 변환
Gitea 폴더 구조:     dev/golang/architecture.md
                  
```

### 10-1. 새 문서 추가

Wiki.js 메인 화면 우측 상단 **[문서+ 아이콘]** 클릭

### 10-2. 경로 설정 (매우 중요)

경로 입력 창에서:

1. 기본으로 입력된 `/new-page` 를 **전부 지웁니다**
2. `dev/golang/architecture` 를 입력합니다

```
입력 예시: dev/golang/architecture
             ↑      ↑       ↑
           폴더1  폴더2  파일명
```

> 💡 슬래시(`/`)가 Gitea에서 폴더 계층으로 변환됩니다.

**[선택]** 버튼 클릭

### 10-3. 문서 작성 및 저장

| 항목 | 입력값 |
|------|--------|
| 제목 | `Golang 아키텍처` |
| 에디터 | Markdown |

본문에 내용 작성 후 우측 상단 **[생성]** 클릭

---

## 11. Step 8 — Gitea 동기화 검증

### 11-1. 강제 동기화 실행

Wiki.js는 기본적으로 5분마다 자동 동기화됩니다. 지금 즉시 확인하려면 강제 동기화를 실행합니다.

**Wiki.js 관리 메뉴(⚙) → [보관소(Storage)] → 맨 아래 [Force Sync (강제 동기화)] 옆 [실행] 버튼 클릭**

### 11-2. Gitea에서 폴더 구조 확인

`http://172.28.136.160:30000` 접속 → `wiki-storage` 저장소 클릭

아래 폴더 구조가 생성되어 있는지 확인:

```
wiki-storage/
├── home.md                 ← Step 6에서 만든 홈 페이지
└── dev/
     └── golang/
            └── architecture.md ← Step 7에서 만든 문서
```

> ✅ `architecture.md` 파일이 존재하면 **Phase 2 핵심 구축 완료!**

### 11-3. 실제 완성 화면 확인

**Gitea 파일 내용 확인:**

`wiki-storage → dev → golang → architecture.md` 파일 클릭 시 아래와 같이 보여야 합니다:

```
위치: wiki-storage/dev/golang/architecture.md
내용: # Golang 아키텍처
      Golang 서버 아키텍처 학습
```

**Wiki.js 홈 화면 확인:**

`http://172.28.136.160:30005/en/home` 접속 시:

```
Title: Home | 나의 개발 위키
내용: 환영합니다!
      이 페이지는 Wiki.js에서 작성되어 Gitea로 자동 백업되는...
```

---

## 12. 트러블슈팅 가이드

### ❌ Pod이 Running이 안 될 때

```bash
# 문제가 있는 Pod 이름 확인
kubectl get pods -n dev-tools

# 특정 Pod 상세 로그 확인
kubectl describe pod <Pod_이름> -n dev-tools

# Pod 로그 직접 확인
kubectl logs <Pod_이름> -n dev-tools
```

**자주 발생하는 원인:**

| 증상 | 원인 | 해결 |
|------|------|------|
| `ImagePullBackOff` | 이미지 다운로드 실패 (인터넷 불안정) | 잠시 후 재시도: `kubectl rollout restart deployment/<이름> -n dev-tools` |
| `CrashLoopBackOff` | 컨테이너 실행 오류 | `kubectl logs <Pod명> -n dev-tools` 로 오류 원인 확인 |
| `Pending` | 리소스 부족 | `kubectl describe pod <Pod명> -n dev-tools` 에서 Events 확인 |

### ❌ Wiki.js에서 "Success" 대신 오류가 뜰 때

**원인 1: Gitea 계정 정보 오류**
- Repository URL, 사용자명, 비밀번호를 다시 한 번 정확히 입력했는지 확인
- Gitea 로그인이 정상인지 별도 브라우저 탭에서 확인

**원인 2: 저장소 URL 오류**
- URL 끝에 `.git`이 포함되어 있는지 확인
  ```
  ✅ 올바름: http://172.28.136.160:30000/infra-lab/wiki-storage.git
  ❌ 틀림:   http://172.28.136.160:30000/infra-lab/wiki-storage
  ```

**원인 3: Branch 이름 오류**
- `main`으로 설정했는지 확인 (Gitea 기본 브랜치가 `master`인 경우 `master`로 입력)
- Gitea 저장소 페이지에서 브랜치 이름 확인 후 동일하게 입력

**원인 4: 저장소가 비어있지 않음**
- wiki-storage 저장소를 생성 시 README.md 등을 추가했다면 충돌 발생
- 저장소를 삭제 후 **아무 파일도 없는 빈 저장소로 재생성**

### ❌ Gitea에 폴더가 생기지 않을 때

1. 경로 입력 시 슬래시(`/`)를 정확히 넣었는지 확인
   ```
   ✅ 올바름: dev/golang/architecture
   ❌ 틀림:   dev golang architecture
   ```
2. 강제 동기화를 실행했는지 확인
3. Wiki.js → 보관소 설정이 **"바꾼 내용만 보관소에 저장하기 (Push)"** 로 선택되어 있는지 확인

### ❌ home.md를 만들었는데도 오류 화면이 뜰 때

- 경로가 정확히 `home` 인지 확인 (대소문자 구분: `Home`은 안 됨)
- 언어가 `ko`로 설정되어 있는지 확인
- 브라우저 캐시 삭제 후 재접속

### ❌ Wiki.js Pod이 postgres에 연결 못할 때

```bash
# postgres Pod가 Running인지 먼저 확인
kubectl get pods -n dev-tools | grep postgres

# postgres 서비스가 등록되어 있는지 확인
kubectl get svc -n dev-tools | grep postgres
```

postgres Pod가 Running이 아니면 → `kubectl logs <postgres Pod명> -n dev-tools` 로 원인 확인

---

## 13. Phase 2 완료 체크리스트

아래 항목을 모두 확인한 후 Phase 3으로 진행하세요.

### 인프라 배포

- ☐ `kubectl get pods -n dev-tools` → Gitea, postgres, wikijs 모두 `Running` 상태
- ☐ `kubectl get svc -n dev-tools` → Gitea(30000), wikijs(30005) NodePort 확인
- ☐ 방화벽 포트 30000, 30005 개방 확인

### Gitea 설정

- ☐ `http://172.28.136.160:30000` 브라우저 접속 성공
- ☐ Gitea 관리자 계정 생성 및 로그인 성공
- ☐ `wiki-storage` 저장소 생성 완료 (빈 저장소)
- ☐ HTTP Clone URL 복사 완료

### Wiki.js 설정

- ☐ `http://172.28.136.160:30005` 브라우저 접속 성공
- ☐ Wiki.js 관리자 계정 생성 및 로그인 성공
- ☐ 한국어 팩 설치 및 적용 완료 (선택)

### 연동 검증

- ☐ Wiki.js 보관소(Storage) → Git 연동 설정 → "Success" 확인
- ☐ `home.md` 홈 문서 생성 완료 (경로: `home`, 언어: `ko`)
- ☐ `dev/golang/architecture` 문서 생성 완료
- ☐ Gitea `wiki-storage` 저장소에 `ko/dev/golang/architecture.md` 파일 존재 확인

### 스냅샷

- ☐ Proxmox 웹 UI → VM 100 (vm-dev) → 스냅샷 → **Phase2_Complete** 스냅샷 생성

---

## 14. 다음 단계

### Phase 2 완성 축하! 🎉

이제 여러분의 로컬 서버에 다음이 구축되었습니다:

```
✅ Gitea  — Self-Hosted GitHub (소스코드 + 마크다운 문서 관리)
✅ Wiki.js — Self-Hosted Notion (웹 기반 문서 편집기)
✅ 자동 동기화 — Wiki.js 작성 → Gitea 자동 백업 (Docs as Code)
```

### 다음 Phase 안내

| Phase | 내용 | 전제 조건 |
|-------|------|-----------|
| **Phase 2-C** | Gitea Actions Runner 배포 (CI/CD) | Phase 2 완료 |
| **Phase 3** | VM 2 — Ollama + Open-WebUI (로컬 AI 서버) | Phase 1 VM 2 준비 |
| **Phase 4** | VM 3 — Mattermost (팀 채팅 서버) | Phase 1 VM 3 준비 |

---

## 📌 자주 쓰는 명령어 모음

```bash
# Pod 상태 전체 확인
kubectl get pods -n dev-tools

# 특정 Pod 실시간 로그 확인
kubectl logs -f <Pod이름> -n dev-tools

# Pod 상세 정보 (오류 원인 파악용)
kubectl describe pod <Pod이름> -n dev-tools

# 서비스 목록 확인
kubectl get svc -n dev-tools

# Deployment 재시작 (Pod 재생성)
kubectl rollout restart deployment/gitea -n dev-tools
kubectl rollout restart deployment/wikijs -n dev-tools
kubectl rollout restart deployment/postgres -n dev-tools

# 모든 리소스 한번에 확인
kubectl get all -n dev-tools
```

---

## 📡 서비스 접속 정보 요약

| 서비스 | 접속 URL | 역할 |
|--------|----------|------|
| **Gitea** | `http://172.28.136.160:30000` | 소스코드 + 마크다운 저장소 |
| **Wiki.js** | `http://172.28.136.160:30005` | 기술 문서 웹 편집기 |

---

> 💡 **기억할 핵심 포인트**
> 1. Wiki.js에서 경로에 `/`(슬래시)를 넣으면 Gitea에 자동으로 폴더가 생성됩니다.
> 2. `home.md`는 반드시 경로를 `home`으로 해야 메인 페이지로 인식됩니다.
> 3. 이제 Gitea 저장소만 백업하면 모든 위키 내용이 안전하게 보관됩니다.

---

*작성일: 2026-04-20 | 버전: v1.0*  
*작성자: Kim Jong-in (Technical Architect)*  
*기반 자료: Full_IT_Infrastructure_Roadmap_v3.md · 서버 명령어 History · SOP · 브라우저 캡처 화면*
