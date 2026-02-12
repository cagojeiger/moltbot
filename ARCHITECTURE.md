# OpenClaw (moltbot) Architecture Analysis

## 전체 시스템 개요

```
+=====================================================================+
|                     OpenClaw System Architecture                     |
+=====================================================================+

                        +-------------------+
                        |     사용자 (User)   |
                        +-------------------+
                               |
            +------------------+------------------+
            |                  |                  |
            v                  v                  v
   +----------------+  +-------------+  +----------------+
   | Native Apps    |  |  Web UI     |  |  CLI           |
   | (macOS/iOS/    |  |  (Control   |  |  (Terminal)    |
   |  Android)      |  |   UI)       |  |                |
   +-------+--------+  +------+------+  +-------+--------+
           |                   |                 |
           +-------------------+-----------------+
                               |
                          WebSocket
                               |
                               v
+======================================================================+
|                        GATEWAY SERVER                                 |
|                    (Central Control Plane)                            |
|======================================================================|
|                                                                      |
|  +------------------+  +------------------+  +------------------+    |
|  | WebSocket Server |  | HTTP Server      |  | Method Router    |    |
|  | (ws)             |  | (Express)        |  | (RPC Dispatch)   |    |
|  +--------+---------+  +--------+---------+  +--------+---------+    |
|           |                     |                     |              |
|           +---------------------+---------------------+              |
|                                 |                                    |
|  +-------------------------------------------------------------+    |
|  |                    Gateway Request Context                    |    |
|  |  +----------------+  +-------------+  +------------------+   |    |
|  |  | Channel Mgr    |  | Session Mgr |  | Plugin Registry  |   |    |
|  |  | (Multi-Account)|  | (Per-Peer)  |  | (Dynamic Load)   |   |    |
|  |  +----------------+  +-------------+  +------------------+   |    |
|  |  +----------------+  +-------------+  +------------------+   |    |
|  |  | Cron Service   |  | Node Mgr    |  | Broadcast Engine |   |    |
|  |  | (Scheduler)    |  | (Remote)    |  | (Event Fan-out)  |   |    |
|  |  +----------------+  +-------------+  +------------------+   |    |
|  +-------------------------------------------------------------+    |
|                                                                      |
+==========================+========+==================================+
                           |        |
              +------------+        +------------+
              |                                  |
              v                                  v
+===========================+    +============================+
|     AGENT RUNTIME         |    |    CHANNEL PLUGINS         |
|===========================|    |============================|
|                           |    |                            |
|  +---------------------+ |    |  +--------+  +--------+   |
|  | Pi Agent SDK        | |    |  |WhatsApp|  |Telegram|   |
|  | (Session + Stream)  | |    |  +--------+  +--------+   |
|  +----------+----------+ |    |  +--------+  +--------+   |
|             |             |    |  | Slack  |  |Discord |   |
|  +----------v----------+ |    |  +--------+  +--------+   |
|  | Tool System         | |    |  +--------+  +--------+   |
|  | (40+ Built-in)      | |    |  | Signal |  | Teams  |   |
|  +---------------------+ |    |  +--------+  +--------+   |
|  +---------------------+ |    |  +--------+  +--------+   |
|  | Skill System        | |    |  | LINE   |  | Matrix |   |
|  | (50+ Bundled)       | |    |  +--------+  +--------+   |
|  +---------------------+ |    |  +--------+  +--------+   |
|  +---------------------+ |    |  |iMessage|  | Zalo   |   |
|  | Session Persistence | |    |  +--------+  +--------+   |
|  | (JSONL Files)       | |    |  ... + 36 extensions ...   |
|  +---------------------+ |    |                            |
+===========================+    +============================+
              |
              v
     +------------------+
     |  ~/.openclaw/     |
     |  (Persistent      |
     |   Storage)        |
     +------------------+
```

---

## 핵심 설계 철학 (Core Design Principles)

### 1. Hub-and-Spoke 아키텍처

