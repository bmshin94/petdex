# Petdex 전수조사 분석 정리 (한국어)

> 작성일: 2026-09-21
> 대상 레포: <https://github.com/bmshin94/petdex>
> 원본(업스트림): <https://github.com/crafter-station/petdex>
> 공식 사이트: <https://petdex.dev> · npm: <https://www.npmjs.com/package/petdex> · Discord: <https://discord.gg/byhubdyBTe>

---

## 1. 이 프로젝트는 무엇인가

**"코딩 에이전트(Codex, Claude Code 등)가 작업할 때 화면 위에서 함께 움직이는 픽셀 펫의 공개 도감"**

원본 레포 기준 지표 (2026-09-21 조회):

| 항목 | 값 |
| --- | --- |
| Stars | 4,133 |
| Forks | 201 |
| 최초 생성 | 2026-05-02 |
| 라이선스 | MIT (펫 에셋은 제출자 소유) |
| 주 언어 | TypeScript |
| 제작 | Crafter Station (Lead: @RaillyHugo) |

### 구성 요소 4가지

| 덩어리 | 위치 | 역할 |
| --- | --- | --- |
| 웹 갤러리 | `src/` | 펫 구경 / 제출 / 심사 / 랭킹 / 컬렉션 |
| CLI | `packages/petdex-cli/` | `npx petdex install <slug>` 로 펫 설치 |
| 데스크톱 앱 | `packages/petdex-desktop-native/` | 화면에 펫 띄우고 에이전트 활동에 반응 |
| 디스코드 봇 | `packages/discord-bot/` | 커뮤니티 서버 운영 |

---

## 2. 코드베이스 실측

- 총 소스 파일: **433개** (`src/`)
- React 컴포넌트: **129개**
- 전체 TS/TSX 라인 수: **약 67,000줄**
- API 라우트: **55개**
- DB 테이블: **24개** (`src/lib/db/schema.ts`, 918줄)
- 로케일 페이지: **22종**
- GitHub Actions 워크플로: **13종**
- "Built with Petdex" 등록 프로젝트: **30개**

### 주요 폴더

| 경로 | 설명 |
| --- | --- |
| `src/app/[locale]/` | 공개 사이트 (pets, collections, leaderboard, built-with, create, download, submit, stickers, requests, u/<handle> 등) |
| `src/app/api/manifest/` | **공개 API.** 승인된 전체 펫 목록 + 스프라이트 URL |
| `src/app/api/cli/` | CLI 전용 (OAuth 설정, presign, 제출, 중복체크, 등록) |
| `src/lib/db/schema.ts` | Drizzle 스키마 |
| `packages/petdex-cli/src/hooks/bubble-runner.ts` | 에이전트 훅 핫패스 |
| `packages/petdex-cli/src/hooks/mcp-server.ts` | SDK 없는 stdio MCP 서버 |
| `packages/petdex-cli/src/cli-auth/` | Clerk OAuth + PKCE |
| `.agents/skills/` | 외부 스킬 번들 (`skills-lock.json` 으로 해시 고정) |
| `docs/agent-adapters-research.md` | 에이전트 지원 여부 판단 리서치 |
| `docs/chatgpt-pet-integration.md` | ChatGPT.app 파서 리버스 엔지니어링 기록 |
| `workers/petdex-assets.ts` | Cloudflare Worker (에셋 엣지 캐싱) |

---

## 3. 기술 스택

| 영역 | 기술 |
| --- | --- |
| 프론트/서버 | Next.js 16.3.2, React 19.2.8, Tailwind v4, next-intl |
| DB | Postgres(Neon) + Drizzle ORM, PGlite(로컬 목) |
| 인증 | Clerk (웹 + CLI OAuth) |
| 스토리지 | Cloudflare R2 (+ 알리바바 OSS, 중국 대응) |
| 캐시/레이트리밋 | Upstash Redis + Neon 기반 레이트리밋 |
| AI | Vercel AI Gateway (자동 태깅, 리뷰, 사운드) |
| 메일 | Resend |
| 런타임/툴 | Bun, Biome |
| 데스크톱 | Zig + Native SDK (vercel-labs/native), WebView/Node 사이드카 없음 |

---

## 4. 동작 원리 (쉬운 버전)

```
① 사용자가 AI 에이전트에게 작업 요청
② 에이전트가 툴 호출 (파일 읽기, 명령 실행 등)
③ 훅 발동 → 127.0.0.1:7777 로 POST
④ 데스크톱 펫이 해당 상태 애니메이션으로 전환
⑤ 완료 시 waving, 실패 시 failed 모션
```

