# 백엔드 설계 및 레포 구조 (architecture.md)

본 문서는 `ux-flow.md`의 UX 명세를 구현하기 위한 백엔드 설계와 레포 전체 파일 구조를 확정한다. 기술 스택은 기존 ANA 웹사이트 레포의 구성을 계승하되, 실시간 세션과 벡터 연산 요구사항에 따라 일부를 교체한다.

---

## 1. 기술 스택

| 계층 | 선택 | 비고 |
|---|---|---|
| 언어 | **TypeScript** | 계승 |
| 런타임 | **Node.js 22 LTS** | 계승 |
| 웹 프레임워크 | **Express 5** | 계승 |
| 검증 | **Zod** | express-validator 교체 |
| 실시간 | **Socket.IO** + `@socket.io/redis-adapter` | 신규 |
| DB | **PostgreSQL 16 + pgvector** | MongoDB 교체 |
| ORM | **Drizzle ORM** + drizzle-kit | Mongoose 교체 |
| 큐·타이머 | **BullMQ** (Redis 7) | node-cron 교체 |
| 인증 | **jsonwebtoken** + **bcrypt** + passport | 계승 |
| 보안 | **helmet**, **cors** | 계승 |
| 객체 저장 | **MinIO** (S3 API) + multer | 계승 |
| ID | **nanoid** | 계승 |
| 테스트 | **Vitest** | 신규 |
| 인프라 | **Docker Compose**, GitHub Actions | 계승 |

### 1.1 교체 항목의 근거

- **MongoDB → PostgreSQL** — 카드·임베딩·배치좌표·클러스터·패싯태그가 전부 조인되는 관계형 구조다. 개입 판정은 다수 테이블을 가로지르는 시간 윈도 집계이며, 여기에 벡터 유사도 연산이 결합된다. pgvector를 사용하면 단일 DB에서 처리되고 카드와 벡터의 트랜잭션 일관성이 보장된다.
- **node-cron → BullMQ** — node-cron은 주기 실행만 지원하며 개별 시각 예약이 불가능하다. `setTimeout` 구현은 프로세스 재시작 시 진행 중인 모든 라운드를 소실시킨다. BullMQ의 **지연 작업(delayed job)** 은 예약 상태를 Redis에 보존해 재시작을 견딘다.
- **express-validator → Zod** — 프론트엔드·백엔드·AI 서비스 3자가 동일 스키마를 소비해야 한다. Zod는 단일 스키마로 런타임 검증, TS 타입 추론, JSON Schema 추출을 모두 제공한다.

---

## 2. 배포 토폴로지

동아리 서버 자원을 두 노드로 분리해 사용한다.

| 노드 | 사양 | 역할 |
|---|---|---|
| **app-node** | 64 Core / 128 GB | API 프로세스 × N, 워커 프로세스, PostgreSQL, Redis, MinIO, nginx |
| **gpu-node** | 32 Core / 64 GB / RTX 5080 | AI 서비스 (STT, 화자 분리, 임베딩, 클러스터링, LLM 라벨링) |

- **GPU 연산을 별도 노드로 분리하는 것이 필수다.** Whisper 추론과 임베딩 인코딩이 app-node에서 실행되면 API 응답과 CPU를 경합해 실시간 전사가 지연된다.
- 두 노드 간 통신은 **내부 네트워크 HTTP + 공유 MinIO**로 구성한다. 오디오 청크는 백엔드가 MinIO에 적재하고, AI 서비스에 객체 키만 전달한다. 바이너리를 HTTP 본문으로 왕복시키지 않는다.
- **API 프로세스를 코어 수에 맞춰 다중 기동한다.** 이 시점에서 Socket.IO의 Redis 어댑터가 필수가 된다. 어댑터 없이는 같은 방의 참가자가 서로 다른 프로세스에 접속했을 때 브로드캐스트가 전달되지 않는다.

### 2.1 백업

`pg_dump`를 일 1회 실행해 app-node 외부 위치에 보관하는 크론을 **구축 첫날에 설정한다.** 관리형 DB를 쓰지 않는 구성이므로 백업 부재는 전체 세션 데이터 소실로 직결된다.

---

## 3. 백엔드 계층 설계

NestJS를 사용하지 않으므로 DI 컨테이너가 없다. 계층 분리를 폴더 규칙과 **명시적 조립 루트(composition root)** 로 대체한다.

### 3.1 계층 정의

