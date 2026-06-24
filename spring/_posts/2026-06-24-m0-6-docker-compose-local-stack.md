---
layout: post
title: "M0-6: Docker Compose 로컬 스택 (멀티스테이지 Dockerfile / Healthcheck)"
date: 2026-06-24 18:00:00 +0900
tags: [spring-boot, docker, docker-compose, dockerfile, healthcheck]
description: "Spring Boot 앱과 PostgreSQL·Redis 인프라를 docker compose up 한 번으로 띄우는 로컬 개발 스택을, 멀티스테이지 Dockerfile·헬스체크 기동 순서·환경변수 외부화·헬스 스모크 테스트로 구성한 과정."
---

> Spring Boot 애플리케이션과 인프라(PostgreSQL, Redis)를 `docker compose up` 한 번으로 띄우는 로컬 개발 스택을 구성한다. 멀티스테이지 Dockerfile, Compose 네트워크, 헬스체크 기반 기동 순서, 바인드 마운트, 환경변수 주입, 스모크 테스트까지 다룬다.

---

## 1. 목표와 산출물

| 항목 | 내용 |
|---|---|
| 목표 | `docker compose up` 한 번에 app + postgres + redis 기동 |
| 완료 기준(DoD) | 클린 환경에서 한 번에 전체 기동 + health 200(UP) |
| 검증 방법 | health 200 스모크 스크립트 PASS |

### 산출물 목록

| 파일 | 위치 | 역할 |
|---|---|---|
| `Dockerfile` | `backend/Dockerfile` | 앱 이미지 빌드(멀티스테이지) |
| `.dockerignore` | `backend/.dockerignore` | 빌드 컨텍스트 슬림화 |
| `build.gradle` | `backend/build.gradle` | plain jar 비활성(M0-6 추가분) |
| `compose.yml` | 루트 `compose.yml` | app/postgres/redis 오케스트레이션 |
| `application.yaml` | `backend/src/main/resources/application.yaml` | 환경변수 바인딩 |
| `.env` / `.env.example` | 루트 | 비밀값(단일 출처) / 템플릿 |
| `.gitignore` | 루트 `.gitignore` | 비밀값·데이터 보호 |
| `smoke.ps1` | `scripts/smoke.ps1` | health 200 스모크 테스트 |

---

## 2. 학습 포인트 요약

| 개념 | 핵심 |
|---|---|
| 멀티스테이지 빌드 | 빌드(JDK)와 실행(JRE) 단계를 분리해 최종 이미지 경량화 |
| Compose 네트워크 | 서비스명이 곧 DNS 호스트명(`postgres`, `redis`) |
| `depends_on` + healthcheck | 의존 서비스가 **healthy** 된 뒤 앱 기동(기동 순서 보장) |
| 바인드 마운트 | 데이터를 호스트 폴더에 보관 → 폴더 복사만으로 이관 |
| 환경변수 주입(`${}`) | 비밀값을 파일·코드에 박지 않고 주입 + 빈값 가드 |
| Actuator health 합산 | 앱 health 200(UP)이 DB·Redis 정상까지 전이적으로 증명 |

---

## 3. 전체 구조

```
ixikey/
├─ compose.yml                # 오케스트레이션 (루트)
├─ .env                       # 비밀값 단일 출처 (git 제외)
├─ .env.example               # 템플릿 (커밋)
├─ .gitignore                 # .env / .data 보호
├─ .data/                     # DB·Redis 실데이터 (git 제외, 첫 기동 시 생성)
│  ├─ postgres/
│  └─ redis/
├─ scripts/
│  └─ smoke.ps1               # health 스모크 테스트
└─ backend/
   ├─ Dockerfile
   ├─ .dockerignore
   ├─ build.gradle
   └─ src/main/resources/application.yaml
```

---

## 4. 소스와 해설

### 4-1. 멀티스테이지 Dockerfile

**위치: `backend/Dockerfile`**