### 펫 패키지 포맷

```
my-pet/
├── pet.json          # 이름, 슬러그, 태그, 바이브, 종류, 프레임 크기, 상태
└── spritesheet.webp  # 8x9 (1536x1872) 또는 v2 8x11 (1536x2288)
                      # 프레임 1개 = 192x208 px
```

지원 상태 9종: `idle`, `running-right`, `running-left`, `waving`, `jumping`, `failed`, `waiting`, `running`, `review`

---

## 5. 설치 및 사용법

### 사용자

```sh
npx petdex install boba     # 펫 설치 (~/.petdex/pets/, ~/.codex/pets/)
npx petdex list             # 목록 보기
```
이후 <https://petdex.dev/download> 에서 데스크톱 앱 설치 → `Cmd + ,` 로 설정 열고 펫/에이전트 연결.

### 창작자

```sh
npx petdex login                   # Clerk OAuth + PKCE (브라우저)
npx petdex submit ./my-pet/        # 단일 제출
npx petdex submit ./my-pets/       # 벌크 제출
npx petdex edit <slug> --desc "…"  # 내 펫 수정
```
제한: 24시간당 10개 제출 (관리자 제외)

### 개발자 (이 레포 작업)

```sh
bun install          # npm 아님. 반드시 bun
bun run dev:docker   # Postgres + Redis 자동 기동 (약 30초)
bun run check        # Biome 린트
bun run format       # 자동 수정
bun run build        # Next 빌드
bun test             # 테스트
bun run i18n:check   # 번역 검증
```

주의사항:
- `bun run dev:mock` 은 폐기됨 (실행 시 종료)
- `packages/` 는 루트 워크스페이스가 아님 → 각 폴더에서 따로 `bun install`
- 루트에 `typecheck` 스크립트 없음
- DB 의존 테스트는 env 필요: `bun test --env-file=.env.mock src/lib/security.test.ts`

---

## 6. 플러그인인가, 스킬인가, MCP인가

**결론: 셋 다 아니고, 셋 다 들어있다.** 본체는 독립 풀스택 제품이고 에이전트 연동 경로가 3종이다.

| 구분 | 위치 | 비고 |
| --- | --- | --- |
| 훅 (주력) | `bubble-runner.ts` | `PreToolUse` / `PostToolUse` / `UserPromptSubmit` / `Stop` 에 `petdex bubble <event>` 등록 |
| MCP 서버 | `mcp-server.ts` | Antigravity용 stdio JSON-RPC 2.0. 외부 SDK 없이 직접 구현 |
| 스킬 | `hatch-pet` (Codex용), `.agents/skills/` | 펫 제작 보조 + 레포가 쓰는 외부 스킬 |
| 플러그인 | `integrations/dsh`, `integrations/herdr` | 데스크톱 앱 플러그인 |

---

## 7. API 토큰 필요 여부

| 하려는 것 | 토큰 |
| --- | --- |
| 펫 설치 / 목록 / `/api/manifest` 호출 | 불필요 (완전 공개) |
| 데스크톱 앱 사용 | 불필요 |
| 펫 제출 / 수정 | 필요하지만 수동 발급 아님 → `petdex login` (Clerk OAuth 2.0 + PKCE S256) |

- 토큰은 OS 키체인에 저장 (macOS Keychain / Windows Credential Manager / Linux Secret Service), 없으면 `chmod 600` 파일 폴백
- 업로드는 presigned R2 PUT (60초 TTL) → 파일 본문이 Petdex 서버를 거치지 않음

### 직접 호스팅 시 필요한 키

Clerk, Postgres/Neon, Cloudflare R2, Upstash Redis, Resend, `AI_GATEWAY_API_KEY`, OpenAI, ElevenLabs
(단 `bun run dev:docker` 는 `.env.dev` 의 공용 Clerk 개발 키 + 로컬 Docker DB로 키 없이 개발 가능)

### 보안 불변식 (AGENTS.md)

- 제출자 신원/크레딧은 검증된 Clerk 세션 또는 CLI 베어러 토큰에서만. 요청 body에서 절대 받지 않음
- 상태 변경 브라우저 엔드포인트는 `requireSameOrigin` 사용
- 외부 URL은 `src/lib/url-allowlist.ts` 허용목록 + `next.config.ts` CSP 동기화 필요

