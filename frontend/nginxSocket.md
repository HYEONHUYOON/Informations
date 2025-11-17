# Nginx에서 Socket.IO 프록시 설정 정리

## 왜 이 설정이 필요한가
- **WebSocket 업그레이드 보장**: Socket.IO는 기본적으로 `path: /socket.io/`를 사용하며, 초기 핸드셰이크 후 WebSocket(또는 폴백)으로 연결을 업그레이드합니다. 프록시(Nginx)가 이 업그레이드 헤더들을 정확히 전달하지 않으면 연결이 끊기거나 폴링으로만 동작합니다.
- **원본 요청 정보 유지**: 백엔드 앱이 CORS, 로깅, 보안(예: IP 제한) 판단을 위해 `Host`, `X-Forwarded-*` 헤더가 필요합니다.
- **컨테이너/내부 네트워크 라우팅**: 도커/내부 네트워크에서 프런트/백엔드로 트래픽을 안정적으로 전달합니다.

## 설정 스니펫 (주석 포함)
```nginx
location /socket.io/ {
    proxy_http_version 1.1;                 # WebSocket 업그레이드는 HTTP/1.1 필요
    proxy_set_header Upgrade $http_upgrade;  # 클라이언트의 Upgrade 헤더 전달 (websocket 등)
    proxy_set_header Connection "upgrade";   # 연결을 업그레이드하라고 명시
    proxy_pass http://frontend:3000;         # 업스트림 앱 (예: 도커 서비스명:포트)
    proxy_set_header Host $host;             # 원래 Host 유지(CORS/도메인 판단용)
    proxy_set_header X-Real-IP $remote_addr; # 실제 클라이언트 IP 전달
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; # 프록시 체인 IP 누적
    proxy_set_header X-Forwarded-Proto $scheme; # 원래 프로토콜(http/https) 전달
}
```

## 각 줄 설명 (간단 요약)
- **location /socket.io/**
  - Socket.IO 기본 경로만 타깃팅. 이 블록에서만 업그레이드 처리를 강제.
  - 트레일링 슬래시(`/`)와 `proxy_pass`의 슬래시 유무에 따라 리라이트 동작이 바뀜. 현재 형태(`proxy_pass`에 슬래시 없음)는 원래 요청 URI를 그대로 유지하므로 Socket.IO에 적합.

- **proxy_http_version 1.1**
  - WebSocket 핸드셰이크는 HTTP/1.1 기능(Upgrade/Connection)을 요구하므로 필수.

- **proxy_set_header Upgrade $http_upgrade**
  - 클라이언트가 보낸 `Upgrade: websocket` 같은 헤더를 업스트림으로 전달.

- **proxy_set_header Connection "upgrade"**
  - 연결을 업그레이드하도록 명시. 다양한 클라이언트/프록시 조합에서 호환성 확보.
  - 대안: 보다 견고하게 하려면 `map`을 써서 빈 Upgrade일 때 `close`로 설정하는 패턴 사용 가능.

- **proxy_pass http://frontend:3000**
  - Nginx가 트래픽을 전달할 업스트림. 컨테이너 환경이라면 `frontend`는 서비스명으로 DNS 해석됨.
  - 슬래시 없는 형태는 원 URI를 그대로 전달(경로 리라이트 없음).

- **proxy_set_header Host $host**
  - 앱이 요청 도메인/Origin 기반 로직(CORS, 멀티테넌시)을 처리할 수 있도록 원 Host 유지.

- **X-Real-IP / X-Forwarded-For**
  - 로깅/보안(레이트 리미트, GeoIP 등)을 위해 실제 클라이언트 IP를 앱에서 식별 가능하게 함.

- **X-Forwarded-Proto**
  - TLS 종료가 Nginx 앞단/로드밸런서에서 이뤄질 때, 앱이 원래 스킴(http/https)을 인지하게 함. `wss`/`ws` 판단에도 유용.

## 추가 팁
- **타임아웃**: 장기 연결 환경에선 `proxy_read_timeout 600s;` 같은 값을 `/socket.io/` 블록에 추가하면 끊김을 줄일 수 있음.
- **버퍼링**: 필요 시 `proxy_buffering off;`로 스트리밍/실시간성 개선.
- **CORS/Origin**: 앱에서 허용 도메인과 `Origin` 검사 로직을 맞추고, 필요한 경우 `proxy_set_header Origin $http_origin;`를 고려.
- **멀티 업스트림**: 다중 인스턴스/로드밸런싱 시 세션 스티키 또는 Socket.IO 어댑터(예: Redis)를 병행해 수평 확장 안정화.

