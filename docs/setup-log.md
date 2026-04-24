# Claude Code Channels + CoinFello 연동 셋업 로그

> 기록 시작: 2026-04-24
> 목표: Telegram → Claude Code Channels → CoinFello CLI 로 크립토 트랜잭션 실행

## 개요

- **Claude Code Channels**: 외부(Telegram 등)에서 실행 중인 Claude Code 세션으로 이벤트를 push
- **CoinFello skill**: `@coinfello/agent-cli`로 smart account 생성, SIWE 로그인, 자연어 prompt로 트랜잭션
- **통합 흐름**: Telegram 메시지 → Claude Code 세션 → CoinFello CLI → 온체인 트랜잭션

## 환경 확인

| 항목 | 값 | 요구사항 | 상태 |
| --- | --- | --- | --- |
| OS | Linux WSL2 (Ubuntu) | - | ✓ |
| Claude Code | 2.1.119 | 2.1.80+ | ✓ |
| Node.js | v24.15.0 | 20+ | ✓ |
| npx | 11.12.1 | 포함 | ✓ |
| Bun | 1.3.13 | 필수 | ✓ (설치함) |
| 인증 | claude.ai | Console/API key 불가 | ✓ |

```bash
# 환경 체크 명령어
claude --version
node --version
bun --version
```

> WSL2/Linux 환경이라 iMessage 채널은 불가 (macOS 전용). Telegram 선택.

---

## 1단계: Bun 설치

Claude Code Channel 플러그인은 Bun 스크립트로 동작.

```bash
curl -fsSL https://bun.sh/install | bash
source ~/.bashrc  # 또는: export PATH="$HOME/.bun/bin:$PATH"
bun --version
# → 1.3.13
```

## 2단계: CoinFello skill 확보

clawhub에서 zip 다운로드:

```bash
curl -sL "https://wry-manatee-359.convex.site/api/v1/download?slug=coinfello" \
  -o /tmp/coinfello-skill.zip
mkdir -p /tmp/coinfello-skill
unzip -o /tmp/coinfello-skill.zip -d /tmp/coinfello-skill/
cp -r /tmp/coinfello-skill /home/nubroo/coinfello_test/coinfello-skill
```

파일 구성:

- `SKILL.md` — 스킬 본문 (CoinFello CLI 사용법)
- `references/REFERENCE.md` — config 스키마, 지원 체인, API
- `_meta.json` — 메타데이터

대안 설치 방법:

- `openclaw skills install brettcleary/coinfello`
- `npx clawhub@latest install coinfello`

## 3단계: Telegram 봇 생성 (사용자 수동 작업)

