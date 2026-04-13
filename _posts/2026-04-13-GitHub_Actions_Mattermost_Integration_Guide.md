---
layout: post
title:  "GitHub Actions ↔ Mattermost Webhook 전체 통합 가이드"
date:   2026-04-13 10:00:00 +0900
---

# 🔗 GitHub Actions ↔ Mattermost Webhook 전체 통합 가이드

> **모든 시스템을 하나로 — CI/CD 알림 · 이슈 자동화 · 문서 동기화 완성**  
> TA/개발자를 위한 GitHub Actions · Mattermost · Wiki.js · GitHub Projects 완전 통합 매뉴얼

---

## 📋 목차

1. [전체 통합이란 무엇인가](#1-전체-통합이란-무엇인가)
2. [사전 요구사항 — 통합 전 체크리스트](#2-사전-요구사항--통합-전-체크리스트)
3. [Mattermost Incoming Webhook 생성](#3-mattermost-incoming-webhook-생성)
4. [GitHub Secrets 등록](#4-github-secrets-등록)
5. [통합 1 — 빌드·배포 결과 알림 (Actions → Mattermost)](#5-통합-1--빌드배포-결과-알림-actions--mattermost)
6. [통합 2 — 이슈 알림 (GitHub Issues → Mattermost)](#6-통합-2--이슈-알림-github-issues--mattermost)
7. [통합 3 — PR 알림 (Pull Request → Mattermost)](#7-통합-3--pr-알림-pull-request--mattermost)
8. [통합 4 — 이슈 Close → 칸반 Done 자동 이동 (Projects 자동화)](#8-통합-4--이슈-close--칸반-done-자동-이동-projects-자동화)
9. [통합 5 — Wiki.js 문서 → GitHub 저장소 자동 동기화](#9-통합-5--wikijs-문서--github-저장소-자동-동기화)
10. [통합 워크플로우 전체 통합본](#10-통합-워크플로우-전체-통합본)
11. [통합 완성 후 자동화 흐름 검증](#11-통합-완성-후-자동화-흐름-검증)
12. [운영 치트시트](#12-운영-치트시트)
13. [문제 해결 (Troubleshooting)](#13-문제-해결-troubleshooting)

---

## 1. 전체 통합이란 무엇인가

Step 5는 지금까지 개별로 구축한 모든 시스템을 **하나의 자동화 파이프라인으로 연결**하는 최종 단계입니다.  
각 시스템이 독립적으로 동작하던 것을 Webhook과 GitHub Actions로 연결하여, 개발 이벤트 하나가 전체 시스템에 자동으로 전파됩니다.

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
│ CI/CD 실행   │ Self-hosted Runner                        │
│ 전체 통합     │ ✅ Webhook 연동 ← 본 문서                  │
└──────────────┴──────────────────────────────────────────┘
```

### 통합 전 vs 통합 후 비교

| 항목 | 통합 전 | 통합 후 |
|------|---------|---------|
| **빌드 결과 확인** | GitHub Actions 탭 직접 방문 | Mattermost 채널 자동 알림 |
| **이슈 생성 공유** | 팀원에게 수동으로 알림 | #dev-alert 채널 자동 전송 |
| **PR 리뷰 요청** | 담당자에게 직접 연락 | Mattermost 채널 자동 알림 |
| **태스크 완료 처리** | 칸반 보드 수동 이동 | PR 머지 시 Done 자동 이동 |
| **문서 백업** | 수동 Export 또는 복사 | GitHub 저장소 자동 커밋 |

### 통합 완성 후 전체 자동화 흐름

```
개발자 이슈 생성
    → #dev-alert 채널에 이슈 알림 자동 전송              ← 통합 2

개발자 브랜치 생성 후 코드 push
    → Self-hosted Runner 빌드·테스트 자동 실행
    → 빌드 결과(성공/실패) Mattermost 자동 알림          ← 통합 1

PR 오픈 / 리뷰 요청
    → #dev-alert 채널에 PR 알림 자동 전송               ← 통합 3
    → 리뷰어에게 Mattermost 실시간 알림

PR 머지 (main 브랜치)
    → 자동 배포 실행
    → 배포 결과 Mattermost 알림                         ← 통합 1
    → 연결된 이슈 자동 Close
    → GitHub Projects 칸반 → Done 자동 이동             ← 통합 4

Wiki.js 문서 작성/수정
    → GitHub 저장소에 .md 파일 자동 커밋·백업           ← 통합 5
```

---

## 2. 사전 요구사항 — 통합 전 체크리스트

Step 5를 시작하기 전에 아래 항목이 모두 완료되었는지 확인합니다.

### 사전 완료 확인

| 단계 | 항목 | 확인 |
|------|------|------|
| **Step 1** | Mattermost 서버 실행 중 (포트 8065 접속 가능) | ☐ |
| **Step 1** | `#dev-alert` 채널 생성 완료 | ☐ |
| **Step 2** | Wiki.js 서버 실행 중 (포트 3000 접속 가능) | ☐ |
| **Step 2** | Wiki.js ↔ GitHub 저장소 연동 설정 완료 | ☐ |
| **Step 3** | GitHub Projects 칸반 보드 생성 완료 | ☐ |
| **Step 3** | 이슈 템플릿 · PR 템플릿 생성 완료 | ☐ |
| **Step 4** | Self-hosted Runner 등록 및 Idle 상태 확인 | ☐ |
| **Step 4** | `.github/workflows/` 디렉터리 생성 완료 | ☐ |

### 통합에 필요한 정보 사전 수집

```bash
# Mattermost 서버 IP 확인
hostname -I | awk '{print $1}'

# Mattermost 서비스 상태 확인
cd ~/mattermost && sudo docker compose ps

# Wiki.js 서비스 상태 확인
cd ~/wikijs && sudo docker compose ps

# Self-hosted Runner 상태 확인
sudo systemctl status actions.runner.*
```

---

## 3. Mattermost Incoming Webhook 생성

GitHub Actions에서 Mattermost로 메시지를 전송하기 위한 **수신 전용 Webhook URL**을 생성합니다.

### 3-1. Incoming Webhooks 활성화 (관리자 설정)

Mattermost 관리자 계정으로 접속 후 활성화합니다.

```
Mattermost 접속 (http://<서버_IP>:8065)
→ 우측 상단 메뉴 (⋮) → System Console
→ Integrations → Integration Management
→ Enable Incoming Webhooks : True
→ [Save]
```

### 3-2. 채널별 Webhook 생성

알림 용도에 따라 채널을 구분하여 Webhook을 생성합니다.

```
Mattermost 메인 화면
→ 좌측 상단 메뉴 (≡) → Integrations
→ Incoming Webhooks → [Add Incoming Webhook]
```

**추천 Webhook 구성:**

| Webhook 이름 | 대상 채널 | 용도 |
|-------------|----------|------|
| `GitHub CI/CD` | `#dev-alert` | 빌드·배포 성공/실패 알림 |
| `GitHub Issues` | `#dev-alert` | 이슈 생성·완료 알림 |
| `GitHub PR` | `#dev-alert` | PR 오픈·머지·리뷰 요청 알림 |

> 💡 **팁**: 채널을 하나로 통합하면 관리가 쉽습니다. 필요에 따라 채널을 분리해도 됩니다.

### 3-3. Webhook URL 복사 및 저장

```
[Add Incoming Webhook] 입력 화면:
  Title      : GitHub CI/CD Notifications
  Description: GitHub Actions 빌드·배포 알림
  Channel    : #dev-alert
  [Save]

→ 생성된 Webhook URL 복사 (다음 단계에서 사용)
  예) http://192.168.93.138:8065/hooks/xxxxxxxxxxxxxxxxxx
```

> ⚠️ **주의**: Webhook URL은 생성 직후에만 전체 확인 가능합니다. 반드시 즉시 복사하여 안전한 곳에 보관합니다.

### 3-4. Webhook 동작 테스트

생성 후 서버 터미널에서 즉시 테스트합니다.

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d '{"text": "✅ Webhook 연결 테스트 성공! GitHub Actions 알림이 정상 동작합니다."}' \
  http://<서버_IP>:8065/hooks/xxxxxxxxxxxxxxxxxx
```

**Mattermost `#dev-alert` 채널에 테스트 메시지가 수신되면 정상입니다.**

---

## 4. GitHub Secrets 등록

워크플로우 YAML 파일에 Webhook URL을 직접 노출하지 않도록 **GitHub Secrets**에 안전하게 저장합니다.

### 4-1. Repository Secrets 등록

```
GitHub 저장소 → Settings → Secrets and variables → Actions
→ [New repository secret]
```

| Secret 이름 | 값 | 설명 |
|------------|-----|------|
| `MATTERMOST_WEBHOOK_URL` | `http://<서버_IP>:8065/hooks/xxxx` | Mattermost 수신 Webhook URL |

### 4-2. Organization Secrets 등록 (여러 저장소 공유 시 권장)

여러 저장소에서 동일한 Webhook을 사용할 경우 Organization 레벨에 등록합니다.

```
GitHub Organization → Settings → Secrets and variables → Actions
→ [New organization secret]
→ Name  : MATTERMOST_WEBHOOK_URL
→ Value : http://<서버_IP>:8065/hooks/xxxx
→ Repository access : All repositories 또는 선택 저장소
→ [Add secret]
```

### 4-3. Secrets 등록 확인

```
GitHub 저장소 → Settings → Secrets and variables → Actions
→ Repository secrets 목록에 MATTERMOST_WEBHOOK_URL 표시 확인
```

> Secrets 값은 등록 후 다시 볼 수 없으며, 워크플로우에서 `${{ secrets.MATTERMOST_WEBHOOK_URL }}` 형식으로 참조합니다.

---

## 5. 통합 1 — 빌드·배포 결과 알림 (Actions → Mattermost)

코드 push 및 PR 머지 시 빌드·테스트·배포 결과를 Mattermost 채널로 자동 전송합니다.

### 5-1. 빌드 결과 알림 워크플로우

```yaml
# .github/workflows/build-notify.yml
name: Build & Deploy Notification

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main, develop]

jobs:
  build:
    runs-on: self-hosted

    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: 의존성 설치
        run: |
          # 프로젝트 유형에 맞게 변경
          # npm ci                        ← Node.js
          # pip install -r requirements.txt  ← Python
          echo "의존성 설치 단계"

      - name: 빌드
        run: |
          # npm run build                 ← Node.js
          # python -m py_compile *.py    ← Python
          echo "빌드 단계"

      - name: 테스트
        run: |
          # npm test                      ← Node.js
          # pytest                        ← Python
          echo "테스트 단계"

      - name: 배포 (main 브랜치만)
        if: github.ref == 'refs/heads/main' && success()
        run: |
          docker compose -f /opt/myapp/docker-compose.yml pull
          docker compose -f /opt/myapp/docker-compose.yml up -d --remove-orphans

      # ─── Mattermost 알림 ──────────────────────────────────

      - name: 빌드 성공 알림
        if: success()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"✅ **빌드 성공** \`${{ github.repository }}\`\n브랜치: \`${{ github.ref_name }}\`\n커밋: ${{ github.event.head_commit.message }}\n작성자: ${{ github.event.head_commit.author.name }}\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}

      - name: 빌드 실패 알림
        if: failure()
        run: |
          curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"❌ **빌드 실패** \`${{ github.repository }}\`\n브랜치: \`${{ github.ref_name }}\`\n즉시 확인이 필요합니다!\n🔍 [로그 확인](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

### 5-2. Mattermost 수신 메시지 예시

**빌드 성공 시:**
```
✅ 빌드 성공 `my-org/my-app`
브랜치: `main`
커밋: feat: #42 사용자 로그인 JWT 인증 구현
작성자: Kim Jong-in
```

**빌드 실패 시:**
```
❌ 빌드 실패 `my-org/my-app`
브랜치: `develop`
즉시 확인이 필요합니다!
🔍 로그 확인 → (링크)
```

---

## 6. 통합 2 — 이슈 알림 (GitHub Issues → Mattermost)

이슈 생성·완료·담당자 배정 시 Mattermost `#dev-alert` 채널로 자동 알림을 전송합니다.

### 6-1. 이슈 알림 워크플로우

```yaml
# .github/workflows/issue-notify.yml
name: Issue Notification to Mattermost

on:
  issues:
    types: [opened, closed, reopened, assigned]

jobs:
  notify:
    runs-on: self-hosted

    steps:
      - name: 이슈 알림 전송
        run: |
          ACTION="${{ github.event.action }}"
          TITLE="${{ github.event.issue.title }}"
          URL="${{ github.event.issue.html_url }}"
          ASSIGNEE="${{ github.event.issue.assignee.login }}"
          NUMBER="${{ github.event.issue.number }}"

          if [ "$ACTION" = "opened" ]; then
            MSG="🆕 **새 이슈 등록** [#${NUMBER} ${TITLE}](${URL})\n담당자: ${ASSIGNEE:-미지정}\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "closed" ]; then
            MSG="✅ **이슈 완료** [#${NUMBER} ${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "reopened" ]; then
            MSG="🔄 **이슈 재오픈** [#${NUMBER} ${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "assigned" ]; then
            MSG="👤 **담당자 배정** [#${NUMBER} ${TITLE}](${URL})\n담당자: ${ASSIGNEE}\n저장소: \`${{ github.repository }}\`"
          fi

          [ -n "$MSG" ] && curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"$MSG\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

### 6-2. Mattermost 수신 메시지 예시

```
🆕 새 이슈 등록  #42 사용자 로그인 기능 구현
담당자: dev-kim
저장소: `my-org/my-app`

✅ 이슈 완료  #42 사용자 로그인 기능 구현
저장소: `my-org/my-app`
```

---

## 7. 통합 3 — PR 알림 (Pull Request → Mattermost)

PR 오픈·머지·리뷰 요청 이벤트를 Mattermost 채널로 자동 전송합니다.

### 7-1. PR 알림 워크플로우

```yaml
# .github/workflows/pr-notify.yml
name: PR Notification to Mattermost

on:
  pull_request:
    types: [opened, closed, review_requested]

jobs:
  notify:
    runs-on: self-hosted

    steps:
      - name: PR 알림 전송
        run: |
          ACTION="${{ github.event.action }}"
          TITLE="${{ github.event.pull_request.title }}"
          URL="${{ github.event.pull_request.html_url }}"
          MERGED="${{ github.event.pull_request.merged }}"
          REVIEWER="${{ github.event.requested_reviewer.login }}"

          if [ "$ACTION" = "opened" ]; then
            MSG="🔀 **PR 오픈** [${TITLE}](${URL})\n작성자: ${{ github.event.pull_request.user.login }}\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "closed" ] && [ "$MERGED" = "true" ]; then
            MSG="🎉 **PR 머지 완료** [${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "closed" ] && [ "$MERGED" = "false" ]; then
            MSG="🚫 **PR 닫힘(미머지)** [${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "review_requested" ]; then
            MSG="👀 **코드 리뷰 요청** [${TITLE}](${URL})\n리뷰어: ${REVIEWER}\n저장소: \`${{ github.repository }}\`"
          fi

          [ -n "$MSG" ] && curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"$MSG\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

### 7-2. Mattermost 수신 메시지 예시

```
🔀 PR 오픈  [feat] #42 사용자 로그인 JWT 인증 구현
작성자: dev-kim
저장소: `my-org/my-app`

👀 코드 리뷰 요청  [feat] #42 사용자 로그인 JWT 인증 구현
리뷰어: senior-dev
저장소: `my-org/my-app`

🎉 PR 머지 완료  [feat] #42 사용자 로그인 JWT 인증 구현
저장소: `my-org/my-app`
```

---

## 8. 통합 4 — 이슈 Close → 칸반 Done 자동 이동 (Projects 자동화)

PR 머지 시 연결된 이슈가 자동으로 Close되고, GitHub Projects 칸반에서 **Done 열로 자동 이동**되도록 설정합니다.

### 8-1. GitHub Projects 자동화 워크플로우 활성화

```
GitHub Projects → 해당 Project 클릭
→ 우측 상단 [···] 메뉴 → Workflows
```

**활성화할 워크플로우 목록:**

| 워크플로우 | 동작 | 활성화 여부 |
|-----------|------|------------|
| `Item added to project` | 항목 추가 시 Status → Todo 자동 설정 | ✅ 활성화 |
| `Item reopened` | 이슈 재오픈 시 Status → In Progress 자동 변경 | ✅ 활성화 |
| `Item closed` | 이슈·PR Close 시 Status → Done 자동 이동 | ✅ 활성화 |
| `Pull request merged` | PR 머지 시 연결 이슈 Close 처리 | ✅ 활성화 |

**활성화 방법:**
```
각 워크플로우 항목 → [Edit] 클릭
→ 설정 확인 후 [Save and turn on]
```

### 8-2. PR 본문에 이슈 연결 키워드 작성 (필수)

PR 머지 시 이슈 자동 Close는 PR 본문의 **연결 키워드**가 있어야 동작합니다.

```markdown
## 🔗 관련 이슈
Closes #42

## 📝 변경 내용
로그인 JWT 인증 기능 구현
```

**자동 Close 동작 키워드 (대소문자 무관):**

```
Closes #42      ← 가장 권장
Fixes #42
Resolves #42
```

> ⚠️ `Related to #42` 또는 `See #42`는 참조만 되며 자동 Close되지 않습니다.

### 8-3. 자동화 흐름 확인

```
PR 본문에 "Closes #42" 작성
    → PR main 브랜치로 머지
    → Issue #42 자동 Close
    → GitHub Projects 칸반에서 #42 → Done 열 자동 이동
    → Mattermost #dev-alert: "✅ 이슈 완료 #42 ..." 알림 전송  (통합 2)
```

---

## 9. 통합 5 — Wiki.js 문서 → GitHub 저장소 자동 동기화

Wiki.js에서 문서를 작성·수정하면 **GitHub 저장소에 `.md` 파일로 자동 커밋**되어 백업됩니다.

### 9-1. GitHub Personal Access Token (PAT) 생성

Wiki.js가 GitHub에 자동 커밋하려면 PAT가 필요합니다.

```
GitHub → 우측 상단 프로필 클릭 → Settings
→ Developer settings → Personal access tokens → Tokens (classic)
→ [Generate new token (classic)]
```

**토큰 설정:**

| 항목 | 값 |
|------|-----|
| **Note** | `wikijs-git-sync` |
| **Expiration** | No expiration (또는 1년) |
| **Scopes** | `repo` (전체 체크) |

```
[Generate token] → 생성된 토큰 즉시 복사 (재확인 불가)
예) ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### 9-2. Wiki.js Git Storage 연동 설정

```
Wiki.js 관리자 콘솔 접속 (http://<서버_IP>:3000)
→ Administration → Storage → Git
→ [Activate] 클릭
```

**Git Storage 설정값:**

| 항목 | 값 |
|------|-----|
| **Authentication Type** | Basic |
| **Repository URI** | `https://github.com/<ORG>/<REPO>.git` |
| **Branch** | `main` |
| **Username** | GitHub 사용자명 |
| **Password** | 위에서 생성한 PAT 값 |
| **Default Author Email** | `wikijs@example.com` |
| **Default Author Name** | `Wiki.js Bot` |
| **Commit Message** | `docs: Wiki.js auto sync` |

**동기화 방향 설정:**

```
Sync Direction : Bidirectional
                 (Wiki.js ↔ GitHub 양방향 동기화)
```

**설정 저장 후 즉시 동기화 테스트:**

```
[Apply Changes] → [Force Sync] 클릭
→ GitHub 저장소에 .md 파일이 커밋되면 성공
```

### 9-3. 동기화 결과 확인

```bash
# GitHub 저장소 커밋 이력 확인 (CLI)
cd ~/my-repo-local
git pull
git log --oneline -5

# 예상 출력
# a1b2c3d docs: Wiki.js auto sync
# ...
```

또는 GitHub 저장소 웹 화면에서:
```
저장소 → Code 탭 → 커밋 이력에서 "docs: Wiki.js auto sync" 확인
```

---

## 10. 통합 워크플로우 전체 통합본

5가지 통합을 하나의 워크플로우 파일로 통합한 버전입니다.  
워크플로우 파일 수를 최소화하고 싶을 때 사용합니다.

### 10-1. 통합 알림 허브 워크플로우

```yaml
# .github/workflows/notify-all.yml
name: Team Notification Hub

on:
  issues:
    types: [opened, closed, reopened, assigned]
  pull_request:
    types: [opened, closed, review_requested]
  push:
    branches: [main]

jobs:
  # ─── 이슈 알림 ───────────────────────────────────────────
  issue-notify:
    if: github.event_name == 'issues'
    runs-on: self-hosted
    steps:
      - name: 이슈 알림 전송
        run: |
          ACTION="${{ github.event.action }}"
          TITLE="${{ github.event.issue.title }}"
          URL="${{ github.event.issue.html_url }}"
          ASSIGNEE="${{ github.event.issue.assignee.login }}"
          NUMBER="${{ github.event.issue.number }}"

          if [ "$ACTION" = "opened" ]; then
            MSG="🆕 **새 이슈 등록** [#${NUMBER} ${TITLE}](${URL})\n담당자: ${ASSIGNEE:-미지정}\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "closed" ]; then
            MSG="✅ **이슈 완료** [#${NUMBER} ${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "reopened" ]; then
            MSG="🔄 **이슈 재오픈** [#${NUMBER} ${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "assigned" ]; then
            MSG="👤 **담당자 배정** [#${NUMBER} ${TITLE}](${URL})\n담당자: ${ASSIGNEE}\n저장소: \`${{ github.repository }}\`"
          fi

          [ -n "$MSG" ] && curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"$MSG\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}

  # ─── PR 알림 ─────────────────────────────────────────────
  pr-notify:
    if: github.event_name == 'pull_request'
    runs-on: self-hosted
    steps:
      - name: PR 알림 전송
        run: |
          ACTION="${{ github.event.action }}"
          TITLE="${{ github.event.pull_request.title }}"
          URL="${{ github.event.pull_request.html_url }}"
          MERGED="${{ github.event.pull_request.merged }}"
          REVIEWER="${{ github.event.requested_reviewer.login }}"

          if [ "$ACTION" = "opened" ]; then
            MSG="🔀 **PR 오픈** [${TITLE}](${URL})\n작성자: ${{ github.event.pull_request.user.login }}\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "closed" ] && [ "$MERGED" = "true" ]; then
            MSG="🎉 **PR 머지 완료** [${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "closed" ] && [ "$MERGED" = "false" ]; then
            MSG="🚫 **PR 닫힘(미머지)** [${TITLE}](${URL})\n저장소: \`${{ github.repository }}\`"
          elif [ "$ACTION" = "review_requested" ]; then
            MSG="👀 **코드 리뷰 요청** [${TITLE}](${URL})\n리뷰어: ${REVIEWER}\n저장소: \`${{ github.repository }}\`"
          fi

          [ -n "$MSG" ] && curl -s -X POST \
            -H 'Content-Type: application/json' \
            -d "{\"text\": \"$MSG\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}

  # ─── 배포 알림 ───────────────────────────────────────────
  deploy-notify:
    if: github.event_name == 'push' && github.ref == 'refs/heads/main'
    runs-on: self-hosted
    steps:
      - name: 코드 체크아웃
        uses: actions/checkout@v4

      - name: 빌드 및 테스트
        run: |
          echo "빌드·테스트 단계 — 프로젝트에 맞게 변경"
          # npm ci && npm run build && npm test
          # pip install -r requirements.txt && pytest

      - name: 배포
        run: |
          echo "배포 단계 — 프로젝트에 맞게 변경"
          # docker compose -f /opt/myapp/docker-compose.yml up -d

      - name: 배포 시작 알림
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
            -d "{\"text\": \"🔴 **배포 실패** \`${{ github.repository }}\`\n즉시 롤백이 필요합니다!\n🔍 [로그 확인](${{ github.server_url }}/${{ github.repository }}/actions/runs/${{ github.run_id }})\"}" \
            ${{ secrets.MATTERMOST_WEBHOOK_URL }}
```

### 10-2. 워크플로우 파일 구성 권장안

```
.github/
└── workflows/
    ├── build-notify.yml       ← 빌드·배포 결과 알림 (통합 1)
    ├── issue-notify.yml       ← 이슈 알림 (통합 2)
    ├── pr-notify.yml          ← PR 알림 (통합 3)
    └── notify-all.yml         ← 위 3개를 하나로 합친 통합본 (선택)
```

> 💡 **권장**: 파일을 분리하면 각 알림을 독립적으로 비활성화하거나 수정할 수 있어 유지보수가 용이합니다.

---

## 11. 통합 완성 후 자동화 흐름 검증

통합이 올바르게 설정되었는지 실제 이벤트를 발생시켜 전체 흐름을 검증합니다.

### 11-1. 단계별 검증 절차

**① Mattermost Webhook 연결 테스트**

```bash
curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d '{"text": "🧪 **통합 테스트** — Webhook 연결 정상"}' \
  $MATTERMOST_WEBHOOK_URL

# Mattermost #dev-alert 채널에 메시지 수신 확인
```

**② Self-hosted Runner 상태 확인**

```bash
sudo systemctl status actions.runner.*
# Active: active (running) 확인
```

**③ 이슈 생성 → 알림 수신 확인**

```
GitHub 저장소 → Issues → [New issue]
→ 제목: [TEST] 통합 테스트 이슈
→ [Submit new issue]
→ Mattermost #dev-alert 채널에서 "🆕 새 이슈 등록" 메시지 수신 확인
```

**④ 코드 push → 빌드 알림 확인**

```bash
# 테스트 브랜치에서 빈 커밋 push
git checkout -b test/integration
git commit --allow-empty -m "test: 통합 테스트 빌드 트리거"
git push origin test/integration

# GitHub Actions 탭에서 워크플로우 실행 확인
# Mattermost #dev-alert 채널에서 빌드 결과 알림 수신 확인
```

**⑤ PR 오픈 → 알림 확인**

```
GitHub → Pull requests → [New pull request]
→ base: develop ← compare: test/integration
→ PR 본문에 "Closes #<이슈번호>" 입력
→ [Create pull request]
→ Mattermost에서 "🔀 PR 오픈" 알림 수신 확인
```

**⑥ PR 머지 → 이슈 Close + 칸반 이동 확인**

```
PR → [Merge pull request] → [Confirm merge]
→ Mattermost: "🎉 PR 머지 완료" 알림 수신 확인
→ 연결된 이슈 자동 Close 확인
→ GitHub Projects 칸반: 이슈가 Done 열로 이동 확인
→ Mattermost: "✅ 이슈 완료" 알림 수신 확인
```

**⑦ Wiki.js 문서 동기화 확인**

```
Wiki.js (http://<서버_IP>:3000) → 새 페이지 생성 및 저장
→ GitHub 저장소 → Code 탭 → 최근 커밋에 "docs: Wiki.js auto sync" 확인
```

### 11-2. 전체 통합 완성 체크리스트

```
□ Mattermost Webhook URL 생성 및 테스트 완료
□ GitHub Secrets에 MATTERMOST_WEBHOOK_URL 등록 완료
□ build-notify.yml 워크플로우 등록 및 빌드 알림 수신 확인
□ issue-notify.yml 워크플로우 등록 및 이슈 알림 수신 확인
□ pr-notify.yml 워크플로우 등록 및 PR 알림 수신 확인
□ GitHub Projects 자동화 워크플로우 3종 활성화 완료
□ PR 본문 "Closes #이슈번호" 작성 규칙 팀 공유
□ PR 머지 → 이슈 자동 Close → 칸반 Done 이동 확인
□ Wiki.js Git Storage 연동 설정 완료
□ Wiki.js 문서 → GitHub 저장소 자동 커밋 확인
```

---

## 12. 운영 치트시트

### 전체 서비스 상태 일괄 확인

```bash
#!/bin/bash
# 전체 시스템 상태 한 번에 확인

echo "=========================================="
echo "  개발 관리 인프라 상태 확인"
echo "=========================================="

echo ""
echo "[ Mattermost ]"
cd ~/mattermost && sudo docker compose ps

echo ""
echo "[ Wiki.js ]"
cd ~/wikijs && sudo docker compose ps

echo ""
echo "[ Self-hosted Runner ]"
sudo systemctl status actions.runner.* --no-pager | grep -E "Active|Loaded"

echo ""
echo "[ 포트 리스닝 ]"
sudo netstat -tulpn | grep -E "8065|3000"

echo ""
echo "[ 디스크 사용량 ]"
df -h /
du -sh ~/mattermost ~/wikijs ~/actions-runner 2>/dev/null
```

### Webhook URL 연결 빠른 테스트

```bash
# 환경변수로 Webhook URL 설정 후 테스트
WEBHOOK_URL="http://<서버_IP>:8065/hooks/xxxxxxxxxx"

curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d "{\"text\": \"🧪 테스트: $(date '+%Y-%m-%d %H:%M:%S')\"}" \
  $WEBHOOK_URL
```

### GitHub Actions 워크플로우 수동 트리거

```bash
# GitHub CLI 설치 시 (선택)
sudo dnf install -y gh
gh auth login

# 특정 워크플로우 수동 실행
gh workflow run build-notify.yml
gh workflow run issue-notify.yml

# 워크플로우 실행 이력 확인
gh run list --limit 10
```

### 이슈 자동 Close 키워드 빠른 참조

```markdown
# PR 본문 작성 시 사용 (대소문자 무관)
Closes #42          ← 단일 이슈
Closes #42, #43     ← 복수 이슈 (쉼표로 구분)
Fixes #42
Resolves #42
```

---

## 13. 문제 해결 (Troubleshooting)

### ❌ Mattermost 채널에 알림이 수신되지 않을 때

**원인 1: Incoming Webhooks 기능이 비활성화됨**

```
Mattermost → System Console → Integrations → Integration Management
→ Enable Incoming Webhooks : True 확인
→ [Save]
```

**원인 2: GitHub Secrets 값이 잘못 등록됨**

```bash
# Webhook URL 직접 테스트 (서버 터미널에서)
curl -s -X POST \
  -H 'Content-Type: application/json' \
  -d '{"text": "테스트"}' \
  http://<서버_IP>:8065/hooks/xxxxxxxxxx

# ok 응답이 오면 URL 자체는 정상
# GitHub Secrets 값 재확인 및 재등록
```

**원인 3: Self-hosted Runner가 Offline 상태**

```bash
# Runner 서비스 재시작
sudo systemctl restart actions.runner.*

# 상태 재확인
sudo systemctl status actions.runner.*
```

---

### ❌ GitHub Actions 워크플로우가 실행되지 않을 때

**원인 1: 워크플로우 파일 경로/문법 오류**

```bash
# 워크플로우 파일 위치 확인
ls -la .github/workflows/

# YAML 문법 검사 (yamllint 사용)
pip install yamllint --break-system-packages
yamllint .github/workflows/build-notify.yml
```

**원인 2: Actions 권한 설정 문제**

```
GitHub 저장소 → Settings → Actions → General
→ Actions permissions : Allow all actions and reusable workflows
→ Workflow permissions : Read and write permissions
→ [Save]
```

**원인 3: 트리거 브랜치 불일치**

```yaml
# 워크플로우에 설정된 브랜치와 실제 push 브랜치 일치 확인
on:
  push:
    branches: [main, develop]   # 이 브랜치로 push해야 트리거됨
```

---

### ❌ PR 머지 후 이슈가 자동 Close되지 않을 때

**원인: PR 본문에 연결 키워드 누락**

```markdown
# PR 본문 수정 (머지 전)
Closes #42          ← 이 키워드가 PR 본문에 있어야 함

# 주의: PR 제목이 아닌 PR 본문(Description)에 작성해야 함
```

**확인 방법:**
```
PR 페이지 → Description 탭 확인
→ "Closes #42" 텍스트 존재 여부 확인
→ 없다면 [Edit] → 추가 후 재저장
```

---

### ❌ GitHub Projects 칸반이 자동 이동되지 않을 때

```
GitHub Projects → [···] → Workflows
→ Item closed 워크플로우가 활성화(ON)인지 확인
→ [Edit] → [Save and turn on] 재클릭

추가 확인:
→ 이슈가 Project에 연결되어 있는지 확인
   (이슈 우측 사이드바 → Projects 항목)
```

---

### ❌ Wiki.js 문서가 GitHub에 동기화되지 않을 때

**원인 1: PAT 만료 또는 권한 부족**

```
GitHub → Settings → Developer settings → Personal access tokens
→ wikijs-git-sync 토큰 상태 확인
→ 만료 시 새 토큰 생성 후 Wiki.js Git Storage 재설정
```

**원인 2: Wiki.js Git Storage 연동 오류**

```
Wiki.js → Administration → Storage → Git
→ [Test] 버튼으로 연결 상태 확인
→ 오류 메시지 확인 후 설정 재점검
→ [Force Sync] 로 강제 동기화 시도
```

**원인 3: 저장소 권한 문제**

```bash
# PAT에 repo 권한이 있는지 확인
# GitHub → Settings → Developer settings → Personal access tokens
# → wikijs-git-sync 토큰 → repo (Full control of private repositories) 체크 확인
```

---

## 📚 참고 자료

| 자료 | URL |
|------|-----|
| GitHub Actions 워크플로우 문법 | https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions |
| GitHub Actions 이벤트 트리거 | https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows |
| Mattermost Incoming Webhook 문서 | https://developers.mattermost.com/integrate/webhooks/incoming |
| GitHub 이슈 자동 Close 키워드 | https://docs.github.com/en/issues/tracking-your-work-with-issues/linking-a-pull-request-to-an-issue |
| GitHub Projects 자동화 | https://docs.github.com/en/issues/planning-and-tracking-with-projects/automating-your-project |
| Wiki.js Git Storage 설정 | https://docs.requarks.io/storage/git |

---

## 🎉 구축 완료 현황

```
✅ Step 1: Mattermost Self-Hosted 구축     (팀 소통 채널)
✅ Step 2: Wiki.js Self-Hosted 구축        (기술 문서 위키)
✅ Step 3: GitHub Projects 구축            (칸반 · 이슈 · 마일스톤)
✅ Step 4: Self-hosted Runner 등록         (CI/CD 실행 환경)
✅ Step 5: 전체 통합 — Webhook 연동        ← 현재 문서 (자동화 완성)
           (Actions → Mattermost 알림)
           (이슈·PR → 칸반 자동화)
           (Wiki.js → GitHub 동기화)

🎉 엔터프라이즈급 개발 관리 인프라 구축 완료!
   운영 비용: $0 / 월
   데이터 위치: 내 서버 (완전 통제)
```

---

<div align="center">

**본 문서는 소프트웨어 개발 프로젝트의 무료 Self-Hosted 인프라 구축을 목적으로 작성되었습니다.**  
`Oracle Linux 8.10` · `GitHub Actions` · `Mattermost Webhook` · `GitHub Projects` · `Wiki.js`

*작성일: 2026-04-13*  
*작성자: Kim Jong-in (Technical Architect)*

</div>