```dockerfile
# syntax=docker/dockerfile:1

# ── 1단계: 빌드 (JDK 17로 bootJar 생성) ─────────────────────────────
# toolchain이 17이므로 빌드도 17로 맞춘다
FROM eclipse-temurin:17-jdk-jammy AS builder
WORKDIR /build

# (1) 의존성 캐시 최적화: 빌드 스크립트/래퍼 먼저 복사
#     → 소스만 바뀌면 의존성 재다운로드를 건너뛴다(레이어 캐시)
COPY gradlew settings.gradle build.gradle ./
COPY gradle ./gradle
RUN chmod +x gradlew && ./gradlew --no-daemon dependencies || true

# (2) 소스 복사 후 실행 가능한 jar 생성 (테스트는 CI에서 → 이미지 빌드는 -x test)
COPY src ./src
RUN ./gradlew --no-daemon clean bootJar -x test

# ── 2단계: 런타임 (가벼운 JRE만) ────────────────────────────────────
FROM eclipse-temurin:17-jre-jammy AS runtime
WORKDIR /app

# healthcheck용 curl 설치 (JRE 이미지엔 없음). 캐시 정리로 용량 최소화
RUN apt-get update && apt-get install -y --no-install-recommends curl \
    && rm -rf /var/lib/apt/lists/*

# root 대신 비특권 사용자로 실행 (보안 기본기)
RUN useradd -r -u 1001 appuser
USER appuser

# 빌드 단계 산출물 jar만 가져온다 (COPY 목적지와 ENTRYPOINT 경로 일치 필수)
COPY --from=builder /build/build/libs/*.jar /app/app.jar

EXPOSE 8090

# 컨테이너 자체 health 신호 (compose가 이 결과로 의존성 게이트를 연다)
HEALTHCHECK --interval=10s --timeout=3s --start-period=40s --retries=5 \
  CMD curl -fsS http://localhost:8090/actuator/health || exit 1

ENTRYPOINT ["java", "-jar", "/app/app.jar"]
```

| 단계 | 베이스 이미지 | 하는 일 |
|---|---|---|
| builder | `eclipse-temurin:17-jdk-jammy` | Gradle로 `bootJar` 생성 (JDK 필요) |
| runtime | `eclipse-temurin:17-jre-jammy` | jar만 복사해 실행 (JRE만, 경량) |

핵심은 **빌드 도구를 최종 이미지에서 제외**하는 것이다. JDK·Gradle·소스는 builder 단계에만 있고, runtime 단계에는 JRE와 jar만 남아 이미지가 가볍고 공격 표면이 줄어든다.

### 4-2. .dockerignore

**위치: `backend/.dockerignore`**

```gitignore
# 빌드 컨텍스트에서 제외 (이미지 빌드 속도·캐시·보안)
.git
.gradle
build/
bin/
out/
.idea
.vscode
.env            # 비밀값은 이미지에 넣지 않는다 (compose environment로 주입)
*.iml
```

`docker build`는 컨텍스트 폴더 전체를 데몬으로 전송한다. `build/`·`.gradle/` 같은 대용량·불필요 파일을 제외하면 전송·빌드가 빨라지고, `.env`를 제외해 **비밀값이 이미지 레이어에 박히는 사고**를 막는다.

### 4-3. plain jar 비활성 (build.gradle)

**위치: `backend/build.gradle`** — 아래 블록을 추가한다.

```groovy
// 실행 불가능한 plain jar 생성을 막아 bootJar 하나만 남긴다
// → Docker COPY 시 *.jar 가 한 개만 잡혀 경로 모호성 제거
tasks.named('jar') {
    enabled = false
}
```

Spring Boot Gradle 플러그인은 기본적으로 jar를 **두 개** 만든다.

| jar | 설명 | 실행 |
|---|---|---|
| `ixikey-0.0.1-SNAPSHOT.jar` | bootJar (의존성 포함 실행본) | 가능 |
| `ixikey-0.0.1-SNAPSHOT-plain.jar` | plain jar (클래스만) | 불가 |

`COPY --from=builder .../*.jar /app/app.jar`에서 `*.jar`가 두 파일을 매칭하면 Docker는 목적지를 **디렉터리로 취급**해 둘 다 그 안에 넣는다. 그러면 `app.jar`가 파일이 아니라 폴더가 되어 `java -jar`가 실패한다. plain jar를 비활성화해 jar를 하나로 만들면 해결된다.

### 4-4. compose.yml

**위치: 루트 `compose.yml`**