| 계층 | 책임 | I/O 허용 |
|---|---|---|
| `routes/`, `ws/` | 전송 계층. 요청 파싱, 응답 직렬화 | — |
| `services/` | 유스케이스 조립. 트랜잭션 경계 | 허용 |
| `domain/` | 순수 로직. 상태 전이, 지표, 판정 | **금지** |
| `repositories/` | DB 접근 (Drizzle) | 허용 |
| `infra/` | DB·Redis·큐·스토리지·AI 클라이언트 연결 | 허용 |

**`domain/`의 I/O 금지가 설계의 핵심이다.** 개입 판정과 지표 계산이 DB나 네트워크에 결합되면 고정 입력에 대한 회귀 테스트가 불가능해진다. 개입 효과를 검증 지표로 증명하려면 이 계층이 결정론적이어야 한다.

### 3.2 의존성 주입 규칙

DI 프레임워크 대신 **팩토리 함수 패턴**을 사용한다. 모든 라우터와 서비스는 의존성을 인자로 받는 팩토리로 정의하고, `container.ts`에서 한 번만 조립한다.

```ts
// services/room.service.ts
export const makeRoomService = (deps: {
  roomRepo: RoomRepository;
  participantRepo: ParticipantRepository;
  io: Server;
}) => ({
  async create(input: CreateRoomInput) { /* ... */ },
});
export type RoomService = ReturnType<typeof makeRoomService>;

// routes/room.routes.ts
export const makeRoomRouter = (svc: RoomService) => {
  const r = Router();
  r.post('/', async (req, res) => { /* ... */ });
  return r;
};
```

- **모듈 최상위에서 DB 클라이언트나 Redis 인스턴스를 직접 import하는 것을 금지한다.** 이를 허용하면 단위 테스트 시 실제 연결이 열리고, 계층 경계가 무의미해진다.
- `container.ts`는 API 프로세스와 워커 프로세스가 공유하되, 각 진입점에서 필요한 부분만 조립한다.

### 3.3 프로세스 분리

| 진입점 | 역할 |
|---|---|
| `src/main.api.ts` | Express + Socket.IO |
| `src/main.worker.ts` | BullMQ 워커, 스케줄러 |

**워커를 API와 분리하는 것이 필수다.** 동일 프로세스에 두면 라운드 종료 처리가 요청 부하에 밀려 타이머 정확도가 무너진다.

---

## 4. 큐 설계

| 큐 | 유형 | 트리거 | 처리 |
|---|---|---|---|
| `round` | 지연 작업 | 라운드 시작 시 `delay = 라운드 시간` | 라운드 종료, `ROUND_REVIEW` 전이 |
| `intervention` | 반복 작업 | `ROUND_ACTIVE` 동안 15초 주기 | 트리거 3종 평가 |
| `transcribe` | 즉시 | 오디오 청크 업로드 완료 | AI 서비스 호출 → 카드 생성 |
| `embed` | 즉시 | 카드 생성 직후 | 임베딩 생성 → `embeddings` 적재 |
| `synthesis` | 즉시 | `SYNTHESIS` 진입 | 투영·클러스터링·라벨링 |

- 라운드가 종료되거나 세션이 조기 종료되면 해당 라운드의 `round` 작업과 `intervention` 반복 작업을 **명시적으로 제거한다.** 제거하지 않으면 종료된 라운드에 대한 전이 시도가 발생한다.
- 모든 작업은 `sessionId`를 jobId 접두어로 사용해 세션 단위 일괄 취소를 가능하게 한다.

---

## 5. 상태 머신

```
LOBBY → SETUP → ROUND_ACTIVE → ROUND_REVIEW
                     ↑                ↓
                     └─ TARGETED_ROUND ┘
                                      ↓
                             SYNTHESIS → POST_PROCESSING → CLOSED
```

전이 표는 `domain/session.ts`에 상수로 정의하고, 서버만이 전이를 수행한다. 클라이언트는 전이 요청만 전송한다.

| 현재 | 이벤트 | 다음 | 권한 |
|---|---|---|---|
| LOBBY | `session:setup` | SETUP | 호스트 |
| SETUP | `session:start` | ROUND_ACTIVE | 호스트 |
| ROUND_ACTIVE | `round:timeout` | ROUND_REVIEW | 서버 |
| ROUND_ACTIVE | `round:stop` | ROUND_REVIEW | 호스트 |
| ROUND_REVIEW | `round:continue` | TARGETED_ROUND | 호스트 |
| ROUND_REVIEW | `round:finish` | SYNTHESIS | 호스트 |
| TARGETED_ROUND | `targeted:timeout` | ROUND_ACTIVE | 서버 |
| SYNTHESIS | `synthesis:done` | POST_PROCESSING | 서버 |
| POST_PROCESSING | `session:close` | CLOSED | 호스트 |