```
                    WhatsApp  Telegram  Slack  Discord
                       \        |       |      /
                        \       |       |     /
                         v      v       v    v
                    +========================+
                    |                        |
                    |    GATEWAY (Hub)       |
                    |                        |
                    +========================+
                       /     |      |     \
                      /      |      |      \
                     v       v      v       v
                  Agent   Agent   Cron    Hooks
                  Main    Sub    Service  System
```

- Gateway가 **단일 허브**로 모든 통신 중계
- 채널(Spoke)은 경량 어댑터로 외부 프로토콜만 정규화
- 에이전트, 크론, 훅 등 내부 서비스도 Spoke로 연결

### 2. 3-프레임 WebSocket 프로토콜

```
  Client                           Gateway
    |                                |
    |-- RequestFrame (req) -------->|     { type:"req", id, method, params }
    |                                |
    |<-- ResponseFrame (res) -------|     { type:"res", id, ok, payload }
    |                                |
    |<-- EventFrame (event) --------|     { type:"event", event, payload, seq }
    |<-- EventFrame (event) --------|     (비동기 브로드캐스트)
    |<-- EventFrame (event) --------|
    |                                |
```

- **Request/Response**: 동기적 RPC 호출
- **Event**: 비동기 브로드캐스트 (채팅 델타, 상태 변경 등)
- **Sequence Number**: 이벤트 갭 감지

### 3. 계층적 세션 키

```
  세션 키 구조:

  agent:<agentId>:main
  ├── 메인 세션 (전체 대화)
  │
  agent:<agentId>:<channel>:direct:<peerId>
  ├── 채널별 1:1 대화 세션
  │
  agent:<agentId>:<channel>:<accountId>:direct:<peerId>
  ├── 멀티 계정 지원 세션
  │
  agent:<agentId>:<channel>:group:<groupId>
  ├── 그룹 채팅 세션
  │
  agent:<agentId>:<channel>:direct:<peerId>:thread:<threadId>
  └── 스레드 세션

  DM Scope 설정:
  ┌──────────────────────┬─────────────────────────────────┐
  │ "main"               │ 모든 DM → 하나의 세션           │
  │ "per-peer"           │ 상대방별 별도 세션              │
  │ "per-channel-peer"   │ 채널+상대방별 별도 세션         │
  │ "per-account-peer"   │ 계정+채널+상대방별 완전 격리    │
  └──────────────────────┴─────────────────────────────────┘
```

---

## 메시지 흐름 (End-to-End Message Flow)