---

## 8. 왜 GitHub에서 유명한가

1. **타이밍** — AI 코딩 에이전트 폭발기에 "에이전트 액세서리"라는 빈 자리를 선점
2. **한 장으로 설명되는 제품** — GIF 하나로 바이럴, 소셜 친화적
3. **진입장벽 제로** — `npx` 한 줄, 가입/설정 불필요
4. **UGC 플라이휠** — 제작 → 등재 → SNS 자랑 → 신규 유입 → 반복. 프로필/리더보드/좋아요/컬렉션으로 창작 동기 설계
5. **생태계 개방** — `/api/manifest` 공개 + 포맷 문서화 → 30개 외부 프로젝트가 위에 올라탐 (Garmin 워치 펫, Swift 라이브러리, 터미널 펫, 플랫포머 게임 등)
6. **엔지니어링 퀄리티** — Zig 네이티브 앱, 명시적 CSP, CI 13종, 스프라이트 아틀라스 자동 감사, **자체 비용 추적 시스템**
7. **MIT + 친절한 온보딩** — README, CONTRIBUTING, 이슈 템플릿, Discord

> 교훈: "쓸모"보다 "재미", "기능"보다 "보여주기 쉬움"이 스타를 만든다. 다만 밑에 진짜 엔지니어링이 없으면 금방 꺼진다.

---

## 9. 로컬 에이전트 구축에 주는 도움

"펫 기능"이 아니라 **배선 방식**이 자산이다.

| # | 패턴 | 참고 파일 |
| --- | --- | --- |
| 1 | 로컬 IPC 서버 (`127.0.0.1:7777` + 토큰 헤더 + 파일 킬스위치) | 데스크톱 훅 서버 |
| 2 | 훅 핫패스 방어 (stdin 64KB 캡, fetch 300ms 타임아웃, 전 에러 무음, 초과분 드레인) | `bubble-runner.ts` |
| 3 | SDK 없는 MCP 서버 (framed/jsonl 트랜스포트, `initialize` 전 stdout 금지) | `mcp-server.ts` |
| 4 | 다중 에이전트 어댑터 판단 프레임워크 | `docs/agent-adapters-research.md` |
| 5 | SSH 원격 에이전트 브리징 | `~/.petdex/remote-agents.json` |
| 6 | CLI OAuth (로컬호스트 랜덤 포트 + PKCE + 키체인) | `src/cli-auth/` |

### 어댑터 리서치 판정 사례

| 에이전트 | 판정 | 근거 |
| --- | --- | --- |
| Kimi Code | 채택 | `PostToolUseFailure` 별도 이벤트 |
| CodeBuddy | 채택 | `PostToolUse` 실패 필드 |
| OMP | 채택 | `tool_result` 의 `isError` |
| OpenClaw | 거절 | **툴 레벨 이벤트 자체가 없음** |

→ "기능 유무"가 아니라 "내 모델에 필요한 이벤트가 있는가"로 판단하는 사고법.

### 읽기 우선순위

1. `packages/petdex-cli/src/hooks/bubble-runner.ts`
2. `packages/petdex-cli/src/hooks/mcp-server.ts`
3. `docs/agent-adapters-research.md`
4. `packages/petdex-cli/src/cli-auth/`
5. `packages/petdex-desktop-native/README.md`

### 한계

- 데스크톱 앱은 Zig → 개념만 차용 가능
- Petdex는 에이전트를 **만드는** 툴이 아니라 에이전트에 **붙는** 툴. LLM 호출/툴 루프 같은 코어는 여기 없음

---

## 10. React / PHP 로 만들 수 있는가

### React — 이미 React다

Next.js 16 + React 19 + Tailwind v4. 웹 부분은 즉시 수정 가능.
바로 가능한 것: 갤러리 사이트, Canvas 스프라이트 플레이어, 스프라이트 에디터, Electron/Tauri 래핑 데스크톱 펫, VSCode 익스텐션, 브라우저 확장

### PHP — 부분 가능

| 부분 | 가능 여부 | 비고 |
| --- | --- | --- |
| 웹 갤러리 | 가능 | Laravel + Blade/Inertia |
| 매니페스트 API | 가능 | API 리소스로 충분 |
| 관리자 심사 도구 | 오히려 쉬움 | Filament / Nova |
| 스프라이트 처리 | 가능 | GD / Imagick |
| 스티커 변환 | 가능 | Imagick WebP 애니메이션 |
| CLI 도구 | 비추 | `npx` 같은 배포 채널 부재 |
| 데스크톱 떠있는 펫 | 불가 | PHP는 GUI 불가 → Electron/Tauri/Zig/Swift |
| 에이전트 훅 | 비추 | 호출당 프로세스 부팅이 너무 느림 (세션당 20~50회) |