```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ixikey                 # DB명은 비밀 아님 → 그대로 명시
      POSTGRES_USER: ${DB_USERNAME}
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD 없음}   # 없으면 기동 중단
    ports:
      - "25432:5432"
    volumes:
      - ./.data/postgres:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USERNAME} -d ixikey"]
      interval: 5s
      timeout: 3s
      retries: 10

  redis:
    image: redis:7-alpine
    environment:
      REDIS_PASSWORD: ${REDIS_PASSWORD:?REDIS_PASSWORD 없음}
    # AOF 영속화 + 비밀번호 인증
    command: ["redis-server", "--appendonly", "yes", "--requirepass", "${REDIS_PASSWORD}"]
    ports:
      - "26379:6379"
    volumes:
      - ./.data/redis:/data
    healthcheck:
      # $$ 는 compose가 치환하지 않고 컨테이너 셸이 REDIS_PASSWORD를 읽게 한다
      test: ["CMD-SHELL", "redis-cli -a \"$$REDIS_PASSWORD\" ping | grep -q PONG"]
      interval: 5s
      timeout: 3s
      retries: 10

  app:
    build:
      context: ./backend
      dockerfile: Dockerfile
    environment:
      SPRING_PROFILES_ACTIVE: ${SPRING_PROFILES_ACTIVE:-local}
      SERVER_PORT: ${SERVER_PORT:-8090}
      # 컨테이너 네트워크 기준 호스트명으로 '명시' (localhost 아님)
      DB_URL: jdbc:postgresql://postgres:5432/ixikey
      DB_USERNAME: ${DB_USERNAME}
      DB_PASSWORD: ${DB_PASSWORD}
      REDIS_HOST: redis
      REDIS_PORT: 6379
      REDIS_PASSWORD: ${REDIS_PASSWORD}
    ports:
      - "8090:8090"
    depends_on:
      postgres:
        condition: service_healthy   # DB가 healthy 된 뒤 앱 시작
      redis:
        condition: service_healthy
```

#### 포트 매핑

| 서비스 | 호스트 포트 | 컨테이너 포트 | 비고 |
|---|---|---|---|
| app | 8090 | 8090 | 외부 접근 |
| postgres | 25432 | 5432 | 호스트 디버깅·IDE 직접 실행용 |
| redis | 26379 | 6379 | 호스트 디버깅용 |

#### 컨테이너 vs 호스트 접속 주소

| 접속 주체 | DB 주소 | Redis 주소 |
|---|---|---|
| 컨테이너 내부(app) | `postgres:5432` | `redis:6379` |
| 호스트(IDE 직접 실행) | `localhost:25432` | `localhost:26379` |

같은 Compose 네트워크 안에서는 **서비스명이 DNS 호스트명**이 된다. 따라서 app 컨테이너는 `localhost`가 아니라 `postgres`/`redis`로 접속해야 한다. 호스트에서 직접 실행하는 경우에만 노출된 포트(`localhost:25432`)를 사용한다.

#### 환경변수 치환 문법

| 문법 | 동작 |
|---|---|
| `${VAR:?메시지}` | 값이 없거나 비어 있으면 **에러 + 기동 중단** |
| `${VAR:-기본값}` | 값이 없거나 비어 있으면 **기본값 사용** |

비밀번호처럼 반드시 필요한 값은 `:?`로 "없으면 멈춤"을, 포트처럼 기본값 허용 값은 `:-`를 쓴다.

### 4-5. 환경변수 바인딩 (application.yaml)

**위치: `backend/src/main/resources/application.yaml`**

```yaml
spring:
  application:
    name: ixikey
  profiles:
    active: ${SPRING_PROFILES_ACTIVE:local}
  config:
    # backend/ 에서 실행 시 ../.env(=루트), 루트에서 실행 시 .env → 둘 다 루트 .env를 가리킴
    import: optional:file:.env[.properties],optional:file:../.env[.properties]

  datasource:
    url: ${DB_URL:jdbc:postgresql://localhost:25432/ixikey}
    username: ${DB_USERNAME:ixikey}
    password: ${DB_PASSWORD:ixikey}
    driver-class-name: org.postgresql.Driver

  data:
    redis:
      host: ${REDIS_HOST:localhost}
      port: ${REDIS_PORT:26379}
      password: ${REDIS_PASSWORD:}
      timeout: 2s

  session:
    timeout: 30m
    redis:
      namespace: ixikey:session
      flush-mode: on_save

  jpa:
    hibernate:
      ddl-auto: validate          # 스키마는 Flyway가 소유, Hibernate는 검증만
    open-in-view: false
    properties:
      hibernate:
        "[format_sql]": true

  flyway:
    enabled: true

server:
  port: ${SERVER_PORT:8090}
```