---

## 6. Socket.IO 설계

### 6.1 네임스페이스와 룸

- 네임스페이스는 기본(`/`) 하나만 사용한다
- 룸 키는 `room:{roomId}`, `session:{sessionId}`, `user:{participantId}` 세 가지다
- **`user:{participantId}` 룸이 개별 알림의 전달 경로다.** 개입 알림을 전체 공지로 보내지 않는다는 명세 요구사항이 이 룸으로 구현된다

### 6.2 핸드셰이크

연결 시 JWT를 검증하고 `socket.data`에 `participantId`, `roomId`, `isHost`를 적재한다. 게스트와 로그인 사용자 모두 JWT를 보유하므로 검증 경로가 단일화된다. 검증 실패 시 연결을 거부한다.

### 6.3 이벤트

**서버 → 클라이언트**

| 이벤트 | 대상 | 페이로드 |
|---|---|---|
| `room:participants` | room | 참가자 목록 |
| `room:kicked` | user | 사유 |
| `session:state` | session | 상태, 라운드 번호, 서버 시각 |
| `round:tick` | session | 종료 예정 시각 |
| `idea:created` | session | 카드 (원문, 화자, 타임스탬프) |
| `idea:duplicate_hint` | session | 중복 후보 ID 쌍 |
| `agent:nudge` | **user** | 개입 메시지, 강도 |
| `agent:target_cell` | session | 지목된 패싯 셀 |
| `canvas:ready` | session | 노드·클러스터·확신도 |

**클라이언트 → 서버**

| 이벤트 | 페이로드 |
|---|---|
| `idea:submit` | 텍스트 |
| `audio:chunk_uploaded` | MinIO 객체 키 |
| `canvas:node_moved` | 노드 ID, 좌표 |
| `session:transition` | 전이 이벤트명 |

`round:tick`은 **잔여 시간이 아니라 종료 예정 시각을 전송한다.** 잔여 시간을 보내면 네트워크 지연이 누적되어 클라이언트 간 타이머가 어긋난다.

---

## 7. 데이터 모델

```
users              id, provider, nickname, created_at
rooms              id, name, host_id, is_private, password_hash, created_at
room_bans          room_id, participant_id, banned_at
participants       id, room_id, user_id(nullable), nickname, is_guest, is_host

sessions           id, room_id, state, problem_statement, input_mode,
                   target_per_person, round_limit, round_duration_sec
facet_axes         id, session_id, axis_index, label, values[]
rounds             id, session_id, index, kind(normal|targeted),
                   started_at, ends_at, ended_at

ideas              id, session_id, round_id, participant_id,
                   text, source(text|voice), audio_key, audio_start_ms,
                   audio_end_ms, parent_idea_id, created_at
embeddings         idea_id, vector(vector), model, created_at
facet_tags         idea_id, axis_id, value

clusters           id, session_id, label, approved_by
cluster_members    cluster_id, idea_id, membership_prob
placements         id, session_id, idea_id, x, y, source(auto|user),
                   participant_id, created_at

interventions      id, session_id, round_id, trigger, level,
                   target_participant_id, payload, created_at
exports            id, session_id, format, object_key, created_at
```

### 7.1 적재 원칙

- **`placements`는 이력으로 적재한다.** 최종 좌표만 남기면 거리 보정의 학습 신호와 개입 효과 검증의 원자료가 동시에 소실된다. `source` 컬럼으로 자동 배치와 사용자 드래그를 구분한다.
- **`ideas`는 원문·오디오 키·구간 타임스탬프를 함께 보관한다.** 카드에서 원본 음성으로 역참조가 불가능하면 신뢰가 붕괴한다.
- **`interventions`는 발생 시각과 트리거 종류를 기록한다.** 개입 전후 커버리지 증가분과 최근접 이웃 거리 변화를 산출하려면 개입 시각이 필요하다.
- `cluster_members.membership_prob`가 배치 확신도 시각화의 입력이다.

### 7.2 인덱스

- `ideas(session_id, created_at)` — 시간 윈도 집계
- `ideas(session_id, participant_id)` — 개인 목표 달성도
- `embeddings` — HNSW 인덱스 (코사인 거리)
- `placements(idea_id, created_at)` — 최신 좌표 조회

---

## 8. 지표 계산 위치