```
  [1] INBOUND (외부 → 시스템)
  ═══════════════════════════════════════════════════════

  WhatsApp 사용자가 "오늘 날씨 알려줘" 전송
          |
          v
  +------------------+
  | Channel Plugin   |    WhatsApp SDK (Baileys) 가 메시지 수신
  | (whatsapp)       |    → 표준 형식으로 정규화
  +--------+---------+       { from, to, body, chatType, accountId }
           |
           v
  +------------------+
  | Route Resolver   |    resolveAgentRoute()
  | (routing/)       |    → 바인딩 매칭 (peer → guild → channel → default)
  +--------+---------+    → agentId, sessionKey 결정
           |
           v
  +------------------+
  | Dispatch         |    dispatchInboundMessage()
  | (auto-reply/)    |    → 세션 로드/생성
  +--------+---------+    → 사용자 메시지 추가
           |
           v
  [2] AGENT EXECUTION (처리)
  ═══════════════════════════════════════════════════════

  +------------------+
  | Lane Manager     |    세션 레인 큐잉 (동일 세션 직렬화)
  | (run queue)      |    글로벌 레인 (전체 속도 제한)
  +--------+---------+
           |
           v
  +------------------+
  | Model Resolver   |    resolveModel() → 모델 유효성 검증
  | + Auth Profile   |    resolveAuthProfileOrder() → API 키 로드
  +--------+---------+    evaluateContextWindowGuard() → 토큰 제한 확인
           |
           v
  +------------------+
  | Tool & Skill     |    createOpenClawCodingTools()
  | Builder          |    → 정책 필터링 (global → agent → sandbox)
  +--------+---------+    → 40+ 도구 등록
           |
           v
  +------------------+
  | Pi Agent Session |    createAgentSession() → SDK 초기화
  | (SDK)            |    session.sendMessage("오늘 날씨 알려줘")
  +--------+---------+
           |
           v
  +------------------+
  | Stream Handler   |    subscribeEmbeddedPiSession()
  | (Event Machine)  |    → message_start/update/end
  +--------+---------+    → tool_execution_start/update/end
           |               → agent_start/end
           |
           |  에러 발생 시:
           |  ├─ Auth 에러 → 다음 프로필로 폴백
           |  ├─ 컨텍스트 오버플로우 → 자동 압축 (3회 재시도)
           |  ├─ 도구 결과 초과 → 결과 잘라내기
           |  └─ 타임아웃 → thinking 레벨 폴백
           |
           v
  [3] OUTBOUND (시스템 → 외부)
  ═══════════════════════════════════════════════════════

  +------------------+
  | Response Builder |    buildEmbeddedRunPayloads()
  | (Payloads)       |    → 텍스트, 미디어, 도구 결과 조합
  +--------+---------+
           |
           +--------+--------+
           |                 |
           v                 v
  +----------------+  +------------------+
  | WebSocket      |  | Channel Plugin   |
  | Broadcast      |  | Outbound         |
  | (UI 업데이트)   |  | (메시지 발송)     |
  +----------------+  +--------+---------+
                               |
                               v
                      +------------------+
                      | Text Chunking    |   채널별 글자 제한 적용
                      | Discord: 2000    |   (Telegram: 4096)
                      | Slack: 40000     |   (WhatsApp: 65536)
                      +--------+---------+
                               |
                               v
                      WhatsApp 사용자에게 답변 전달
```

---

## 크론/스케줄링 시스템

```
  +----------------------------------------------------------+
  |                    CRON SERVICE                           |
  +----------------------------------------------------------+
  |                                                          |
  |   jobs.json (디스크 영속)                                 |
  |   ┌────────────────────────────────────────────────┐     |
  |   │ { version: 1, jobs: [                          │     |
  |   │   { id, name, schedule, payload, state, ... }  │     |
  |   │ ]}                                             │     |
  |   └────────────────────────────────────────────────┘     |
  |          |                                               |
  |          v                                               |
  |   ┌──────────────┐                                      |
  |   │ Timer Loop   │◄── setTimeout (최대 60초 간격)        |
  |   │ (event loop) │                                      |
  |   └──────+───────┘                                      |
  |          |                                               |
  |          v                                               |
  |   ┌──────────────┐    아니오                             |
  |   │ 실행할 작업   │──────────► 타이머 재설정              |
  |   │ 있는가?      │                                      |
  |   └──────+───────┘                                      |
  |          | 예                                            |
  |          v                                               |
  |   ┌──────────────────────────────────────┐              |
  |   │         실행 모드 분기               │              |
  |   ├──────────────────┬───────────────────┤              |
  |   │   Main Session   │  Isolated Session │              |
  |   │                  │                   │              |
  |   │  systemEvent     │  agentTurn        │              |
  |   │  텍스트 주입     │  독립 에이전트 실행│              |
  |   │       ↓          │       ↓           │              |
  |   │  enqueueSystem   │  runIsolated      │              |
  |   │  Event()         │  AgentJob()       │              |
  |   │       ↓          │       ↓           │              |
  |   │  heartbeat       │  결과 요약 추출   │              |
  |   │  wake            │       ↓           │              |
  |   │                  │  Delivery Plan    │              |
  |   │                  │  (announce/none)  │              |
  |   └──────────────────┴───────────────────┘              |
  |          |                                               |
  |          v                                               |
  |   ┌──────────────────────────────────────┐              |
  |   │         결과 처리                     │              |
  |   ├──────────────────────────────────────┤              |
  |   │  성공 (ok):                          │              |
  |   │    consecutiveErrors = 0             │              |
  |   │    nextRunAtMs 재계산                │              |
  |   │    at 스케줄 → deleteAfterRun 시 삭제│              |
  |   │                                      │              |
  |   │  에러 (error):                       │              |
  |   │    consecutiveErrors++               │              |
  |   │    지수 백오프 적용:                  │              |
  |   │      1회→30초  2회→1분  3회→5분      │              |
  |   │      4회→15분  5회+→60분             │              |
  |   │    at 스케줄 → enabled=false         │              |
  |   │                                      │              |
  |   │  스킵 (skipped):                     │              |
  |   │    at 스케줄 → enabled=false         │              |
  |   └──────────────────────────────────────┘              |
  |          |                                               |
  |          v                                               |
  |   디스크에 영속화 → 타이머 재설정                        |
  +----------------------------------------------------------+

  스케줄 타입:
  ┌─────────┬───────────────────────────┬──────────────────┐
  │  "at"   │ 1회성 절대 시간            │ ISO-8601 타임스탬프│
  │ "every" │ 고정 간격 반복             │ everyMs (밀리초)  │
  │ "cron"  │ 크론 표현식 + 타임존       │ "0 7 * * *"      │
  └─────────┴───────────────────────────┴──────────────────┘
```

