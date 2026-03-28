# cmux-orchestrator — 멀티 AI 오케스트레이션 스킬

> **버전** : v7.3 (2026-03-26)
> **호환** : Claude Code v2.1+ / cmux CLI
> **파일 수** : 34개 (스킬 + 에이전트 + 스크립트 + Hook + 레퍼런스)

---

## 이게 뭐하는 건가요?

**cmux 터미널에서 여러 AI를 동시에 부려먹는** 오케스트레이션 스킬입니다.

- Claude (Opus)가 **지휘관**, 다른 AI(Codex, Gemini, GLM, MiniMax 등)가 **워커**
- 조사, 코딩, 리뷰를 병렬로 분배하고 결과를 수집
- **8개 GATE 시스템**으로 물리적 강제 — 미완료 커밋 차단, 수집 전 결론 금지, --workspace 강제 등
- Eagle Watcher가 **API 0원**으로 모든 surface 상태를 실시간 감시
- speckit-tracker로 **태스크 단위 추적** (등록 → 완료 → 재배정)
- 워크트리 기반 **안전한 병렬 작업** (2+ surface 동시 작업 시 파일 충돌 방지)

---

## 설치 (1분)

### 전제 조건

| 소프트웨어 | 최소 버전 | 확인 명령어 | 설치 |
|-----------|----------|------------|------|
| **cmux** | 최신 | `cmux --version` | https://openclaw.ai |
| **Claude Code** | v2.1+ | `claude --version` | `npm install -g @anthropic-ai/claude-code` |
| **Python 3** | 3.6+ | `python3 --version` | OS 기본 포함 |
| **Bash** | 4.0+ | `bash --version` | macOS/Linux 기본 포함 |

### 설치 실행

```bash
# 1. zip 압축 해제
unzip cmux-orchestrator.zip

# 2. 설치
cd cmux-orchestrator
bash install.sh
```

설치 스크립트가 자동으로:
1. cmux / Python3 / Claude Code 전제 조건 확인
2. `~/.claude/skills/cmux-orchestrator/`에 28개 파일 복사
3. `~/.claude/agents/`에 에이전트 3개 설치
4. 스크립트 실행 권한 설정 (`chmod +x`)
5. `~/.claude/settings.json`에 Hook 5개 자동 등록
6. 전체 설치 검증

### 설치 옵션

```bash
bash install.sh              # 전체 설치 (기본)
bash install.sh --hooks-only # 훅만 재등록 (스킬 이미 설치된 경우)
bash install.sh --check      # 설치 상태 확인
bash install.sh --uninstall  # 제거
```

### 설치 경로

| 대상 | 기본 경로 | 변경 가능 |
|------|----------|----------|
| 스킬 | `~/.claude/skills/cmux-orchestrator/` | `SKILL_INSTALL_DIR` 환경변수 |
| 에이전트 | `~/.claude/agents/` | 고정 |
| Hook 설정 | `~/.claude/settings.json` | 자동 |

---

## 사용법

### 기본 명령어

```
/cmux              — 전체 surface 상태 확인
/cmux 조사 [주제]  — 모든 AI에 조사 분배
/cmux 배정 [작업]  — AI별 태스크 분해 + 배정
/cmux 수집         — 결과 수집 (GATE 0 검증)
/cmux 리뷰         — 코드 리뷰 (서브에이전트 자동)
/cmux 커밋         — 안전 커밋 (GATE 1-5 통과 필수)
/cmux 초기화       — cmux 환경 설정 + Hook 등록
/cmux eagle        — surface 상태 감시 실행
/cmux gate         — GATE 검증 전체 실행
/cmux 에러         — ERROR surface 탐지 + 재배정
/cmux 다음 라운드  — 6-Step 라운드 프로토콜 자동 실행
```

### HARD GATE 시스템 (물리적 강제)