| 지표 | 계산 위치 | 근거 |
|---|---|---|
| 카드 개수 | 백엔드 `domain/metrics.ts` | 벡터 불필요 |
| 화자 점유율 | 백엔드 `domain/metrics.ts` | 벡터 불필요 |
| 중복도 | AI 서비스 | 벡터 필요 |
| 커버리지 | AI 서비스 | **원본 임베딩 차원에서 계산** |
| 개별 신규성 | AI 서비스 | 벡터 필요 |
| **최종 개입 판정** | **백엔드 `domain/intervention.ts` 단독** | 재현성 |

판정 로직이 양쪽에 분산되면 트리거 재현이 불가능해지므로, AI 서비스는 지표 값만 반환하고 판정은 백엔드가 단독으로 수행한다.

커버리지를 2D 투영 결과가 아닌 원본 벡터로 계산하는 이유는 UMAP·t-SNE 여백이 투영 아티팩트이기 때문이다. 이를 코드 구조로 강제하기 위해 `metrics/diversity.py`는 `cluster/project.py`를 import하지 않는다.

---

## 9. 레포 파일 구조

### 9.1 루트

```
repo/
├── ai/
├── backend/
├── frontend/
├── packages/
│   └── contracts/              # Zod 스키마, WS 이벤트 타입, 상태 전이 표
├── nginx/
├── docs/
│   ├── ux-flow.md
│   ├── architecture.md
│   └── research-references.md
├── .github/workflows/
├── docker-compose.yml          # 로컬 인프라
├── docker-compose.prod.yml     # app-node 배포
├── docker-compose.gpu.yml      # gpu-node 배포
├── pnpm-workspace.yaml
└── turbo.json
```

**`packages/contracts`는 선택이 아니다.** WS 이벤트 타입과 요청 스키마를 backend와 frontend에 각각 두면 반드시 어긋난다. pnpm workspace 패키지로 분리하고, Python은 여기서 생성한 JSON Schema를 소비한다.

```
packages/contracts/src/
├── schemas/            # Zod — room, session, idea, canvas
├── events/             # WS 이벤트 타입 (서버↔클라 공유)
├── state-machine.ts    # 전이 표 (백엔드가 판정에 사용, 프론트는 UI 분기에 사용)
└── scripts/to-json-schema.ts   # AI 서비스용 JSON Schema 생성
```

### 9.2 backend/

```
backend/
├── package.json
├── Dockerfile
├── drizzle.config.ts
├── drizzle/                        # 마이그레이션
├── tests/
│   ├── unit/                       # domain/ 전용, I/O 없음
│   └── integration/
└── src/
    ├── main.api.ts                 # 진입점 1: Express + Socket.IO
    ├── main.worker.ts              # 진입점 2: BullMQ
    ├── container.ts                # 조립 루트
    ├── config.ts                   # 환경변수 (Zod 검증)
    │
    ├── routes/
    │   ├── auth.routes.ts
    │   ├── room.routes.ts
    │   ├── session.routes.ts
    │   ├── idea.routes.ts
    │   ├── audio.routes.ts         # multer → MinIO presigned
    │   ├── canvas.routes.ts
    │   └── export.routes.ts
    │
    ├── ws/
    │   ├── index.ts                # Socket.IO 초기화, redis adapter
    │   ├── auth.ts                 # 핸드셰이크 JWT 검증
    │   ├── rooms.ts                # 룸 키 규칙
    │   └── handlers/
    │       ├── idea.handler.ts
    │       ├── canvas.handler.ts
    │       └── session.handler.ts
    │
    ├── domain/                     # ★ I/O 금지
    │   ├── room.ts                 # 입장 규칙, 호스트 권한
    │   ├── session.ts              # 상태 전이 판정
    │   ├── round.ts                # 종료 조건
    │   ├── facet.ts                # 크로스탭, 빈 셀 탐색
    │   ├── metrics.ts              # 개수, 화자 점유율
    │   └── intervention.ts         # 트리거 판정, 강도 결정
    │
    ├── services/
    │   ├── auth.service.ts
    │   ├── room.service.ts
    │   ├── session.service.ts
    │   ├── idea.service.ts
    │   ├── agent.service.ts        # AI 호출, 결과 반영
    │   └── export.service.ts
    │
    ├── repositories/
    │   ├── room.repo.ts
    │   ├── session.repo.ts
    │   ├── idea.repo.ts
    │   ├── placement.repo.ts
    │   └── intervention.repo.ts
    │
    ├── db/
    │   ├── client.ts
    │   └── schema/                 # Drizzle 테이블 정의
    │
    ├── infra/
    │   ├── redis.ts
    │   ├── queue.ts                # BullMQ 큐 정의
    │   ├── storage.ts              # MinIO
    │   └── ai.client.ts            # gpu-node HTTP 클라이언트
    │
    ├── workers/
    │   ├── round.worker.ts
    │   ├── intervention.worker.ts
    │   ├── transcribe.worker.ts
    │   ├── embed.worker.ts
    │   └── synthesis.worker.ts
    │
    └── middleware/
        ├── validate.ts             # Zod 미들웨어
        ├── auth.ts
        └── error.ts
```