### 추천 조합

```
웹 갤러리 + API : Laravel (PHP)
프론트          : React + Inertia.js
CLI             : Node.js (작게)
데스크톱 펫      : Tauri 또는 Electron
```
또는 데스크톱을 포기하고 브라우저 확장 / VSCode 확장 / 웹 대시보드만 → PHP + React로 100% 커버

### 난이도별 로드맵

| 단계 | 결과물 | 기간 | 스택 |
| --- | --- | --- | --- |
| 1 | Canvas 스프라이트 플레이어 | 1일 | React |
| 2 | 펫 갤러리 웹 (Petdex API 소비) | 3일 | React 또는 Laravel |
| 3 | 업로드/관리 시스템 | 1주 | Laravel + R2/S3 |
| 4 | 스티커 변환기 | 1주 | PHP Imagick |
| 5 | 데스크톱 펫 | 2~3주 | Electron/Tauri |

---

## 11. 수익화 아이디어

전제: **소스는 MIT라 상업 이용 가능. 단 펫 에셋은 제출자 소유이므로 남의 펫으로 장사 불가. IP 리스크는 직접 관리.**

### TIER 1 — 단기

#### 1) 카카오 이모티콘 / 라인 스티커 (최우선 추천)

- 레포에 스티커 파이프라인이 **이미 존재** (`/api/pets/[slug]/sticker`, `/wastickers`, `src/components/stickers/`)
- 그런데 **왓츠앱용만 있고 카톡/라인은 비어있음**
- 픽셀 애니메이션은 이미 프레임 단위 분할되어 있어 변환이 쉬움

| 플랫폼 | 가격대 | 정산 | 비고 |
| --- | --- | --- | --- |
| 카카오 이모티콘 | 2,500원 | 약 35~40% | 승인 경쟁 치열, 대신 대박 가능 |
| 라인 스티커 | 약 1~2천원 | 50% | 승인 쉬움, 글로벌 |
| 왓츠앱 | 무료 | - | 유입/마케팅용 |

난이도 ★★ / 기간 2~4주 / 초기비용 거의 0

#### 2) 한국어 Petdex (ko 로케일 선점)

- 현재 i18n 로케일은 `en`, `es`, `zh` 뿐 → **한국어 없음**
- 루트 A: `ko` 로케일 PR 기여 → 공식 컨트리뷰터 + 한국 시장 포지션 확보
- 루트 B: K-캐릭터 특화 독립 서비스 (프리미엄 펫팩 구독 + 스티커 + 굿즈)

난이도 ★★ / 기간 1~2주 (번역만이면 3일)

#### 3) POD 굿즈

| 상품 | 판매가 | 원가 | 마진 |
| --- | --- | --- | --- |
| 아크릴 키링 | 8,000 | 2,500 | 5,500 |
| 스티커팩 | 5,000 | 800 | 4,200 |
| 마우스패드 | 18,000 | 7,000 | 11,000 |
| 반팔티 | 25,000 | 12,000 | 13,000 |
| 노트북 스티커 세트 | 12,000 | 2,000 | 10,000 |

채널: 마플샵 / 오하프린트 / 레드버블 / 스마트스토어. 재고 0 POD로 시작 후 히트 상품만 대량 발주.

### TIER 2 — 중기 구독형

#### 4) AI 스프라이트 생성 SaaS (수익성 최고)

문제: 8x9 그리드 72프레임 스프라이트 제작이 생태계 최대 병목.
해결: 사진/텍스트 → 픽셀화 → 포즈 추정 + 프레임 보간 → 1536x1872 시트 + `pet.json` 자동 생성.

| 플랜 | 가격 | 내용 |
| --- | --- | --- |
| Free | 0원 | 월 1개, 워터마크 |
| Creator | 9,900원/월 | 월 20개, 상업이용 |
| Studio | 39,000원/월 | 무제한, API, 팀 공유 |
| 단건 | 3,000원 | 펫 1개 |

시장이 Petdex 유저를 넘어섬: 인디 게임 개발자, VTuber, 디스코드 봇 제작자.
기술: Stable Diffusion + ControlNet, 또는 Replicate/fal.ai. 난이도 ★★★★ / 2~3개월