| GATE | 규칙 | 강제 수단 |
|------|------|----------|
| **GATE 0** | 수집 완료 전 결론/구현 금지 | gate-enforcer.py 디스패치 레지스트리 |
| **GATE 0.5** | "붙여넣어주세요" 절대 금지 | cmux send 강제 |
| **GATE 1** | WORKING surface 있으면 커밋 금지 | gate-blocker.sh PreToolUse block |
| **GATE 2** | Main 직접 코드리뷰 금지 | 서브에이전트 강제 |
| **GATE 5** | 미완료 태스크 스킵 금지 | speckit-tracker.py |
| **GATE 6** | IDLE surface 있으면 서브에이전트 금지 | cmux send 우선 위임 |
| **GATE 7** | 2+ surface 동시 작업 시 워크트리 필수 | git worktree 강제 |
| **GATE 8** | 다른 workspace surface에 --workspace 필수 | cmux 명령 파라미터 강제 |

### GATE 8 상세 (v7.2 신규)

다른 workspace의 surface에 접근할 때 `--workspace` 파라미터 없이 cmux 명령 실행 금지:
```bash
# 올바른 사용 (다른 workspace)
cmux send --workspace "workspace:2" --surface "surface:5" "내용"
cmux read-screen --workspace "workspace:3" --surface "surface:7" --lines 20

# 현재 workspace (생략 가능)
cmux send --surface "surface:2" "내용"

# 잘못된 사용 (다른 workspace인데 --workspace 없음)
cmux send --surface "surface:5" "내용"  # Error: Surface is not a terminal
```

### 강제 메커니즘 (Hook 자동 등록)

| Hook | 이벤트 | 스크립트 | 역할 |
|------|--------|---------|------|
| **PreToolUse** | git commit 시도 | gate-blocker.sh | GATE 0/1/2/5 위반 시 커밋 물리적 차단 |
| **PostToolUse** | Edit/Write 사용 | gate-enforcer.py | Main 코드 수정 시 IDLE surface 위임 경고 |
| **SessionStart** | 세션 시작 | cmux-orchestra-enforcer.sh | cmux 환경 자동 감지 + 온보딩 |
| **SessionStart** | 세션 시작 | cmux-claude-bridge.sh | cmux 사이드바 상태 연동 |
| **UserPromptSubmit** | 매 메시지 | cmux-idle-reminder.sh | IDLE surface 자동 알림 |

### Eagle Watcher (감시 루프)

```bash
# 수동 1회 실행
bash scripts/eagle_watcher.sh --once

# 백그라운드 루프 (20초 간격)
bash scripts/eagle_watcher.sh &

# 출력: /tmp/cmux-eagle-status.json
cat /tmp/cmux-eagle-status.json | python3 -m json.tool
```

Eagle Watcher는 **API 0원** (순수 bash + cmux read-screen):
- 모든 surface의 WORKING/IDLE/ERROR/WAITING 상태 감지
- cmux 공식 기능 활용 (set-progress, rename-tab, trigger-flash, display-message)
- 30분+ 오래된 버퍼 자동 정리

### Surface Dispatcher (상태 일괄 확인)

```bash
bash scripts/surface-dispatcher-v6.sh "1 2 3 5 10"
# 출력: S:1=DONE S:2=WORK(3m) S:3=IDLE S:5=RATE_LIM S:10=ACTIVE
```

| 상태 | 의미 | 행동 |
|------|------|------|
| DONE | 작업 완료 | /new + 다음 작업 재배정 |
| WORK(Xm) | 작업 중 (X분 경과) | 대기 |
| ENDED(Xm) | 끝났으나 DONE 미출력 | /clear + 재배정 |
| QUEUED | 큐 멈춤 | enter 전송 또는 /clear + 재전송 |
| COMPACT! | 컨텍스트 위험 | 즉시 /clear + 재배정 |
| RATE_LIM | API 제한 | 리셋까지 대기 |
| ACTIVE | 활성 (시간 미파싱) | 대기 |
| IDLE | 대기 중 | 새 작업 배정 |

### Speckit Tracker (태스크 추적)

```bash
# 라운드 초기화
python3 scripts/speckit-tracker.py --init "Round 1"

# 태스크 등록
python3 scripts/speckit-tracker.py --add T1 surface:3 "epub_export"

# 완료 마킹
python3 scripts/speckit-tracker.py --done T1

# 상태 확인
python3 scripts/speckit-tracker.py --status

# GATE 5 검증 (미완료 있으면 exit 1)
python3 scripts/speckit-tracker.py --gate

# 재배정 (실패 → 다른 surface로)
python3 scripts/speckit-tracker.py --fail T1 "sandbox 제약"
python3 scripts/speckit-tracker.py --reassign T1 surface:5
```