### 9.3 ai/

```
ai/
├── pyproject.toml
├── Dockerfile                      # CUDA 베이스
├── scripts/download_models.py
├── models/                         # 가중치 (gitignore)
├── contracts/                      # packages/contracts에서 생성된 JSON Schema
├── tests/fixtures/                 # 고정 아이디어 집합 → 결정론 검증
└── src/
    ├── server.py                   # FastAPI
    ├── stt/
    │   ├── transcribe.py
    │   └── diarize.py
    ├── atomize/
    │   ├── splitter.py             # 원문·타임스탬프 보존
    │   └── prompts/
    ├── embed/
    │   ├── encoder.py
    │   └── cache.py
    ├── cluster/
    │   ├── project.py              # UMAP
    │   ├── cluster.py              # HDBSCAN
    │   ├── label.py                # LLM 라벨링
    │   └── calibrate.py            # 드래그 좌표 → 거리 보정
    ├── facet/classify.py
    ├── metrics/diversity.py        # ★ cluster/project.py import 금지
    └── synth/draft.py
```

### 9.4 frontend/

```
frontend/
├── package.json
├── .eslintrc.cjs                   # ★ 시각화 잠금 규칙
└── src/
    ├── pages/
    │   └── rooms/
    │       ├── Lobby.tsx
    │       ├── Setup.tsx
    │       ├── Round.tsx           # 시각화 잠금 구간
    │       ├── Review.tsx
    │       └── Canvas.tsx          # 시각화 개방 구간
    ├── features/
    │   ├── room/
    │   ├── session/
    │   ├── ideation/               # 라운드 입력·카드 목록
    │   ├── agent/                  # 개입 배너, 빈 셀 지목
    │   ├── canvas/                 # 2D 노드, 드래그
    │   └── export/
    ├── shared/
    │   ├── ws/                     # Socket.IO 클라이언트
    │   ├── api/                    # axios
    │   ├── store/                  # zustand
    │   └── ui/
    └── types/
```

**시각화 잠금을 린트로 강제한다.** `no-restricted-imports`로 `pages/rooms/Round.tsx`와 `features/ideation/**`에서 `features/canvas/**` import를 금지한다. 규칙으로 고정하지 않으면 개발 편의를 이유로 캔버스가 발산 화면에 유입되고, 고착 방지라는 설계 근거가 무효화된다. 규칙 파일에 금지 사유를 주석으로 명시한다.

---

## 10. 로컬 개발 구성

인프라만 컨테이너로 올리고 애플리케이션은 호스트에서 직접 기동한다.

```yaml
# docker-compose.yml
services:
  postgres: pgvector/pgvector:pg16
  redis:    redis:7
  minio:    minio/minio
```

로컬 DB를 공용 인스턴스로 대체하지 않는다. 팀원 전원이 동일 DB를 공유하면 스키마 실험이 상호 간섭한다.

---

## 11. 구현 순서

의존 관계상 아래 순서를 권장한다.

1. `packages/contracts` — 스키마와 이벤트 타입 확정
2. `backend/db/schema` + 마이그레이션
3. 인증 및 룸 (P0~P1) — WS 핸드셰이크 포함
4. 세션 상태 머신 + `round` 큐 (P2~P3) — **여기까지가 골격이다**
5. 텍스트 모드 카드 생성 — 음성 없이 전체 흐름 검증
6. AI 서비스 임베딩·클러스터링 (P6)
7. 지표·개입·타겟 라운드 (P4~P5)
8. 음성 입력 및 화자 분리
9. 포스트 프로세싱·내보내기 (P7~P8)

**음성을 마지막에 붙이는 것이 중요하다.** 텍스트 모드로 전체 파이프라인이 동작하는 상태를 먼저 확보하면, 음성 관련 문제가 발생해도 서비스 골격은 검증된 상태로 남는다.