---

## 플러그인/확장 시스템

```
  플러그인 발견 순서 (우선순위):
  ┌─────────────────────────────────────────────────┐
  │ 1. config.plugins.loadPaths  (설정 지정 경로)    │
  │ 2. .openclaw/extensions/     (워크스페이스)      │
  │ 3. ~/.config/openclaw/ext/   (글로벌)           │
  │ 4. extensions/               (번들)             │
  └─────────────────────────────────────────────────┘
          |
          v
  ┌─────────────────────────────────────────────────┐
  │              Plugin Module Format                │
  │                                                  │
  │  export default {                                │
  │    id: "my-plugin",                              │
  │    register(api: OpenClawPluginApi) {             │
  │      api.registerChannel(...)    // 채널 추가     │
  │      api.registerTool(...)       // 도구 추가     │
  │      api.registerHook(...)       // 훅 추가      │
  │      api.registerGatewayMethod() // RPC 추가     │
  │      api.registerService(...)    // 서비스 추가   │
  │      api.registerProvider(...)   // 프로바이더     │
  │      api.registerCli(...)        // CLI 추가     │
  │    }                                             │
  │  }                                               │
  └─────────────────────────────────────────────────┘
          |
          v
  ┌─────────────────────────────────────────────────┐
  │              Plugin Registry                     │
  │                                                  │
  │  plugins[]       → 메타데이터 + 상태             │
  │  channels[]      → 채널 플러그인                 │
  │  tools[]         → 에이전트 도구                 │
  │  hooks[]         → 라이프사이클 훅               │
  │  gatewayHandlers → RPC 핸들러                   │
  │  httpRoutes[]    → HTTP 엔드포인트              │
  │  services[]      → 백그라운드 서비스             │
  │  providers[]     → 인증/LLM 프로바이더          │
  │  diagnostics[]   → 로딩 에러/경고               │
  └─────────────────────────────────────────────────┘

  훅 실행 모델:
  ┌────────────────────┬────────────────────┬─────────────────┐
  │ Void Hooks         │ Modifying Hooks    │ Sync Hooks      │
  │ (fire-and-forget)  │ (결과 수정)         │ (동기 실행)      │
  │                    │                    │                 │
  │ 병렬 실행          │ 순차 실행 + 머지    │ 동기 실행       │
  │                    │                    │                 │
  │ message_received   │ message_sending    │ tool_result_    │
  │ message_sent       │ before_tool_call   │ persist         │
  │ agent_end          │                    │                 │
  └────────────────────┴────────────────────┴─────────────────┘
```

