 ## 개요
 - Next.js 14에서 무거운 작업을 API 타임아웃과 분리하기 위해, Redis + BullMQ 워커로 백그라운드 실행하고 Socket.IO로 실시간 진행률/결과를 주고받는 패턴을 정리합니다.
 - 핵심 아이디어: API는 "작업 등록(큐 enqueue) + jobId 반환"만 담당하고, 워커가 실제 연산을 수행하며, QueueEvents → Socket.IO → 클라이언트로 실시간 브릿지합니다.

 ---

 ## 전체 아키텍처 한눈에 보기
 - 클라이언트(App Router 클라이언트 컴포넌트)
   - 작업 등록 API 호출 → `{ jobId }` 수신
   - 소켓 연결 후 `joinRoom(jobId)`로 방 입장 → 진행률/완료/실패 이벤트 수신
 - Next.js 14 API(Route Handlers 또는 pages API)
   - `POST /api/<job>`에서 BullMQ Queue에 `add()`만 수행 → `202 Accepted + jobId` 즉시 반환
 - Redis + BullMQ
   - Queue: 작업 등록 지점
   - Worker: 실제 heavy job 처리, `job.updateProgress(%)` 호출
   - QueueEvents: `progress/completed/failed` 이벤트 스트림
 - Socket.IO 서버(Next API 위)
   - 1회 초기화된 Socket.IO 인스턴스가 QueueEvents를 구독해 같은 `jobId` 방으로 이벤트 emit
 - Nginx(선택: Docker/운영)
   - `/socket.io/` WebSocket 업그레이드 처리로 프록시 뒤에서도 연결 유지

 ---

 ## 동작 흐름(시퀀스)
 1) 사용자가 클라이언트에서 작업 요청(예: PDF 생성, 대량 처리)
 2) 클라이언트가 필요한 데이터를 선취득(필요 시) 후, `POST /api/<job>` 호출
 3) 서버는 BullMQ Queue에 작업 등록 → `202`로 `{ jobId }` 반환(타임아웃과 분리)
 4) 클라이언트는 즉시 Socket.IO 연결 → `joinRoom(jobId)`로 조인
 5) 워커가 작업 처리 중 `job.updateProgress(%)` → QueueEvents가 발행
 6) Socket 서버가 QueueEvents를 수신해 같은 방으로 `progress/completed/failed` emit
 7) 클라이언트는 실시간 진행률 UI 갱신, 완료 시 결과(URL/데이터)로 후처리(다운로드/표시 등)

 ---

 ## Next.js 14 구현 포인트
 ### 1) API 라우트 구조(App Router 권장)
 - `app/api/<job>/route.ts`의 `POST`에서 BullMQ Queue에 `add()` 후 `jobId`만 반환합니다.
 - pages API와 혼용 가능하나, Socket.IO 서버는 pages API(`pages/api/socket.ts`) 패턴이 단순합니다.

 ### 2) Socket.IO 서버를 Next API 위에 올리기
 - `(res.socket as any).server`에 한 번만 `new Server()`로 Socket.IO 인스턴스 생성하고 재사용합니다.
 - 전역 플래그(예: `global.__qe_init__`)로 BullMQ `QueueEvents` 중복 구독을 방지합니다.
 - `QueueEvents('queue-name')`에서 `progress/completed/failed`를 수신해 `io.to(jobId).emit(...)`으로 브릿지합니다.

 ### 3) 클라이언트 훅(`useSocket`)
 - 최초 한 번 `/api/socket` 호출로 서버/QueueEvents 초기화 → 이후 `io()`로 현재 오리진 기준 연결
 - `joinRoom(jobId)`로 방 입장, `on/once/off/emit` 인터페이스 제공
 - 주의: 반드시 클라이언트 컴포넌트(`'use client'`)에서 사용합니다.

 ---

 ## Redis · BullMQ 설정
 - `REDIS_URL` 예시
   - 로컬: `redis://localhost:6379/0`
   - Docker: `redis://redis_container:6379/0`
 - ioredis 옵션(중요)
   - `maxRetriesPerRequest: null`
   - `enableReadyCheck: false`
 - Queue/Worker/QueueEvents 역할
   - Queue: API에서 `add()`로 작업 등록
   - Worker: 별도 프로세스/컨테이너에서 상시 실행, `job.updateProgress(%)` 사용
   - QueueEvents: 진행률/완료/실패 이벤트 스트림 → Socket.IO로 전달

 ---

 ## Socket.IO · Nginx · Docker 관점
 ### 브라우저 연결 규칙
 - 클라이언트는 보통 `io()`만 호출(호스트 생략) → 현재 오리진의 `/socket.io/`로 연결 시도
 - `useSocket`은 `/api/socket`을 먼저 호출해 서버 인스턴스를 준비시킨 뒤 연결합니다.

 ### Nginx 뒤에서 WebSocket 업그레이드(중요)
 - `/socket.io/` 전용 location에 다음이 필요합니다.
   - `proxy_http_version 1.1;`
   - `proxy_set_header Upgrade $http_upgrade;`
   - `proxy_set_header Connection "upgrade";`
   - `proxy_pass http://frontend:3000;`
 - 일반 요청은 `/`에서 동일하게 `proxy_pass http://frontend:3000;` 처리합니다.

 ### Docker Compose 서비스 역할(예시)
 - `redis_container`: Redis
 - `frontend`: Next.js(3000) – pages API로 Socket.IO 서버, App Router로 작업 등록 API
 - `worker`: BullMQ 워커 – `npm run worker:report` 등으로 큐 소비, Puppeteer/연산 수행
 - `nginx`: 리버스 프록시(80/443) – `/socket.io/` 업그레이드 처리

 ---

 ## 데이터 계약(예시)
 - 작업 등록 API(202)
   - Res: `{ jobId: string }`
 - Socket.IO 이벤트
   - `joinRoom(jobId: string)`
   - `report:progress` → `{ jobId, progress: number }`
   - `report:completed` → `{ jobId, result: any }`(예: `{ url: string }`)
   - `report:failed` → `{ jobId, reason: string }`

 ---

 ## 요약
 - 타임아웃 없는 안정적 처리 = "API는 큐 등록", "워커는 처리", "소켓은 실시간 전달"로 역할을 분리하는 것이 핵심입니다.
 - UI는 진행률/완료 이벤트에 즉각 반응하도록 설계(프로그레스바/상태 메시지)하면 체감 품질이 높아집니다.
