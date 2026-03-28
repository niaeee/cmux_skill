# 공유 스킬 패키지 — 설치 가이드

> 다른 AI(Claude Code, GLM, MiniMax 등)가 이 파일을 읽고 완벽히 설치할 수 있도록 작성됨.

## 포함된 스킬

| 스킬 | 용도 | 필수 |
|------|------|------|
| **cmux-orchestrator** | cmux 멀티 AI 오케스트레이션 (15+ surface 동시 제어) | ✅ |
| **cmux-watcher** | surface 실시간 감시 (IDLE/ERROR/STALL 감지) | ✅ |
| agent-selector-pack.zip | 124개 도메인 전문 에이전트 선택 | 선택 |
| agent-team-forge-v3.0.zip | 에이전트 팀 자동 생성 | 선택 |
| apple-neural-engine | Apple Neural Engine OCR/Vision | 선택 |
| mflux-image-forge | 로컬 AI 이미지 생성 | 선택 |

## 설치 방법

### 방법 1 : 자동 설치 (권장)

```bash
# 1. 압축 해제 (zip으로 받은 경우)
unzip cmux-orchestrator-watcher-pack.zip

# 2. install.sh 실행 — 모든 것이 자동으로 설치됨
bash cmux-orchestrator/install.sh
```

이 명령이 수행하는 것:
1. `cmux-watcher/` → `~/.claude/skills/cmux-watcher/` 자동 복사
2. `cmux-orchestrator/` → `~/.claude/skills/cmux-orchestrator/` 자동 복사
3. `hooks/` 18개 → `~/.claude/hooks/` 심링크
4. `settings.json`에 Hook 18개 자동 등록
5. 실행 권한 자동 설정

### 방법 2 : 수동 설치

```bash
# 1. 스킬 복사
mkdir -p ~/.claude/skills
cp -r cmux-orchestrator ~/.claude/skills/
cp -r cmux-watcher ~/.claude/skills/

# 2. Hook 심링크
mkdir -p ~/.claude/hooks
for f in ~/.claude/skills/cmux-orchestrator/hooks/*; do
  ln -sf "$f" ~/.claude/hooks/$(basename "$f")
done
for f in ~/.claude/skills/cmux-watcher/hooks/*; do
  ln -sf "$f" ~/.claude/hooks/$(basename "$f")
done

# 3. 실행 권한
chmod +x ~/.claude/skills/cmux-orchestrator/hooks/*.sh
chmod +x ~/.claude/skills/cmux-orchestrator/hooks/*.py
chmod +x ~/.claude/skills/cmux-orchestrator/scripts/*.sh
chmod +x ~/.claude/skills/cmux-orchestrator/scripts/*.py

# 4. settings.json에 Hook 등록은 install.sh가 자동으로 해줌
bash ~/.claude/skills/cmux-orchestrator/install.sh
```

## 전제 조건

| 요구사항 | 확인 방법 |
|---------|----------|
| cmux 설치 | `cmux --version` |
| Claude Code CLI | `claude --version` |
| Python 3.11+ | `python3 --version` |
| Node.js 18+ | `node --version` |

## 설치 후 확인

```bash
# Hook 심링크 확인
ls -la ~/.claude/hooks/cmux-*.sh
ls -la ~/.claude/hooks/cmux-*.py

# settings.json Hook 등록 확인
cat ~/.claude/settings.json | python3 -c "
import json, sys
hooks = json.load(sys.stdin).get('hooks', {})
cmux_count = sum(1 for event in hooks.values() for h in event if 'cmux' in h.get('command', ''))
print(f'cmux Hook {cmux_count}개 등록됨')
"

# cmux 동작 확인
cmux tree --all
```

## 제거

```bash
bash ~/.claude/skills/cmux-orchestrator/install.sh --uninstall
```

## 파일 구조

```
공유스킬/
├── cmux-orchestrator/
│   ├── SKILL.md              (스킬 정의 — 802줄)
│   ├── install.sh            (자동 설치 스크립트)
│   ├── activation-hook.sh    (스킬 활성화 Hook)
│   ├── hooks/                (18개 Hook 스크립트)
│   │   ├── cmux-init-enforcer.py
│   │   ├── cmux-watcher-notify-enforcer.py
│   │   ├── cmux-no-stall-enforcer.py
│   │   ├── cmux-gate6-agent-block.sh
│   │   ├── cmux-workflow-state-machine.py
│   │   └── ... (13개 더)
│   ├── scripts/              (28개 유틸리티 스크립트)
│   │   ├── eagle_watcher.sh  (surface 상태 감시)
│   │   ├── gate-blocker.sh   (커밋 게이트)
│   │   ├── read-surface.sh   (surface 화면 읽기)
│   │   ├── speckit-tracker.py
│   │   └── ... (24개 더)
│   ├── agents/               (3개 에이전트 정의)
│   ├── commands/             (cmux 명령어)
│   ├── config/               (설정)
│   └── references/           (14개 참고 문서)
├── cmux-watcher/
│   ├── SKILL.md              (와쳐 스킬 정의)
│   ├── activation-hook.sh
│   ├── scripts/              (감시 스크립트)
│   │   ├── watcher-scan.py
│   │   ├── surface-monitor.py
│   │   └── vision-stall-detector.py
│   ├── hooks/
│   ├── commands/
│   └── references/
├── README.md                 (이 파일)
└── cmux-orchestrator-watcher-pack.zip
```

## 트러블슈팅

| 증상 | 원인 | 해결 |
|------|------|------|
| `cmux: command not found` | cmux 미설치 | cmux 설치 필요 |
| Hook 에러 반복 | settings.json 미등록 | `bash install.sh` 재실행 |
| `Permission denied` | 실행 권한 없음 | `chmod +x ~/.claude/hooks/*` |
| 와쳐가 작동 안 함 | cmux-watcher 미설치 | `install.sh`가 자동 설치 |
| `surface:29 not found` | 하드코딩된 surface 번호 | `/tmp/cmux-roles.json`에서 동적 감지 (v4.1+) |