#### 5) 기업 마스코트 B2B (객단가 최고)

컨셉: "귀사의 마스코트가 개발자 화면에서 살아 움직입니다"

| 플랜 | 가격 | 내용 |
| --- | --- | --- |
| 스타터 | 200만원 | 마스코트 펫 1종 + 갤러리 등재 |
| 브랜드 | 500만원 | 펫 3종 + 전용 설치 페이지 + 1년 운영 |
| 캠페인 | 1,000만원~ | 위 + 이벤트 + 굿즈 + 성과 리포트 |

타겟: 개발자 대상 SaaS, 국내 테크 기업, 컨퍼런스/해커톤 주최사, 부트캠프.
근거: 개발자 마케팅은 광고가 안 먹히고, 하루 8시간 화면 노출은 강력한 브랜드 자산. 국내 경쟁자 전무.

#### 6) 데스크톱 펫 Pro

| 무료 | Pro (월 4,900원) |
| --- | --- |
| 펫 1마리 | 여러 마리 동시 (파티 모드) |
| 기본 9개 상태 | 커스텀 반응 스크립팅 |
| - | 포모도로 타이머 |
| - | 일일/주간 코딩 리포트 |
| - | 펫 레벨업/성장 시스템 |
| - | 팀 모드 (동료 펫 표시) |
| - | 프리미엄 펫 팩 |

육성 요소 = 리텐션 장치. 난이도 ★★★★ / 2~3개월

### TIER 3 — 장기 플랫폼

#### 7) 펫 크리에이터 마켓플레이스
유료 펫 판매(2,000~10,000원), 플랫폼 수수료 20~30%. 커미션 중개 수수료 15%. 콘텐츠 제작비 0원.

#### 8) IP 콜라보 라이선싱
웹툰/인디게임/유튜버 캐릭터의 공식 픽셀 펫화. 수익 배분 또는 제작비. **합법 IP만** (원본이 takedown 템플릿을 둔 이유).

#### 9) 에이전트 관측 SaaS (가장 큰 시장)
Petdex 훅 인프라의 본질은 **AI 에이전트 활동 추적 시스템**. 툴 호출 횟수/실패 지점/세션 시간/비용을 팀 대시보드로 → "AI 코딩 에이전트판 Datadog". 시트당 월 2~3만원 B2B. 난이도 ★★★★★

#### 10) 교육 콘텐츠
"Petdex로 배우는 Next.js 16 풀스택", "AI 에이전트 훅 & MCP 서버 만들기" 강의. 5만원 x 500명 = 2,500만원 규모.

### 추천 로드맵

```
1~2개월  : ko 로케일 PR + 픽셀 펫 24종 디자인 + 라인 스티커 출시
3~4개월  : 카카오 이모티콘 제안 + POD 굿즈 + 한국형 갤러리 웹 런칭
5~8개월  : AI 스프라이트 SaaS 베타 + B2B 마스코트 영업
9개월~   : 크리에이터 마켓 + 에이전트 관측 SaaS 피벗 검토
```

### 우선순위 평가

| 아이디어 | 추천도 | 이유 |
| --- | --- | --- |
| 스티커/이모티콘 | ★★★★★ | 즉시 착수 가능, 리스크 최소, 파이프라인 기존재 |
| AI 스프라이트 SaaS | ★★★★★ | 실제 병목을 해결 |
| B2B 마스코트 | ★★★★ | 객단가 최고, 국내 무경쟁 |
| ko 로케일 | ★★★★ | 수익은 작지만 레버리지가 큼 |
| 에이전트 관측 | ★★★ | 시장 최대, 난이도 최대 |
| 굿즈 | ★★★ | 쉽지만 마진 한계 |

---

## 12. 참고 링크

- 이 레포: <https://github.com/bmshin94/petdex>
- 원본 레포: <https://github.com/crafter-station/petdex>
- 공식 사이트: <https://petdex.dev>
- 생태계 카탈로그: <https://petdex.dev/built-with>
- 펫 만들기: <https://petdex.dev/create>
- 데스크톱 앱: <https://petdex.dev/download>
- npm 패키지: <https://www.npmjs.com/package/petdex>
- Discord: <https://discord.gg/byhubdyBTe>
- CLI 레퍼런스: `packages/petdex-cli/README.md`
- 기여 가이드: `CONTRIBUTING.md`