---

## 에이전트 런타임

```
  ┌──────────────────────────────────────────────────────┐
  │               Agent Execution Pipeline               │
  ├──────────────────────────────────────────────────────┤
  │                                                      │
  │  [1] 레인 관리 (Lane Management)                     │
  │  ┌────────────────────────────────────────────┐      │
  │  │ Session Lane: 동일 세션 직렬화             │      │
  │  │ Global Lane:  전체 속도 제한               │      │
  │  │ Message Queue: 스트리밍 중 메시지 큐잉     │      │
  │  └────────────────────────────────────────────┘      │
  │                    |                                  │
  │                    v                                  │
  │  [2] 모델 & 인증 해결                                │
  │  ┌────────────────────────────────────────────┐      │
  │  │ resolveModel()        → 모델 유효성 검증   │      │
  │  │ resolveAuthProfile()  → API 키 로드        │      │
  │  │ contextWindowGuard()  → 토큰 제한 확인     │      │
  │  └────────────────────────────────────────────┘      │
  │                    |                                  │
  │                    v                                  │
  │  [3] 도구 & 스킬 빌드                               │
  │  ┌────────────────────────────────────────────┐      │
  │  │ 정책 해결 순서:                            │      │
  │  │   global → provider → agent → group        │      │
  │  │   → sandbox → subagent                     │      │
  │  │                                            │      │
  │  │ 도구 파이프라인:                           │      │
  │  │   create → filter → normalize → wrap       │      │
  │  │     (정책)  (파라미터)  (훅+abort)          │      │
  │  └────────────────────────────────────────────┘      │
  │                    |                                  │
  │                    v                                  │
  │  [4] Pi SDK 세션 실행                                │
  │  ┌────────────────────────────────────────────┐      │
  │  │ SessionManager.open()  → 히스토리 로드     │      │
  │  │ createAgentSession()   → SDK 초기화        │      │
  │  │ session.subscribe()    → 이벤트 핸들러     │      │
  │  │ session.sendMessage()  → 프롬프트 전송     │      │
  │  └────────────────────────────────────────────┘      │
  │                    |                                  │
  │                    v                                  │
  │  [5] 스트리밍 이벤트 처리                            │
  │  ┌────────────────────────────────────────────┐      │
  │  │ message_start  → 새 응답 시작              │      │
  │  │ message_update → 텍스트 델타 누적          │      │
  │  │ message_end    → 최종 텍스트 확정          │      │
  │  │ tool_exec_*    → 도구 실행 추적            │      │
  │  │ auto_compact   → 컨텍스트 압축 (최대 3회)  │      │
  │  └────────────────────────────────────────────┘      │
  │                    |                                  │
  │                    v                                  │
  │  [6] 에러 복구 전략                                  │
  │  ┌────────────────────────────────────────────┐      │
  │  │ Auth 에러      → 다음 프로필 폴백          │      │
  │  │ 컨텍스트 초과  → 자동 압축 (3회)           │      │
  │  │ 도구 결과 초과 → 결과 잘라내기             │      │
  │  │ 타임아웃       → thinking 레벨 폴백        │      │
  │  │ Rate Limit    → 지수 백오프                │      │
  │  └────────────────────────────────────────────┘      │
  │                                                      │
  └──────────────────────────────────────────────────────┘
```

---

## 데이터 영속화 (Persistence)

