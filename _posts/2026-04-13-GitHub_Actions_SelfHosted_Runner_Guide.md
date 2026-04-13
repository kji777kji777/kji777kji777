# ⚙️ GitHub Actions Self-hosted Runner 구축 가이드

> **CI/CD 실행 환경을 내 서버에 — GitHub 무료 제한 없는 빌드·테스트·배포 파이프라인**  
> TA/개발자를 위한 Self-hosted Runner 초기 등록부터 워크플로우 실전 운영까지 완전 매뉴얼

---

## 📋 목차

1. [Self-hosted Runner란 무엇인가](#1-self-hosted-runner란-무엇인가)
2. [사전 요구사항](#2-사전-요구사항)
3. [Runner 다운로드 및 설치](#3-runner-다운로드-및-설치)
4. [GitHub에 Runner 등록](#4-github에-runner-등록)
5. [systemd 서비스 등록 — 자동 실행 설정](#5-systemd-서비스-등록--자동-실행-설정)
6. [Runner 상태 확인 및 검증](#6-runner-상태-확인-및-검증)
7. [워크플로우 작성 — runs-on: self-hosted](#7-워크플로우-작성--runs-on-self-hosted)
8. [실전 워크플로우 예제](#8-실전-워크플로우-예제)
9. [Runner 라벨 및 다중 Runner 구성](#9-runner-라벨-및-다중-runner-구성)
10. [보안 설정 및 운영 권장사항](#10-보안-설정-및-운영-권장사항)
11. [운영 치트시트](#11-운영-치트시트)
12. [문제 해결 (Troubleshooting)](#12-문제-해결-troubleshooting)

---

## 1. Self-hosted Runner란 무엇인가

GitHub Actions Self-hosted Runner는 **내 서버에서 직접 CI/CD 작업을 실행하는 에이전트 프로그램**입니다.  
GitHub이 제공하는 클라우드 Runner 대신 자체 서버를 실행 환경으로 사용하여 빌드·테스트·배포를 무제한으로 수행합니다.

### 개발 인프라 내 역할

```
┌─────────────────────────────────────────────────────────┐
│              개발 관리 인프라 전체 구성                      │
├──────────────┬──────────────────────────────────────────┤
│ 영역          │ 도구                                      │
├──────────────┼──────────────────────────────────────────┤
│ 소스 관리     │ GitHub (버전 관리 / PR 코드 리뷰)            │
│ 팀원 소통     │ Mattermost [Self-Hosted]                  │
│ 문서/공유     │ Wiki.js [Self-Hosted]                     │
│ 일정/태스크   │ GitHub Projects [SaaS · 무료]              │
│ CI/CD 실행   │ ✅ Self-hosted Runner ← 본 문서             │
│ 상태 알림     │ Mattermost Incoming Webhook               │
└──────────────┴──────────────────────────────────────────┘
```

### GitHub-hosted Runner vs Self-hosted Runner 비교

| 항목 | GitHub-hosted Runner | Self-hosted Runner |
|------|---------------------|-------------------|
| **실행 환경** | GitHub 클라우드 서버 | 내 서버 (Oracle Linux) |
| **무료 제한** | 2,000분/월 (Public 무제한) | **무제한** |
| **실행 속도** | 네트워크 상태에 따라 변동 | 내부 네트워크 — 빠름 |
| **내부 서버 접근** | ❌ 불가 | ✅ 가능 (Mattermost·DB 직접 접근) |
| **소프트웨어 커스터마이즈** | 제한적 | ✅ 자유롭게 설치 가능 |
| **데이터 보안** | GitHub 클라우드 통과 | 서버 내부에서 처리 |
| **운영 비용** | 초과 시 유료 | **$0** |

### CI/CD 전체 흐름에서 Runner의 위치

```
개발자 git push / PR 오픈
        │
        ▼
GitHub Actions 워크플로우 트리거
(.github/workflows/*.yml)
        │
        ▼
┌────────────────────────────────────────┐
│  Self-hosted Runner (내 서버에서 실행)  │
│                                        │
│  1. 코드 체크아웃 (actions/checkout)   │
│  2. 의존성 설치 (npm install / pip)    │
│  3. 빌드 (npm run build / gradle)      │
│  4. 테스트 (npm test / pytest)         │
│  5. 배포 (docker compose / rsync)      │
└────────────────────────────────────────┘
        │
        ▼
결과 알림 → Mattermost Webhook
✅ 성공 / ❌ 실패 즉시 채널 전송
```

---

## 2. 사전 요구사항

### 필요 조건

| 항목 | 내용 |
|------|------|
| **서버 OS** | Oracle Linux 8.10 (또는 RHEL 8 계열) |
| **GitHub 계정** | Organization 또는 개인 계정 |
| **GitHub 권한** | 저장소 `Admin` 권한 (Runner 등록용) |
| **네트워크** | 서버 → `github.com` 아웃바운드 HTTPS(443) 허용 |
| **사전 완료** | Step 1 Mattermost, Step 2 Wiki.js, Step 3 GitHub Projects 구축 완료 |

### 서버 사전 패키지 확인

```bash
# curl 설치 확인
curl --version

# tar 설치 확인
tar --version

# git 설치 확인 (워크플로우 내 체크아웃에 필요)
git --version

# 미설치 시 일괄 설치
sudo dnf install -y curl tar git
```

### Runner 전용 사용자 생성 (권장)

Runner를 root가 아닌 전용 계정으로 운영하면 보안이 강화됩니다.

```bash
# runner 전용 사용자 생성
sudo useradd -m -s /bin/bash runner

# docker 그룹에 추가 (워크플로우에서 docker 명령 사용 시)
sudo usermod -aG docker runner

# 사용자 전환 확인
sudo su - runner
whoami   # runner 출력 확인
exit
```

> ⚠️ **주의**: 이후 Runner 설치·등록 작업은 모두 `runner` 사용자로 진행합니다.

---

## 3. Runner 다운로드 및 설치

### 3-1. 작업 디렉터리 생성

```bash
# runner 사용자로 전환
sudo su - runner

# 작업 디렉터리 생성
mkdir -p ~/actions-runner
cd ~/actions-runner
```

### 3-2. Runner 패키지 다운로드

GitHub에서 최신 Runner 버전을 확인하여 다운로드합니다.

```bash
# 최신 버전 확인 (작성 시점 기준: 2.323.0)
# https://github.com/actions/runner/releases 에서 최신 버전 확인 후 교체

RUNNER_VERSION="2.323.0"

# Linux x64 패키지 다운로드
curl -o actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz -L \
  https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz
```

### 3-3. 체크섬 검증 (선택사항 — 권장)

```bash
# SHA256 해시 검증 (GitHub Releases 페이지의 해시값과 비교)
echo "$(sha256sum actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz)"
```

### 3-4. 압축 해제

```bash
tar xzf actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

# 파일 목록 확인
ls -la
```

**압축 해제 후 주요 파일:**

```
~/actions-runner/
├── config.sh       ← Runner 등록 스크립트
├── run.sh          ← 수동 실행 스크립트
├── svc.sh          ← systemd 서비스 관리 스크립트
├── bin/            ← Runner 실행 바이너리
└── externals/      ← 외부 의존성
```

---

## 4. GitHub에 Runner 등록

### 4-1. GitHub에서 등록 토큰 발급

Runner를 등록할 범위에 따라 두 가지 방법이 있습니다.

**방법 A — 저장소(Repository) 단위 등록 (단일 프로젝트용)**

```
GitHub 저장소 → Settings → Actions → Runners
→ [New self-hosted runner] 버튼 클릭
→ OS: Linux / Architecture: x64 선택
→ 화면에 표시된 config.sh 명령어에서 --token 값 복사
```

**방법 B — Organization 단위 등록 (여러 저장소 공유용 · 권장)**

```
GitHub Organization → Settings → Actions → Runners
→ [New self-hosted runner] 버튼 클릭
→ OS: Linux / Architecture: x64 선택
→ 화면에 표시된 config.sh 명령어에서 --token 값 복사
```

> ✅ **권장**: Organization 단위로 등록하면 모든 저장소에서 Runner를 공유할 수 있습니다.

### 4-2. config.sh 실행 — Runner 등록

```bash
# ~/actions-runner 디렉터리에서 실행
cd ~/actions-runner

# 저장소 단위 등록 예시
./config.sh \
  --url https://github.com/<OWNER>/<REPO> \
  --token <REGISTRATION_TOKEN>

# Organization 단위 등록 예시
./config.sh \
  --url https://github.com/<ORG_NAME> \
  --token <REGISTRATION_TOKEN>
```

**실제 등록 예시:**

```bash
./config.sh \
  --url https://github.com/my-org \
  --token AABCDEFG1234567890ABCDEF
```

### 4-3. 등록 시 대화형 입력

`config.sh` 실행 시 아래 항목을 순서대로 입력합니다.

```
Enter the name of the runner group to add this runner to:
[press Enter for Default]
→ Enter (기본값 사용)

Enter the name of runner:
[press Enter for hostname]
→ oracle-linux-runner  (식별하기 쉬운 이름 입력)

This runner will have the following labels: 'self-hosted', 'Linux', 'X64'
Enter any additional labels (ex. production,gpu):
[press Enter to skip]
→ oracle-linux,dev-server  (추가 라벨 입력 또는 Enter)

Enter name of work folder:
[press Enter for _work]
→ Enter (기본값 사용)

✔ Runner successfully added
✔ Runner connection is good
```

### 4-4. 등록 성공 확인

GitHub 웹 브라우저에서 확인:

```
저장소 또는 Organization → Settings → Actions → Runners
→ 등록한 Runner 이름이 목록에 표시되면 성공
→ 상태: Idle (대기 중)
```

---

## 5. systemd 서비스 등록 — 자동 실행 설정

Runner를 수동으로 실행하면 터미널을 닫을 때 중단됩니다.  
`svc.sh` 스크립트로 systemd 서비스에 등록하면 **서버 재부팅 시에도 자동으로 시작**됩니다.

### 5-1. systemd 서비스 설치

```bash
# root 권한 필요 — runner 사용자에서 sudo 사용
cd ~/actions-runner

sudo ./svc.sh install runner
```

> `install runner` 인수는 서비스를 실행할 OS 사용자(runner)를 지정합니다.

### 5-2. 서비스 시작

```bash
sudo ./svc.sh start
```

### 5-3. 자동 시작 활성화 (부팅 시)

```bash
# 서비스 이름 확인
sudo systemctl list-units | grep actions.runner

# 자동 시작 활성화
sudo systemctl enable actions.runner.<서비스명>

# 예시
sudo systemctl enable actions.runner.my-org.oracle-linux-runner
```

### 5-4. svc.sh 명령어 요약

| 명령어 | 설명 |
|--------|------|
| `sudo ./svc.sh install runner` | systemd 서비스로 등록 |
| `sudo ./svc.sh start` | 서비스 시작 |
| `sudo ./svc.sh stop` | 서비스 중지 |
| `sudo ./svc.sh status` | 서비스 상태 확인 |
| `sudo ./svc.sh uninstall` | 서비스 등록 해제 |

---

## 6. Runner 상태 확인 및 검증

### 6-1. 서비스 상태 확인

```bash
# systemd 서비스 상태
sudo systemctl status actions.runner.*

# 예상 출력
● actions.runner.my-org.oracle-linux-runner.service
   Loaded: loaded (/etc/systemd/system/actions.runner.*.service; enabled)
   Active: active (running) since ...
```

### 6-2. 실시간 로그 확인

```bash
# 최근 50줄 로그 확인
journalctl -u actions.runner.* -n 50

# 실시간 로그 스트리밍
journalctl -u actions.runner.* -f
```

**정상 동작 로그 예시:**

```
Listening for Jobs
```

### 6-3. GitHub 웹에서 상태 확인

```
Settings → Actions → Runners
→ 상태: ● Idle     (대기 중 — 정상)
→ 상태: ● Active   (작업 실행 중 — 정상)
→ 상태: ○ Offline  (비정상 — 서비스 재시작 필요)
```

### 6-4. 테스트 워크플로우로 동작 검증

Runner가 정상 동작하는지 간단한 워크플로우로 확인합니다.

```yaml
# .github/workflows/runner-test.yml
name: Self-hosted Runner Test

on:
  workflow_dispatch:   # 수동 실행

jobs:
  test:
    runs-on: self-hosted
    steps:
      - name: Check Runner Environment
        run: |
          echo "=== Runner 환경 정보 ==="
          echo "Hostname    : $(hostname)"
          echo "OS          : $(cat /etc/os-release | grep PRETTY_NAME)"
          echo "User        : $(whoami)"
          echo "Working Dir : $(pwd)"
          echo "Docker      : $(docker --version)"
          echo "Git         : $(git --version)"
```

**수동 실행 방법:**
```
저장소 → Actions → Self-hosted Runner Test → Run workflow
```

---

## 7. 워크플로우 작성 — runs-on: self-hosted

### 7-1. 핵심 설정 — runs-on

Self-hosted Runner를 사용하려면 워크플로우 YAML에서 `runs-on`을 아래와 같이 지정합니다.

```yaml
jobs:
  build:
    runs-on: self-hosted      # 기본 라벨 — 등록된 모든 Self-hosted Runner 사용
```

**특정 Runner만 지정하려면 라벨을 배열로 지정:**

```yaml
jobs:
  build:
    runs-on: [self-hosted, oracle-linux]   # oracle-linux 라벨이 있는 Runner만 선택
```

### 7-2. 워크플로우 기본 구조

```yaml
name: 워크플로우 이름

on:                           # 트리거 이벤트
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  job-name:
    runs-on: self-hosted      # Self-hosted Runner 지정

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: 작업 수행
        run: |
          echo "명령어 실행"
```

### 7-3. 주요 트리거 이벤트

| 트리거 | 설명 | 예시 사용 |
|--------|------|-----------|
| `push` | 브랜치에 push 시 | 빌드·테스트 자동 실행 |
| `pull_request` | PR 생성·업데이트 시 | 코드 품질 검사 |
| `workflow_dispatch` | 수동 실행 | 배포 트리거 |
| `schedule` | 크론 표현식 | 정기 배치 작업 |
| `issues` | 이슈 생성·변경 시 | Mattermost 알림 |

---

## 8. 실전 워크플로우 예제

### 8-1. Node.js 프로젝트 빌드 및 테스트

```yaml
# .github/workflows/node-build.yml
name: Node.js Build & Test

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build:
    runs-on: self-hosted

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: Node.js 버전 확인
        run: node --version && npm --version

      - name: 의존성 설치
        run: npm ci

      - name: 빌드
        run: npm run build

      - name: 테스트 실행
        run: npm test

      - name: 성공 알림 (Mattermost)
        if: success()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"✅ **빌드 성공** \`${{ github.repository }}\` → \`${{ github.ref_name }}\`\n커밋: ${{ github.event.head_commit.message }}\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}

      - name: 실패 알림 (Mattermost)
        if: failure()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"❌ **빌드 실패** \`${{ github.repository }}\` → \`${{ github.ref_name }}\`\n즉시 확인이 필요합니다!\n로그: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

### 8-2. Python 프로젝트 테스트

```yaml
# .github/workflows/python-test.yml
name: Python Test

on:
  push:
    branches: [main, develop]

jobs:
  test:
    runs-on: self-hosted

    steps:
      - uses: actions/checkout@v4

      - name: Python 버전 확인
        run: python3 --version

      - name: 의존성 설치
        run: |
          python3 -m pip install --upgrade pip
          pip install -r requirements.txt

      - name: pytest 실행
        run: pytest --tb=short -v

      - name: 결과 알림
        if: always()
        run: |
          STATUS="${{ job.status }}"
          if [ "$STATUS" = "success" ]; then EMOJI="✅"; else EMOJI="❌"; fi
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"$EMOJI **Python 테스트 ${STATUS}**: \`${{ github.repository }}\`\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

### 8-3. Docker 이미지 빌드 및 배포

```yaml
# .github/workflows/docker-deploy.yml
name: Docker Build & Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: self-hosted

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: Docker 이미지 빌드
        run: |
          docker build -t myapp:${{ github.sha }} .
          docker tag myapp:${{ github.sha }} myapp:latest

      - name: 컨테이너 재시작 (무중단 배포)
        run: |
          docker compose -f docker-compose.prod.yml pull
          docker compose -f docker-compose.prod.yml up -d --remove-orphans

      - name: 배포 성공 알림
        if: success()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"🚀 **배포 완료** \`${{ github.repository }}\`\n커밋: ${{ github.event.head_commit.message }}\n작성자: ${{ github.event.head_commit.author.name }}\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}

      - name: 배포 실패 알림
        if: failure()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"🔴 **배포 실패** \`${{ github.repository }}\`\n즉시 롤백이 필요합니다!\n로그: ${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }}\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

### 8-4. GitHub Secrets 등록 — Mattermost Webhook URL

워크플로우에서 `${{ secrets.MATTERMOST_WEBHOOK_URL }}`을 사용하려면 먼저 Secret을 등록합니다.

```
GitHub 저장소 → Settings → Secrets and variables → Actions
→ [New repository secret]
```

| Secret 이름 | 값 |
|------------|-----|
| `MATTERMOST_WEBHOOK_URL` | `http://<서버_IP>:8065/hooks/xxxxxxxxxxxxxxxx` |

---

## 9. Runner 라벨 및 다중 Runner 구성

### 9-1. 라벨(Label) 개념

라벨은 워크플로우에서 특정 Runner를 선택하기 위한 식별자입니다.  
Runner 등록 시 자동으로 부여되는 기본 라벨과 사용자 지정 라벨을 조합해 사용합니다.

**기본 라벨 (자동 부여):**

| 라벨 | 의미 |
|------|------|
| `self-hosted` | 모든 Self-hosted Runner |
| `Linux` | Linux OS Runner |
| `X64` | x86-64 아키텍처 Runner |

**사용자 지정 라벨 예시:**

| 라벨 | 용도 |
|------|------|
| `oracle-linux` | Oracle Linux 서버 식별 |
| `dev-server` | 개발 서버용 Runner |
| `production` | 운영 배포 전용 Runner |
| `gpu` | GPU 가속 빌드 전용 |

### 9-2. Runner에 라벨 추가 (등록 후)

```
GitHub → Settings → Actions → Runners → 해당 Runner 클릭
→ [Edit] → Labels 항목에 추가
```

또는 `config.sh` 재실행 시 `--labels` 옵션으로 지정:

```bash
./config.sh \
  --url https://github.com/my-org \
  --token <TOKEN> \
  --labels oracle-linux,dev-server,production
```

### 9-3. 다중 Runner 구성 (병렬 빌드)

서버 리소스가 충분할 경우 여러 Runner를 등록해 병렬 빌드가 가능합니다.

```bash
# Runner 1 (이미 설치됨)
~/actions-runner-1/

# Runner 2 추가 설치
mkdir -p ~/actions-runner-2
cd ~/actions-runner-2
tar xzf ../actions-runner-linux-x64-2.323.0.tar.gz

./config.sh \
  --url https://github.com/my-org \
  --token <NEW_TOKEN> \
  --name oracle-linux-runner-2

sudo ./svc.sh install runner
sudo ./svc.sh start
```

> **Runner 토큰**: Runner 추가 등록 시마다 GitHub에서 새 토큰을 발급받아야 합니다. 토큰은 1시간 유효합니다.

---

## 10. 보안 설정 및 운영 권장사항

### 10-1. 방화벽 설정

Self-hosted Runner는 서버 → GitHub 방향의 **아웃바운드** 통신만 필요합니다.  
인바운드 포트를 별도로 열 필요가 없습니다.

```bash
# Runner 동작에 필요한 아웃바운드 허용 확인
# GitHub API: api.github.com (443/tcp)
# GitHub Actions: *.actions.githubusercontent.com (443/tcp)

# 기본적으로 아웃바운드 HTTPS는 허용되어 있음
# 만약 엄격한 아웃바운드 정책이 있다면 아래 도메인을 허용
# - github.com
# - api.github.com
# - objects.githubusercontent.com
# - actions-results-receiver-production.githubapp.com
```

### 10-2. runner 사용자 권한 최소화

```bash
# runner 사용자에게 sudo 권한 부여 시 필요한 명령어만 허용
# /etc/sudoers.d/runner 파일 생성 (권장)

sudo tee /etc/sudoers.d/runner << 'EOF'
runner ALL=(ALL) NOPASSWD: /usr/bin/docker, /usr/bin/docker-compose, /usr/local/bin/docker-compose
EOF

sudo chmod 440 /etc/sudoers.d/runner
```

### 10-3. 작업 디렉터리 정기 정리

Runner 실행 후 `_work` 디렉터리에 빌드 아티팩트가 누적됩니다.

```bash
# 작업 디렉터리 용량 확인
du -sh ~/actions-runner/_work/

# 오래된 작업 디렉터리 정리 (cron 등록 권장)
find ~/actions-runner/_work -maxdepth 2 -type d -mtime +7 -exec rm -rf {} + 2>/dev/null

# cron 등록 예시 (매일 새벽 3시)
crontab -e
# 0 3 * * * find ~/actions-runner/_work -maxdepth 2 -type d -mtime +7 -exec rm -rf {} + 2>/dev/null
```

### 10-4. Public 저장소 Runner 사용 주의

```
⚠️  Public 저장소에 Self-hosted Runner를 등록하면
    외부 사용자의 Fork PR이 Runner에서 실행될 수 있습니다.

안전 설정:
Settings → Actions → General → Fork pull request workflows
→ "Require approval for all outside collaborators" 선택
```

---

## 11. 운영 치트시트

### Runner 서비스 관리 명령어

```bash
# 상태 확인
sudo systemctl status actions.runner.*

# 시작 / 중지 / 재시작
sudo systemctl start   actions.runner.*
sudo systemctl stop    actions.runner.*
sudo systemctl restart actions.runner.*

# 자동 시작 활성화 / 비활성화
sudo systemctl enable  actions.runner.*
sudo systemctl disable actions.runner.*

# 실시간 로그
journalctl -u actions.runner.* -f

# 최근 100줄 로그
journalctl -u actions.runner.* -n 100
```

### Runner 재등록 절차 (토큰 만료 등)

```bash
cd ~/actions-runner

# 1. 서비스 중지 및 제거
sudo ./svc.sh stop
sudo ./svc.sh uninstall

# 2. Runner 등록 해제
./config.sh remove --token <REMOVE_TOKEN>

# 3. 새 토큰으로 재등록
./config.sh \
  --url https://github.com/my-org \
  --token <NEW_REGISTRATION_TOKEN> \
  --name oracle-linux-runner

# 4. 서비스 재등록 및 시작
sudo ./svc.sh install runner
sudo ./svc.sh start
```

### 전체 서비스 상태 일괄 확인

```bash
# Runner + Docker 서비스 한 번에 확인
echo "=== Self-hosted Runner ===" && sudo systemctl status actions.runner.* --no-pager
echo ""
echo "=== Docker Containers ===" && sudo docker ps -a
echo ""
echo "=== Disk Usage ===" && df -h && du -sh ~/actions-runner/_work/
```

---

## 12. 문제 해결 (Troubleshooting)

### ❌ Runner가 GitHub에 Offline으로 표시될 때

**원인 1: 서비스가 중지됨**

```bash
# 서비스 상태 확인
sudo systemctl status actions.runner.*

# 서비스 재시작
sudo systemctl restart actions.runner.*

# 로그에서 오류 원인 확인
journalctl -u actions.runner.* -n 50
```

**원인 2: 네트워크 연결 문제**

```bash
# GitHub 연결 테스트
curl -s https://api.github.com/zen
# "Something pithy" 형태의 문구가 출력되면 연결 정상
```

**원인 3: 등록 토큰 만료**

```bash
# Runner 재등록 (11장 '재등록 절차' 참고)
```

---

### ❌ 워크플로우에서 "No runner matching" 오류가 발생할 때

**원인:** `runs-on` 라벨과 Runner 라벨 불일치

```yaml
# 워크플로우 확인
runs-on: [self-hosted, oracle-linux]   # 두 라벨 모두 있어야 실행됨
```

```
GitHub → Settings → Actions → Runners → Runner 클릭
→ 현재 라벨 목록 확인 후 필요한 라벨 추가
```

---

### ❌ Permission denied — Docker 명령어 실패 시

```bash
# runner 사용자가 docker 그룹에 속해 있는지 확인
groups runner

# docker 그룹에 추가
sudo usermod -aG docker runner

# 적용을 위해 서비스 재시작
sudo systemctl restart actions.runner.*
```

---

### ❌ 빌드 디스크 공간 부족

```bash
# Runner 작업 디렉터리 용량 확인
du -sh ~/actions-runner/_work/

# Docker 이미지·컨테이너·볼륨 정리
sudo docker system prune -f

# 오래된 빌드 캐시 정리
sudo docker builder prune -f

# 작업 디렉터리 강제 정리
rm -rf ~/actions-runner/_work/*
```

---

### ❌ actions/checkout 단계에서 git 오류 발생

```bash
# git이 설치되어 있는지 확인
git --version

# 미설치 시
sudo dnf install -y git

# git 글로벌 설정 (runner 사용자)
sudo su - runner
git config --global user.email "runner@example.com"
git config --global user.name "Self-hosted Runner"
```

---

## 📚 참고 자료

| 자료 | URL |
|------|-----|
| GitHub Actions Self-hosted Runner 공식 문서 | https://docs.github.com/en/actions/hosting-your-own-runners |
| Runner 최신 버전 릴리즈 | https://github.com/actions/runner/releases |
| GitHub Actions 워크플로우 문법 | https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions |
| GitHub Actions 보안 강화 | https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions |

---

## 🗺️ 다음 단계

```
✅ Step 1: Mattermost Self-Hosted 설치       (완료)
✅ Step 2: Wiki.js Self-Hosted 설치           (완료)
✅ Step 3: GitHub Projects 구축              (완료)
✅ Step 4: GitHub Actions Self-hosted Runner  ← 현재 문서
⬜ Step 5: 전체 통합 — Webhook 연동
           (Actions → Mattermost 알림 · 이슈 → 칸반 자동화 · Wiki.js → GitHub 동기화)
```

---

<div align="center">

**본 문서는 소프트웨어 개발 프로젝트의 무료 Self-Hosted 인프라 구축을 목적으로 작성되었습니다.**  
`Oracle Linux 8.10` · `GitHub Actions` · `Self-hosted Runner` · `systemd` · `Docker`

*작성일: 2026-04-13*  
*작성자: Kim Jong-in (Technical Architect)*

</div>