---

## 포함된 파일 (34개)

### 핵심
| 파일 | 줄 수 | 설명 |
|------|-------|------|
| `SKILL.md` | 840 | 전체 오케스트레이션 명세 (GATE 0-8, Phase 0-6, 프로토콜) |
| `commands/cmux.md` | 186 | /cmux 명령어 라우팅 정의 |
| `config/orchestra-config.json` | 36 | AI 프리셋 (workspace/quit/start/reset 명령어 포함) |

### 에이전트 (3개)
| 파일 | 모델 | 역할 |
|------|------|------|
| `cmux-git.md` | Haiku | Git 커밋/푸시 |
| `cmux-reviewer.md` | Sonnet | 코드 리뷰 (A/B 테스트 검증) |
| `cmux-security.md` | Sonnet | 보안 감사 (OWASP Top 10) |

### 스크립트 (13개) — Hook + 감시 루프 + 추적기
| 파일 | 줄 수 | 역할 |
|------|-------|------|
| `gate-blocker.sh` | 129 | **PreToolUse Hook** — 커밋 물리적 차단 |
| `gate-enforcer.py` | 285 | **PostToolUse Hook** — GATE 0/5/6 디스패치 레지스트리 검증 |
| `gate-checker.sh` | 150 | GATE 상태 일괄 검증 스크립트 |
| `eagle_watcher.sh` | 631 | **감시 루프** — API 0원, WAITING 감지, cmux 공식 기능 연동 |
| `eagle_analyzer.py` | 515 | Screen text → 상태 분류기 (14 테스트 내장) |
| `speckit-tracker.py` | 743 | 태스크 추적 풀스택 (등록/완료/재배정/GATE 5 검증/실패 추적) |
| `detect_surfaces.py` | 112 | cmux tree 파싱 → AI별 surface 매핑 |
| `install_agents.sh` | 224 | 에이전트 + Hook 설치 |
| `cmux-orchestra-enforcer.sh` | 33 | **SessionStart Hook** — 자동 활성화 |
| `cmux-idle-reminder.sh` | 69 | **UserPromptSubmit Hook** — IDLE 알림 |
| `cmux-claude-bridge.sh` | 55 | **SessionStart/Stop Hook** — cmux 사이드바 연동 |
| `surface-dispatcher.sh` | 57 | Surface 상태 일괄 확인 (v5) |
| `surface-dispatcher-v6.sh` | 71 | Surface 상태 일괄 확인 (v6 개선) |

### 레퍼런스 (14개)
| 파일 | 줄 수 | 내용 |
|------|-------|------|
| `cmux-commands-full.md` | 231 | cmux 전체 명령어 (85개+, workspace 명령 포함) |
| `circuit-breaker.md` | 165 | 529 에러 방지 Circuit Breaker |
| `eagle-patterns.md` | 221 | Eagle Watcher 아키텍처 |
| `dispatch-templates.md` | 300 | 태스크→큐 변환 + Wave 분할 |
| `error-recovery.md` | 166 | 에러 복구 프로토콜 (워크스페이스 범위 에러 대응 추가) |
| `gate-enforcement.md` | 130 | GATE 강제 체계 4중 레이어 (L0~L3) |
| `octopus-concepts.md` | 124 | Octopus 아키텍처 개념 |
| `onboarding-detail.md` | 212 | 온보딩 상세 절차 |
| `polling-selfrenew.md` | 85 | 2분 폴링 + Self-Renewing Loop (자동 핸드오프) |
| `rate-limit-handling.md` | 110 | GLM/Codex rate limit 대응 (429, CST→KST) |
| `subagent-definitions.md` | 256 | 서브에이전트 정의 + 스킬 최적화 |
| `worker-stuck-recovery.md` | 95 | 워커 멈춤 복구 (5분 타임아웃 → escape → /clear) |
| `workspace-gate.md` | 115 | GATE 8 --workspace 강제 상세 |
| `worktree-workflow.md` | 105 | Git worktree 병렬 작업 (MiniMax 제한 주의사항 포함) |

---

## 핵심 개념

### 역할 분담
```
Main (Claude Opus) : 계획 + 판단 + 취합 + 커밋 (직접 코딩 금지!)
워커 (다른 AI들)  : 조사 + 분석 + 구현 + 탐색 (cmux send로만 통신)
```