```
  ~/.openclaw/
  ├── config.yaml              # 메인 설정 파일
  ├── cron/
  │   ├── jobs.json            # 크론 작업 저장소
  │   ├── jobs.json.bak        # 백업 (매 쓰기 시)
  │   └── runs/
  │       └── <jobId>.jsonl    # 작업별 실행 이력 (JSONL)
  ├── sessions/
  │   └── <agentId>/
  │       └── sessions.json    # 세션 메타데이터
  ├── workspaces/
  │   └── <agentId>/
  │       ├── skills/          # 에이전트별 스킬
  │       └── session-*.json   # 대화 히스토리
  └── extensions/              # 글로벌 확장

  영속화 패턴:
  ┌───────────────────────────────────────────────┐
  │  원자적 쓰기 (Atomic Write)                   │
  │                                               │
  │  1. tmpfile에 새 내용 쓰기                    │
  │  2. 기존 파일 → .bak 으로 백업               │
  │  3. tmpfile → 원본 파일명으로 rename          │
  │                                               │
  │  → 프로세스 크래시 시에도 데이터 손실 방지     │
  └───────────────────────────────────────────────┘
```

---

## 설계 패턴 요약 (Design Pattern Summary)

```
  ┌──────────────────────────┬──────────────────────────────────────┐
  │ 패턴                     │ 적용 위치                            │
  ├──────────────────────────┼──────────────────────────────────────┤
  │ Hub-and-Spoke            │ Gateway 중심 아키텍처                │
  │ Thin Facade + DI         │ CronService, Agent Runtime           │
  │ Plugin Registry          │ 채널, 도구, 훅, 서비스 등록          │
  │ Push-based Registration  │ api.registerXxx() 패턴               │
  │ Discriminated Union      │ CronSchedule, CronPayload 타입      │
  │ Event-Driven Streaming   │ Pi SDK subscribe() 패턴              │
  │ Lane-based Serialization │ 세션별 직렬화, 글로벌 속도 제한      │
  │ Exponential Backoff      │ 크론 에러 복구, API 재시도           │
  │ Atomic Write             │ 크론 저장소, 세션 파일               │
  │ Hierarchical Config      │ 도구 정책 (global→agent→sandbox)     │
  │ Adapter Pattern          │ 채널 플러그인 (정규화 계층)          │
  │ Circuit Breaker          │ Auth 프로필 쿨다운                   │
  │ Request/Response + Event │ WebSocket 3-프레임 프로토콜          │
  │ Session Persistence      │ JSONL 기반 대화 히스토리             │
  │ Crash Recovery           │ runningAtMs 마커 + stale 감지        │
  └──────────────────────────┴──────────────────────────────────────┘
```

---

## 보안 모델 (Security Model)

```
  ┌──────────────────────────────────────────────────────────┐
  │                    Security Layers                        │
  ├──────────────────────────────────────────────────────────┤
  │                                                          │
  │  [1] 연결 인증 (Connection Auth)                         │
  │  ├─ 토큰 기반 / 비밀번호 기반                            │
  │  ├─ 디바이스 서명 검증 (10분 스큐 허용)                   │
  │  └─ Tailscale 신원 확인 (선택)                           │
  │                                                          │
  │  [2] 메서드 인가 (Method Authorization)                   │
  │  ├─ 역할 기반: "operator" vs "node"                      │
  │  ├─ 스코프 기반: admin/read/write/approvals              │
  │  └─ 메서드별: NODE_ROLE, APPROVAL, PAIRING 등            │
  │                                                          │
  │  [3] 채널 보안 (Channel Security)                        │
  │  ├─ DM 페어링 코드 (기본 활성화)                         │
  │  ├─ 허용 목록 (Allowlist)                                │
  │  └─ 소유자 전용 도구 제한                                │
  │                                                          │
  │  [4] 도구 정책 (Tool Policy)                             │
  │  ├─ 다단계 필터: global → agent → sandbox                │
  │  ├─ 서브에이전트 제한 (자식은 부모보다 더 제한적)         │
  │  └─ 샌드박스 실행 (컨테이너 격리)                        │
  │                                                          │
  │  [5] Origin 체크                                         │
  │  └─ 브라우저 오리진 검증 (Control UI)                    │
  │                                                          │
  └──────────────────────────────────────────────────────────┘
```
