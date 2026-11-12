
# PuzzleLive
Go로 만든 웹 퍼즐 게임 라이브 서비스 서버. 재화 정합성부터 이벤트 운영, 장애 대응까지 직접 운영합니다.

클라이언트는 최소한으로 만들고, 서버(Go, MySQL, Redis)에 힘을 싣는 라이브 운영형 사이드프로젝트입니다.

## 로드맵

| 단계 | 주제 | 핵심 |
|---|---|---|
| 1 | 로그인, 스테이지 기록, 하트·재화 시스템 | 게임 서버 구조, DB 설계 |
| 2 | 이벤트 보상, 우편함, 중복 지급 방지 | 신규 콘텐츠, 정합성 |
| 3 | Redis 주간 랭킹 | Redis ZSET |
| 4 | 운영 어드민 (유저 조회, 재화 지급, 이벤트 설정, 변경 이력) | 웹 기반 툴 |
| 5 | 점수 조작 검증, 어뷰징 탐지 | 라이브 운영 |
| 6 | GitHub Actions 배포, Kubernetes 전환 | CI/CD, K8s |
| 7 | 지표·로그 대시보드, 알림, 부하 테스트 | 모니터링, 처리량 수치 |

## 설계 결정

| 결정 | 선택 | 이유 |
|---|---|---|
| 게임 로직 위치 | 서버 권위(authoritative) | 클라이언트 점수를 신뢰하지 않음 (5단계 전제) |
| 퍼즐 종류 | 클래식 3-매치 | 규칙이 단순하고 replay 검증이 쉬움 |
| 점수 검증 | 시드 + 이동 로그를 서버가 Go로 재현(replay) | 스테이지 시작 시 서버가 seed 발급, 클리어 시 이동 목록 제출, 서버가 재계산해 불일치 시 거절 및 어뷰징 로그 |
| 클라이언트 엔진 | TypeScript로 동일 규칙 구현 + 공유 테스트 벡터(JSON) | Go→WASM은 과함. 양쪽이 같은 입력에 같은 결과를 내는지 테스트로 보장 |
| 하트 회복 | Lazy 계산 (마지막 갱신 시각 기준) | 배치 잡 불필요, 정합성 단순 |
| 재화 | 원장(ledger) 테이블 + 잔액 컬럼 | 변경 이력, 중복 지급 방지, 감사 추적 |
| 중복 지급 방지 | `idempotency_key` UNIQUE + 트랜잭션 내 `SELECT ... FOR UPDATE` | 이벤트 보상·우편 수령 공통 적용 |
| 로그인 | 게스트(device ID) + 이메일 가입 | 진입 장벽 최소화 |
| 서비스 구조 | 모듈러 모놀리스 | 1인 프로젝트에 마이크로서비스는 과함. 모듈 경계는 명확히 유지 |

## 기술 스택

### 백엔드 (Go)

| 영역 | 선택 |
|---|---|
| Go | 1.23+ |
| HTTP 라우터 | chi (`net/http` 호환) |
| DB 접근 | sqlc + `database/sql` + go-sql-driver/mysql (ORM 미사용, SQL 직접 작성) |
| 마이그레이션 | goose |
| Redis 클라이언트 | go-redis v9 |
| 인증 | Access JWT(15분) + Refresh 토큰(Redis 저장, 로테이션), 비밀번호 argon2id |
| 설정 | 환경변수 + envconfig |
| 로깅 | 표준 `log/slog` (JSON) |
| 메트릭 | prometheus/client_golang |
| 검증 | go-playground/validator |
| 스케줄러 | robfig/cron + Redis 분산 락 (K8s 전환 후 CronJob) |
| 테스트 | testing + testify + testcontainers-go (실제 MySQL/Redis) |
| API 문서 | OpenAPI |

모듈 구성: `internal/{auth,stage,wallet,mail,event,ranking,admin,antiabuse}`

### 데이터

| 영역 | 선택 |
|---|---|
| RDB | MySQL 8 (InnoDB, utf8mb4) |
| 주간 랭킹 | Redis ZSET, 키 `rank:{ISO주차}`, `ZADD GT`, 마감 시 MySQL 스냅샷 + 보상 우편 발송 |
| 그 외 Redis | 리프레시 토큰, 레이트 리밋, 분산 락, 스테이지 세션(seed) |
| 주요 테이블 | users, wallets, wallet_ledger, hearts, stage_records, stage_sessions, mails, events, event_claims, admin_users, admin_audit_logs, abuse_flags, weekly_rank_snapshots |

### 프론트엔드

| 영역 | 선택 |
|---|---|
| 프레임워크 | React + TypeScript + Vite |
| 퍼즐 렌더링 | DOM/CSS grid (Canvas 미사용) |
| 상태/통신 | TanStack Query + Zustand |
| UI | 어드민은 MUI, 게임 화면은 직접 스타일링 |
| 구성 | 모노레포 내 `web`(게임) / `admin`(어드민) 두 Vite 앱 |
| 어드민 인증 | 별도 `admin_users` + RBAC(viewer/operator/admin), 모든 변경은 `admin_audit_logs`에 기록 |

### 인프라 · 배포 (무료 우선)

Oracle Cloud Always Free는 가입 오류로 제외했습니다.

| 구성요소 | 선택 | 비고 |
|---|---|---|
| Go 서버 | Render 무료 웹 서비스 (대안: Koyeb) | 유휴 시 슬립, 시연 전 깨워 둠 |
| MySQL | Aiven 무료 MySQL (대안: TiDB Cloud Serverless) | |
| Redis | Upstash 무료 | 월 명령 수 제한, 랭킹 조회는 짧은 캐시 |
| 프론트 | Cloudflare Pages | |
| CI | GitHub Actions + GHCR | |
| 모니터링 | Grafana Cloud 무료 (Prometheus, Loki, 알림) | 알림은 Telegram 웹훅 |
| 로컬 개발 | docker-compose (Go + MySQL + Redis) | 배포는 환경변수만 교체 |

위 무료 티어 조건은 가입 전에 공식 문서로 재확인합니다.

### Kubernetes (6단계)

- 실서비스는 무료 PaaS 조합으로 운영합니다.
- K8s는 k3d(또는 kind) 로컬 클러스터에서 Kustomize 매니페스트, HPA, probe, CronJob(주간 랭킹 마감)을 구성하고, GitHub Actions에서 kind로 매니페스트를 적용해 헬스체크까지 검증합니다.
- 상시 운영이 필요하면 6단계 시점에 월 약 $5 VPS 한 대에 k3s를 한 달만 올려 시연 자료를 확보합니다.

### 부하 테스트 (7단계)

무료 티어에 부하를 걸지 않고, docker-compose 로컬 환경에 k6를 돌려 RPS, p95, 에러율을 측정합니다. 측정 환경 사양을 함께 기록합니다.
