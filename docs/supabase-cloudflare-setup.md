# Supabase + Cloudflare Tunnel 설정 가이드

이 문서는 Cloudflare Tunnel을 이용해 `supabase.neocompany.dev` 도메인으로 self-hosted Supabase(우분투, Docker 환경)를 안전하게 노출하는 방법을 안내합니다.

---

## 1. Cloudflare Tunnel 설정 개요

Cloudflare Tunnel은 서버가 방화벽/NAT 뒤에 있어도 안전하게 외부에서 접근할 수 있게 해주는 서비스입니다. 이 가이드에서는 Docker로 실행되는 Supabase를 Cloudflare Tunnel과 연동하여 안전한 HTTPS 접속을 제공하는 방법을 설명합니다.

### 최종 목표
- Supabase를 Docker로 실행
- Cloudflare Tunnel을 통해 안전한 외부 접속 설정
- `https://supabase.neocompany.dev`로 접속 가능

---

## 2. Cloudflared CLI 설치 및 Tunnel 설정

### 2-1. Cloudflared CLI 설치 (Ubuntu)

```bash
# 최신 안정 버전 다운로드
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64 -o cloudflared

# 실행 권한 부여
chmod +x cloudflared

# 시스템 경로에 이동
sudo mv cloudflared /usr/local/bin/

# 설치 확인
cloudflared --version
```

### 2-2. Cloudflare 계정 인증

```bash
# Cloudflare 계정 로그인
cloudflared login
```

- 브라우저가 자동으로 열리면서 Cloudflare 인증 페이지로 이동합니다.
- `neocompany.dev` 도메인을 선택하고 인증을 완료합니다.
- 성공 시 `~/.cloudflared/cert.pem` 파일이 생성됩니다.

### 2-3. Tunnel 생성

```bash
# Tunnel 생성
cloudflared tunnel create supabase-tunnel
```

- 터널 생성 시 `~/.cloudflared/` 디렉토리에 UUID 형식의 이름을 가진 인증 파일(예: `2a216d05-a8df-4ffc-9461-ee26ce39ee70.json`)이 자동으로 생성됩니다.
- 생성된 터널과 파일을 확인:

```bash
cloudflared tunnel list
ls -la ~/.cloudflared/
```

### 2-4. DNS 레코드 자동 연결

```bash
# DNS 레코드 생성
cloudflared tunnel route dns supabase-tunnel supabase.neocompany.dev
```

- 이 명령어를 실행하면 Cloudflare DNS에 자동으로 CNAME 레코드가 생성됩니다.
- CNAME은 `supabase.neocompany.dev → *.cfargotunnel.com`으로 설정됩니다.
- Cloudflare TLS 인증서도 자동으로 적용됩니다.

---

## 3. Supabase Docker Compose와 Cloudflared 통합

### 3-1. 프로젝트 디렉토리 구조 설정

```bash
# 프로젝트 디렉토리로 이동 (예: Supabase 프로젝트 루트)
cd ~/infra/supabase

# cloudflared 설정 파일을 저장할 디렉토리 생성
mkdir -p cloudflared

# 인증 파일과 인증서 파일 복사
# 아래에서 TUNNEL_ID.json은 실제 생성된 파일명으로 대체해야 합니다
# (예: 2a216d05-a8df-4ffc-9461-ee26ce39ee70.json)
cp ~/.cloudflared/cert.pem cloudflared/
cp ~/.cloudflared/TUNNEL_ID.json cloudflared/

# 실제 터널 인증 파일명 확인 방법
ls -la ~/.cloudflared/ | grep .json
```

### 3-2. Cloudflared 구성 파일 작성

```bash
# cloudflared 구성 파일 생성
nano cloudflared/config.yml
```

구성 파일에 다음 내용을 입력합니다:

```yaml
tunnel: supabase-tunnel
credentials-file: /etc/cloudflared/TUNNEL_ID.json
origincert: /etc/cloudflared/cert.pem

ingress:
  - hostname: supabase.neocompany.dev
    service: http://kong:8000
  - service: http_status:404
```

> **중요**: 
> - `TUNNEL_ID.json`을 실제 생성된 파일명(예: `2a216d05-a8df-4ffc-9461-ee26ce39ee70.json`)으로 대체하세요.
> - `origincert` 설정이 없으면 인증 오류가 발생할 수 있습니다.
> - `service` 값이 `http://kong:8000`인 것에 주목하세요. Docker Compose 네트워크 내에서는 컨테이너 이름(`kong`)으로 서비스에 접근할 수 있습니다.

### 3-3. Docker Compose 파일 수정

기존 `docker/docker-compose.yml` 파일을 수정하여 `cloudflared` 서비스를 추가합니다:

```bash
# 기존 docker-compose.yml 파일을 백업
cp docker/docker-compose.yml docker/docker-compose.yml.bak

# docker-compose.yml 파일 수정
nano docker/docker-compose.yml
```

파일의 `services` 섹션 맨 아래, `volumes` 선언 바로 위에 다음 내용을 추가합니다:

```yaml
  cloudflared:
    container_name: supabase-cloudflared
    image: cloudflare/cloudflared:latest
    restart: unless-stopped
    volumes:
      - ./cloudflared:/etc/cloudflared
    depends_on:
      kong:
        condition: service_started
    command: tunnel --no-autoupdate run supabase-tunnel
    healthcheck:
      test:
        [
          "CMD",
          "cloudflared",
          "tunnel",
          "info"
        ]
      interval: 10s
      timeout: 5s
      retries: 3
```