모든 값은 `${ENV:기본값}` 형태로 외부화되어 있다. 따라서 **동일한 jar가 환경변수만 바꿔 컨테이너(`postgres`/`redis`)에서도, 호스트(`localhost`)에서도** 동작한다. `config.import`에 `.env`와 `../.env`를 함께 두어, 실행 위치(루트/백엔드)에 관계없이 루트 `.env`를 읽도록 했다.

### 4-6. 비밀값 단일 출처 (.env / .env.example)

**위치: 루트 `.env`** (git 제외, 실제 값)

```dotenv
SPRING_PROFILES_ACTIVE=local
SERVER_PORT=8090

# PostgreSQL
DB_URL=jdbc:postgresql://localhost:25432/ixikey
DB_USERNAME=ixikey_admin
DB_PASSWORD=<실제 비밀번호>

# Redis
REDIS_HOST=localhost
REDIS_PORT=26379
REDIS_PASSWORD=<실제 비밀번호>
```

**위치: 루트 `.env.example`** (커밋, 자리값만)

```dotenv
SPRING_PROFILES_ACTIVE=local
SERVER_PORT=8090
DB_URL=jdbc:postgresql://localhost:25432/ixikey
DB_USERNAME=ixikey_admin
DB_PASSWORD=changeme
REDIS_HOST=localhost
REDIS_PORT=26379
REDIS_PASSWORD=changeme
```

`.env`를 **루트 한 곳**에만 두면 Compose(자동 로드)와 앱(`../.env`)이 같은 파일을 본다. 파일이 둘로 갈리면 값이 어긋나(드리프트) 인증 실패 같은 미묘한 버그가 생기므로, 단일 출처로 유지한다.

### 4-7. .gitignore (비밀값·데이터 보호)

**위치: 루트 `.gitignore`** — 아래 두 줄을 추가한다.

```gitignore
# 비밀값 / 실데이터
.env
.data/
```

`.env`(실제 비밀번호)와 `.data/`(DB·Redis 데이터)가 저장소에 올라가지 않도록 막는다. 적용 확인은 `git check-ignore .env .data/`로 한다(두 줄이 출력되면 정상).

### 4-8. 스모크 테스트

**위치: `scripts/smoke.ps1`** (UTF-8 with BOM으로 저장)

```powershell
# M0-6 스모크 테스트: 전체 스택 기동 후 health 200(UP) 확인
# 사용법:  .\scripts\smoke.ps1           (현재 상태로 기동·검증)
#          .\scripts\smoke.ps1 -Clean    (.data 삭제 후 '클린 환경'에서 검증)
param([switch]$Clean)

$ErrorActionPreference = "Stop"
Set-Location (Split-Path $PSScriptRoot -Parent)   # 루트로 이동

if ($Clean) {
    Write-Host "클린 모드: 컨테이너·볼륨·데이터 폴더 제거"
    docker compose down -v
    if (Test-Path .\.data) { Remove-Item .\.data -Recurse -Force }  # 바인드 마운트는 down -v로 안 지워짐
}

Write-Host "1) 전체 스택 빌드 + 기동..."
docker compose up -d --build

Write-Host "2) app health 대기 (최대 180초)..."
$ok = $false
for ($i = 0; $i -lt 36; $i++) {
    # curl.exe 로 직접 호출 (-s 조용히, 연결 실패/503이어도 예외 안 던짐)
    $resp = curl.exe -s http://127.0.0.1:8090/actuator/health 2>$null
    if ($LASTEXITCODE -eq 0 -and $resp -match '"status"\s*:\s*"UP"') {
        Write-Host "OK: health UP -> $resp" -ForegroundColor Green
        $ok = $true; break
    }
    Start-Sleep -Seconds 5
}

if (-not $ok) {
    Write-Host "FAIL: health 200 UP 아님. app 로그:" -ForegroundColor Red
    docker compose logs app --tail 50
    exit 1
}
Write-Host "SMOKE PASS" -ForegroundColor Green
```

Spring Boot Actuator의 `/actuator/health`는 DB·Redis 같은 구성요소 상태를 **합산**해 UP/DOWN을 반환한다. 따라서 앱 health가 200(UP)이면 postgres·redis까지 정상임이 전이적으로 증명되어, 별도 체크 없이 한 번의 호출로 전체 스택을 검증할 수 있다.

---

## 5. 의사결정 정리