### 워크플로우 (Phase -1 → 6)
```
Phase -1: 온보딩 (최초 1회 — AI 매핑 + 프리셋 설정)
Phase  0: Eagle 부트 + surface 상태 확인
Phase  1: 작업 분해 + 작업 큐 생성 (speckit 사용)
Phase  2: cmux send로 워커에 배포 (워크트리 병렬, --workspace 필수)
Phase  3: 2분 폴링 → DONE 감지 → 병합 → 재배정
Phase  4: 장애 대응 (에러 복구, rate limit, 컨텍스트 초과)
Phase  5: 테스트 + 코드리뷰 (서브에이전트 필수)
Phase  6: 커밋 (모든 GATE 통과 후)
```

### DONE 프로토콜
모든 워커 AI는 작업 완료 시:
```
요약: auth.ts에 JWT 미들웨어 추가, 3개 파일 수정.

DONE

DONE
```
- DONE 2회 출력 (화면 짤림 방지)
- DONE 확인 전 /new 금지, 새 프롬프트 전송 금지

---

## 문제 해결

| 증상 | 해결 |
|------|------|
| gate-blocker.sh permission denied | `chmod +x ~/.claude/skills/cmux-orchestrator/scripts/*.sh` |
| Hook 미작동 | `bash install.sh --hooks-only` |
| cmux: command not found | https://openclaw.ai 에서 cmux 설치 |
| surface 상태 UNKNOWN | `cmux tree --all`로 접근 가능 surface 확인 |
| Surface is not a terminal | `--workspace "workspace:N"` 추가 (GATE 8) |
| 설치 재실행 | `bash install.sh` (기존 설정 자동 백업) |
| 전체 제거 | `bash install.sh --uninstall` |

---

## FAQ

**Q: cmux 없이도 쓸 수 있나요?**
A: 아니요. cmux CLI 필수. cmux 없으면 모든 Hook이 자동 비활성화됩니다.

**Q: 지원 AI는?**
A: cmux로 접근 가능한 모든 AI CLI: Claude Code, Codex, Gemini CLI, GLM, MiniMax, OpenCode 등.

**Q: surface 번호는 어떻게?**
A: `cmux tree --all`로 확인. 세션마다 다를 수 있으므로 항상 확인 후 사용.

**Q: --workspace는 언제 필요?**
A: 다른 workspace의 surface에 접근할 때 필수 (GATE 8). 같은 workspace면 생략 가능.

**Q: orchestra-config.json은 공유 가능?**
A: presets(AI별 시작/종료 명령어)만 유효. surfaces(번호)는 세션마다 다름.

---

## v7.3 변경 이력 (v7.2 대비)

### 신규 파일 6개
- `references/gate-enforcement.md` — GATE 강제 4중 레이어 (L0~L3)
- `references/polling-selfrenew.md` — 2분 폴링 + Self-Renewing Loop
- `references/rate-limit-handling.md` — GLM/Codex rate limit 대응
- `references/worker-stuck-recovery.md` — 워커 멈춤 복구 프로토콜
- `references/workspace-gate.md` — GATE 8 --workspace 강제
- `scripts/gate-checker.sh` — GATE 상태 일괄 검증

### 대폭 강화된 스크립트
| 파일 | 변화 | 내용 |
|------|------|------|
| `eagle_watcher.sh` | 220→631줄 | WAITING 감지, cmux 공식 기능 연동, 오래된 버퍼 정리 |
| `gate-enforcer.py` | 34→285줄 | 디스패치 레지스트리 기반 GATE 0/5/6 완전 검증 |
| `speckit-tracker.py` | 151→743줄 | 라운드 초기화, 재배정, GATE 5 검증, 실패 추적 |

### SKILL.md 핵심 변경
- **GATE 8 신설** : 다른 workspace surface 접근 시 --workspace 강제
- **MiniMax 워크트리 제한** 문서화 및 대응 방안
- **모든 Phase에 --workspace 파라미터 반영**
- **Self-Renewing Loop** : 컨텍스트 50%+ 시 자동 핸드오프
- **에러 복구 30줄 추가** : 워크스페이스 범위 에러 대응

---

*마지막 업데이트 : 2026-03-26 (v7.3)*
