# Claude Code 연구 리포트 모음

[![Version](https://img.shields.io/badge/Claude_Code-v2.1.88-blueviolet?style=for-the-badge)](https://www.npmjs.com/package/@anthropic-ai/claude-code)
[![Reports](https://img.shields.io/badge/연구_리포트-12%2B-red?style=for-the-badge)](docs/)

**Claude Code v2.1.88**에 대한 독립 연구 리포트 모음입니다. 아키텍처, 에이전트 루프, 권한 모델, 시스템 프롬프트, MCP 통합, 컨텍스트 관리에 초점을 맞춥니다.

이 저장소는 의도적으로 **문서 전용**으로 유지됩니다. 여기에는 분석과 논평만 포함되며, 실행 가능한 소스 배포나 지원되는 CLI 빌드를 제공하지 않습니다.

**언어**: [English](README.md) | [中文](README_CN.md) | [日本語](README_JA.md) | **한국어** | [Español](README_ES.md)

---

## 저장소 목적

Claude Code는 현재 가장 정교한 프로덕션 AI 코딩 에이전트 중 하나입니다. 이 리포트들은 그 내부 구조를 연구하기 쉽게 만드는 것을 목표로 합니다.

- **에이전트 개발자**는 도구 오케스트레이션, 컨텍스트 관리, 복구 로직 패턴을 살필 수 있습니다
- **보안 연구자**는 텔레메트리, 원격 제어 메커니즘, 권한 경계를 관찰할 수 있습니다
- **AI 연구자**는 시스템 프롬프트 조립, 모델 라우팅, 에이전트 루프 설계를 분석할 수 있습니다
- **엔지니어**는 CLI 기반 에이전트 시스템 설계 참고 자료로 활용할 수 있습니다

이 리포트들은 **공개 배포된 npm 패키지**에 대한 기술 분석에 기반한 연구와 논평입니다.

---

## 연구 리포트

### 핵심 아키텍처 분석

| # | 리포트 | 배울 수 있는 것 | 링크 |
|---|--------|---------------|------|
| 06 | **에이전트 루프 심층 분석** | `query.ts` 분석 — 메시지 흐름, 스트리밍 도구 실행, 자동 압축, 복구 로직 | [읽기 →](docs/en/06-agent-loop-deep-dive.md) |
| 07 | **도구 시스템 아키텍처** | 40+ 도구의 등록, 검증, 권한 체크, 병렬 실행 방식 | [읽기 →](docs/en/07-tool-system-architecture.md) |
| 08 | **권한 및 보안 모델** | 화이트리스트/블랙리스트, 자동 승인, YOLO 모드, 샌드박스 통합 | [읽기 →](docs/en/08-permission-security-model.md) |
| 09 | **시스템 프롬프트 엔지니어링** | 15,000+ 토큰 시스템 프롬프트가 20+ 파트로 조립되는 방식 | [읽기 →](docs/en/09-system-prompt-engineering.md) |
| 10 | **MCP 통합 및 플러그인 시스템** | Model Context Protocol 클라이언트 구현 상세 | [읽기 →](docs/en/10-mcp-integration.md) |
| 11 | **컨텍스트 윈도우 관리** | 자동 압축, 대화 압축, 토큰 카운팅 | [읽기 →](docs/en/11-context-window-management.md) |
| 12 | **상태 관리 및 영속성** | 세션 상태, 대화 기록, 메모리 시스템 | [읽기 →](docs/en/12-state-management.md) |

### 발견 및 조사 리포트

| # | 리포트 | 배울 수 있는 것 | 링크 |
|---|--------|---------------|------|
| 01 | **텔레메트리 및 데이터 수집** | 이중 분석 파이프라인, 환경 핑거프린팅 | [읽기 →](docs/en/01-telemetry-and-privacy.md) |
| 02 | **숨겨진 기능과 코드네임** | 동물 코드네임, 기능 플래그, 내부/외부 빌드 차이 | [읽기 →](docs/en/02-hidden-features-and-codenames.md) |
| 03 | **언더커버 모드** | 공개 저장소에서 AI 흔적을 숨기는 메커니즘 | [읽기 →](docs/en/03-undercover-mode.md) |
| 04 | **원격 제어 및 킬스위치** | 서버 측 설정, GrowthBook 플래그, 긴급 제어면 | [읽기 →](docs/en/04-remote-control-and-killswitches.md) |
| 05 | **미래 로드맵** | KAIROS, Numbat, 미래 모델 단서, 미출시 도구 | [읽기 →](docs/en/05-future-roadmap.md) |

---

## 분석 대상 코드베이스 개요

아래 수치는 **분석 대상이 된 Claude Code 코드 스냅샷**을 가리키며, 현재 문서 전용 저장소 자체의 파일 수를 뜻하지 않습니다.

| 지표 | 값 |
|------|---|
| TypeScript 소스 파일 | **1,884** |
| 총 코드 라인 | **512,664** |
| 최대 파일 | `query.ts` — **785KB** |
| 내장 도구 | **40+** |
| 슬래시 커맨드 | **80+** |
| npm 의존성 | **192 패키지** |
| Feature-Gated 모듈 | **108개** |
| 런타임 모델 | Bun 빌드, Node.js 대상 패키지 |

---

## 아키텍처 개요

```
                          ┌─────────────────┐
                          │   User Input     │
                          │ (CLI / SDK / IDE)│
                          └────────┬────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        Entry Layer           │
                    │                              │
                    │  cli.tsx → main.tsx → REPL   │
                    │              └→ QueryEngine   │
                    └──────────────┬──────────────┘
                                   │
              ┌────────────────────▼────────────────────┐
              │           Query Engine Core              │
              │  System Prompt Assembly / Agent Loop    │
              └────────────────────┬────────────────────┘
                                   │
         ┌─────────────────────────▼─────────────────────────┐
         │              Tool Layer (40+ tools)               │
         └─────────────────────────────────────────────────────┘
```

---

## 관찰된 모듈 구조

리포트에서 분석 대상으로 삼은 소스 구조는 대체로 다음과 같습니다.

```
src/
├── main.tsx
├── QueryEngine.ts
├── query.ts
├── Tool.ts
├── Task.ts
├── tools.ts
├── commands.ts
├── context.ts
├── cost-tracker.ts
├── setup.ts
├── bridge/
├── cli/
├── commands/
├── components/
├── entrypoints/
├── hooks/
├── services/
├── state/
├── tasks/
├── tools/
├── types/
├── utils/
└── vendor/
```

현재 저장소 자체는 계속 문서 전용 구조를 유지합니다.

```
docs/
├── en/
└── zh/

README.md
README_CN.md
README_JA.md
README_KO.md
README_ES.md
QUICKSTART.md
```

---

## 사용 방법

- `docs/` 아래 리포트를 읽기
- 개별 리포트 링크를 공유하기
- 소프트웨어 배포물이 아니라 문서 아카이브로 취급하기

최소한의 읽기 안내는 [QUICKSTART.md](QUICKSTART.md)를 참고하세요.

---

## 법적 메모

이 프로젝트는 공개 배포된 소프트웨어 패키지(npm의 `@anthropic-ai/claude-code`)에 대한 **연구, 논평, 교육 분석**입니다.

`docs/` 디렉토리의 리포트는 유지관리자가 작성한 오리지널 분석과 논평입니다. 권리상 우려가 있으면 issue로 알려 주세요.