| 결정 | 이유 | 대안 / 비고 |
|---|---|---|
| 멀티스테이지 빌드 | 최종 이미지에서 JDK·Gradle 제외 → 경량·보안 | 단일 스테이지(이미지 비대) |
| JRE 런타임 + 비특권 사용자 | 공격 표면 축소, root 실행 회피 | root 실행(비권장) |
| plain jar 비활성 | `*.jar` 단일화로 COPY 모호성 제거 | COPY에서 bootJar만 명시 |
| `depends_on` + healthcheck | DB·Redis가 준비된 뒤 앱 기동(연결 실패 방지) | 단순 `depends_on`(기동만 보장, 준비 미보장) |
| 바인드 마운트(`.data`) | 폴더 복사만으로 데이터 이관 = 서버 교체 | named volume(이관 어려움) |
| 환경변수 `${}` 주입 + `:?` | 비밀값 분리(12-factor) + 빈값 기동 차단 | 값 하드코딩(노출 위험) |
| `.env` 루트 단일 출처 | Compose·앱이 같은 파일 참조 → 드리프트 제거 | 파일 분리(값 불일치 위험) |
| health 스모크로 검증 | 한 번의 200(UP)으로 3개 서비스 동시 검증 | 서비스별 개별 점검 |

---

## 6. 트러블슈팅 기록

| 증상 | 원인 | 해결 |
|---|---|---|
| `Unable to access jarfile /app.jar` | COPY 목적지(`/app/app.jar`)와 ENTRYPOINT(`/app.jar`) 경로 불일치 | 두 경로를 `/app/app.jar`로 통일 |
| 위 에러 반복 | `*.jar`가 bootJar+plain jar 2개 매칭 → `app.jar`가 디렉터리화 | `build.gradle`에서 plain jar 비활성 |
| 앱이 DB·Redis 연결 실패 | 컨테이너에서 `localhost`로 접속 | 서비스명 `postgres`/`redis`로 접속 |
| `.env` 비밀값 노출 위험 | 루트 `.gitignore`에 `.env` 미등록 | 루트 `.gitignore`에 `.env`·`.data/` 추가 |
| 콘솔 한글 깨짐 | PowerShell 5.1이 BOM 없는 UTF-8을 시스템 코드페이지로 해석 | 스크립트를 UTF-8 with BOM으로 저장 |
| 스모크 FAIL(health 못 받음) | `Invoke-WebRequest`의 IPv6 `localhost`·프록시·503 예외 | `curl.exe`로 `127.0.0.1` 폴링 + 타임아웃 상향 |

---

## 7. 실행과 검증

### 전체 스택 기동 (도커 통합 검증)

```powershell
docker compose up --build
curl.exe http://127.0.0.1:8090/actuator/health   # {"status":"UP"}
```

### 인프라만 기동 (개발: 앱은 IDE에서 직접 실행)

```powershell
docker compose up -d postgres redis    # 인프라만
# 앱은 IDE 디버그 실행 또는: cd backend; .\gradlew bootRun
```

| 방식 | 인프라 | 앱 | 용도 |
|---|---|---|---|
| 전체 도커 | 컨테이너 | 컨테이너 | 배포 전 통합 검증 |
| 하이브리드 | 컨테이너 | IDE 직접 실행 | 평소 개발(핫리로드·디버깅) |

### 클린 환경 검증 (DoD)

```powershell
.\scripts\smoke.ps1 -Clean    # .data 삭제 후 완전히 깨끗한 상태에서 기동·검증
```

`-Clean`은 컨테이너·볼륨·`.data` 폴더를 모두 지우고 처음부터 기동하므로, "클린 환경에서 한 번에 전체 기동"을 그대로 증명한다.

---

## 8. 정리

| 핵심 | 한 줄 |
|---|---|
| 이미지 | 멀티스테이지로 빌드/실행 분리 → JRE+jar만 남겨 경량화 |
| 기동 순서 | healthcheck + `depends_on: service_healthy`로 DB·Redis 준비 후 앱 시작 |
| 네트워크 | Compose 내부에선 서비스명이 호스트명, 외부엔 포트 매핑 |
| 데이터 | `.data` 바인드 마운트 → 폴더 이관으로 서버 교체 |
| 설정 | 환경변수 외부화 + `.env` 단일 출처 + 빈값 가드 |
| 검증 | Actuator health 200(UP) 한 번으로 3개 서비스 동시 확인 |