이 예제에서는 Supabase docker-compose.yml 파일이 이미 컨테이너 간 통신을 위한 기본 네트워크를 정의하고 있으므로, 별도의 네트워크 설정은 필요하지 않습니다. Supabase에서는 `name: supabase`로 이미 네트워크가 정의되어 있습니다.

> **참고**: 
> - `command`에 반드시 터널 이름(`supabase-tunnel`)을 포함해야 합니다.
> - Supabase docker-compose.yml 파일에서는 대부분의 서비스가 health check와 다른 서비스에 대한 의존성을 정의하고 있습니다.

### 3-4. Supabase 아키텍처 이해

Supabase는 여러 컨테이너로 구성되어 있으며, 외부 접속은 Kong API Gateway를 통해 이루어집니다:

- **Kong API Gateway**: 외부 요청을 받아 내부 서비스로 라우팅하는 API 게이트웨이
- **Studio**: Supabase 관리 UI
- **Auth, REST, Realtime, Storage 등**: 다양한 백엔드 서비스

`docker-compose.yml`에서 Kong 서비스는 다음과 같이 포트를 노출합니다:

```yaml
kong:
  container_name: supabase-kong
  ports:
    - ${KONG_HTTP_PORT}:8000/tcp
    - ${KONG_HTTPS_PORT}:8443/tcp
```

일반적으로 환경 변수는 다음과 같이 설정됩니다:
- `KONG_HTTP_PORT`: 8000
- `KONG_HTTPS_PORT`: 8443

### 3-5. Docker Compose로 Supabase 및 Cloudflared 실행

```bash
# docker-compose.yml 파일이 있는 디렉토리로 이동
cd ~/infra/supabase

# Docker Compose로 모든 서비스 실행
docker compose -f docker/docker-compose.yml up -d

# 모든 컨테이너 상태 확인
docker ps

# Cloudflared 로그 확인
docker logs -f supabase-cloudflared
```

### 3-6. 작동 확인

웹 브라우저에서 다음 URL에 접속합니다:

```
https://supabase.neocompany.dev
```

성공적으로 설정되었다면:
- 브라우저에 🔒 아이콘(HTTPS 보안 연결)이 표시됩니다
- Supabase Studio UI가 정상적으로 로드됩니다

---

## 4. 문제 해결 가이드

### 4-1. 연결 문제 해결

**cloudflared 컨테이너가 시작되지 않을 경우:**

```bash
# 자세한 로그 확인
docker logs supabase-cloudflared

# 컨테이너 내부 확인
docker exec -it supabase-cloudflared sh
ls -la /etc/cloudflared
cat /etc/cloudflared/config.yml
```

**인증 관련 오류가 발생할 경우:**

가장 흔한 오류 중 하나는 인증서 파일 또는 터널 인증 파일 문제입니다:

```bash
# 다음 오류가 발생할 경우
# "Cannot determine default origin certificate path" 또는
# "error parsing tunnel ID: Error locating origin cert"

# 인증 파일이 올바른 위치에 있는지 확인
docker exec -it supabase-cloudflared sh -c "ls -la /etc/cloudflared"

# config.yml 파일에 origincert 설정이 있는지 확인
docker exec -it supabase-cloudflared sh -c "cat /etc/cloudflared/config.yml"
```

**Kong에 연결할 수 없는 경우:**

Docker 네트워크 내에서 Kong 컨테이너가 올바르게 실행 중인지 확인:

```bash
docker ps | grep kong
docker exec -it supabase-cloudflared ping kong
```

**Docker Compose 네트워크 구성 확인:**

```bash
docker network ls
docker network inspect supabase
```

### 4-2. 도메인 연결 확인

DNS 레코드가 올바르게 설정되었는지 확인:

```bash
dig supabase.neocompany.dev
```

결과에 `CNAME` 레코드와 `*.cfargotunnel.com`이 포함되어 있어야 합니다.

### 4-3. 유용한 명령어

**모든 서비스 중지:**
```bash
docker compose -f docker/docker-compose.yml down
```

**모든 서비스 시작:**
```bash
docker compose -f docker/docker-compose.yml up -d
```

**Cloudflared만 재시작:**
```bash
docker restart supabase-cloudflared
```

**로그 모니터링:**
```bash
docker logs -f supabase-cloudflared
```

---

## 5. 요약 및 참고사항

### 최종 디렉토리 구조
```
~/infra/supabase/
├── docker/
│   └── docker-compose.yml (cloudflared 서비스 추가)
└── cloudflared/
    ├── config.yml (cloudflared 설정)
    ├── cert.pem (인증서 파일)
    └── TUNNEL_ID.json (터널 인증 파일, 실제 UUID 파일명)
```

### 주요 구성 요소
- **Supabase**: Docker Compose로 관리되는 모든 컨테이너 구성
- **Kong API Gateway**: HTTP(8000) 및 HTTPS(8443) 포트 노출
- **Cloudflared**: Docker 컨테이너로 실행, Supabase의 내부 네트워크에 연결
- **도메인**: `supabase.neocompany.dev`로 외부에서 접근 가능
- **보안**: Cloudflare 관리 TLS 인증서로 자동 HTTPS 지원 