1. Telegram에서 [@BotFather](https://t.me/BotFather) 검색
2. `/newbot` 전송
3. 봇 display name 입력 (예: "My Coinfello Bot")
4. 봇 username 입력 — **반드시 `bot`으로 끝나야 함** (예: `my_coinfello_bot`)
5. BotFather가 돌려주는 **HTTP API token** 복사
   - 형식: `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`
   - 이 토큰은 봇 제어권 = 외부 유출 금지

## 4단계: Telegram 플러그인 설치 (Claude Code 세션 내)

Claude Code 세션에서 슬래시 커맨드 입력:

```text
/plugin install telegram@claude-plugins-official
```

> 만약 "plugin not found" 에러:
>
> ```text
> /plugin marketplace add anthropics/claude-plugins-official
> /plugin marketplace update claude-plugins-official
> /plugin install telegram@claude-plugins-official
> ```

설치 후 활성화:

```text
/reload-plugins
```

## 5단계: 토큰 설정

3단계에서 받은 토큰으로:

```text
/telegram:configure <BOTFATHER_TOKEN>
```

저장 위치: `~/.claude/channels/telegram/.env`

대안: 환경 변수로 설정 (Claude Code 실행 전)

```bash
export TELEGRAM_BOT_TOKEN=123456789:ABC...
```

## 6단계: Channel 활성화 상태로 Claude Code 재시작

현재 Claude Code 세션 종료 후:

```bash
claude --channels plugin:telegram@claude-plugins-official
```

> 여러 채널 동시 사용 시 space-separated:
>
> ```bash
> claude --channels plugin:telegram@claude-plugins-official plugin:fakechat@claude-plugins-official
> ```

## 7단계: 페어링 (Telegram 계정과 연결)

1. Telegram에서 본인 봇에게 **아무 메시지나** 전송
2. 봇이 pairing code 응답 (예: `A1B2-C3D4`)
3. Claude Code 세션에서:

    ```text
    /telegram:access pair <code>
    ```

4. 허용 정책을 allowlist로 잠금:

    ```text
    /telegram:access policy allowlist
    ```

이후 본인 Telegram 계정만 메시지 push 가능.

## 8단계: CoinFello 초기 설정 (최초 1회)

Claude Code 세션 내 또는 직접 터미널에서:

```bash
# (선택) 서명 daemon 시작 — Touch ID/password 반복 입력 회피
npx @coinfello/agent-cli@latest signer-daemon start

# Smart account 생성 (기본: Secure Enclave, WSL/Linux는 software key 폴백)
npx @coinfello/agent-cli@latest create_account
# 개발용 plaintext 키 사용 시:
# npx @coinfello/agent-cli@latest create_account --use-unsafe-private-key

# 현재 주소 확인
npx @coinfello/agent-cli@latest get_account

# SIWE 로그인
npx @coinfello/agent-cli@latest sign_in
```

저장 위치: `~/.clawdbot/skills/coinfello/config.json`

> **보안 주의**: Linux 환경에서는 Secure Enclave 미지원 → plaintext 키 저장됨. 프로덕션/실사용은 macOS 또는 TPM 2.0 환경 권장.

## 9단계: End-to-End 테스트

### 테스트 A — read-only prompt (트랜잭션 없음)

Telegram에서 봇에게 전송:

```text
what is the chain ID for Base?
```

→ Claude가 Telegram 메시지 수신 → `send_prompt` 실행 → 응답을 Telegram으로 회신.

### 테스트 B — 트랜잭션 흐름 (delegation)

Telegram에서:

```text
send 0.001 USDC to 0xRecipientAddress... on Base
```

→ Claude 측에서 내부적으로:

1. `npx @coinfello/agent-cli@latest send_prompt "..."` 실행
2. 서버가 delegation 요청 → `~/.clawdbot/skills/coinfello/pending_delegation.json` 저장
3. Claude가 터미널에 delegation 요약 출력
4. (사용자 검토 후) `npx @coinfello/agent-cli@latest approve_delegation_request` 실행
5. `txn_id` 회신

## 핵심 파일 경로 요약

| 용도 | 경로 |
| --- | --- |
| Telegram 토큰 | `~/.claude/channels/telegram/.env` |
| CoinFello config | `~/.clawdbot/skills/coinfello/config.json` |
| 대기 중 delegation | `~/.clawdbot/skills/coinfello/pending_delegation.json` |
| 이 문서 | `/home/nubroo/coinfello_test/docs/setup-log.md` |
| Skill 본문 | `/home/nubroo/coinfello_test/coinfello-skill/SKILL.md` |

## 지원 체인 (CoinFello)

Ethereum · OP Mainnet · BNB Smart Chain · Polygon · Mantle · Base · Arbitrum One · Linea · Sepolia · Base Sepolia

> 이외 체인으로 smart account에 자금 이체 시 **영구 잠김**.

## 트러블슈팅

| 증상 | 대응 |
| --- | --- |
| `/plugin install` 실패 ("not found") | `/plugin marketplace add anthropics/claude-plugins-official` 후 재시도 |
| Telegram 봇 응답 없음 | Claude Code가 `--channels plugin:telegram@...` 플래그로 실행 중인지 확인 |
| `context window` 에러 | `npx @coinfello/agent-cli@latest new_chat` 로 대화 리셋 |
| Secure Enclave 미지원 경고 | WSL/Linux 정상 현상 — software key로 폴백됨 |
| 권한 요청 팝업이 데스크탑 없이 | `--dangerously-skip-permissions` (신뢰 환경에서만) 또는 permission relay 사용 |

## 실제 실행 기록 (2026-04-24)

### 주의: 슬래시 커맨드 vs bash

`/telegram:configure` 는 **Claude Code TUI 안에서** 입력하는 슬래시 커맨드. 일반 bash 쉘에서 실행하면 `bash: /telegram:configure: No such file or directory` 에러 발생.

### 토큰 노출 주의

봇 토큰을 Claude Code 채팅 본문에 붙여넣으면 대화 로그에 남음. 가능하면 **슬래시 커맨드 인자로만** 전달하고, 노출된 토큰은 BotFather `/revoke` 로 재발급.

### 실행한 슬래시 커맨드 (Claude Code 세션 내)

```text
/plugin marketplace add anthropics/claude-plugins-official
# → Successfully added marketplace: claude-plugins-official

/plugin install telegram@claude-plugins-official
# → ✓ Installed telegram. Run /reload-plugins to apply.

/reload-plugins
# → Reloaded: 3 plugins · 5 skills · 26 agents · 4 hooks · 2 plugin MCP servers

/telegram:configure <BOTFATHER_TOKEN>
# → 토큰이 ~/.claude/channels/telegram/.env 에 저장 (chmod 600)
```

### 저장 결과

| 항목 | 값 |
| --- | --- |
| `.env` 경로 | `~/.claude/channels/telegram/.env` |
| 권한 | `-rw-------` (600) |
| `access.json` | 아직 없음 (기본값: `dmPolicy=pairing`, allowlist 비어있음) |
| 플러그인 버전 | telegram@0.0.6 |

## 진행 체크리스트

- [x] Bun 설치
- [x] CoinFello skill 다운로드
- [x] BotFather에서 Telegram 봇 생성 + 토큰 확보
- [x] `/plugin marketplace add anthropics/claude-plugins-official`
- [x] `/plugin install telegram@claude-plugins-official`
- [x] `/reload-plugins`
- [x] `/telegram:configure <token>`
- [ ] **Claude Code 종료 후 `claude --channels plugin:telegram@claude-plugins-official` 재시작** ← 다음
- [ ] Telegram 봇에 DM → 페어링 코드 수신
- [ ] `/telegram:access pair <code>`
- [ ] `/telegram:access policy allowlist`
- [ ] CoinFello `create_account` / `sign_in`
- [ ] 테스트 A (read-only) + 테스트 B (트랜잭션)

## 참고 링크

- Channels 공식 문서: <https://code.claude.com/docs/en/channels>
- Telegram 플러그인 소스: <https://github.com/anthropics/claude-plugins-official/tree/main/external_plugins/telegram>
- CoinFello: <https://coinfello.com>
- CoinFello skill (clawhub): <https://clawhub.ai/brettcleary/coinfello>
