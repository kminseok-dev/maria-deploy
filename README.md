# maria-deploy

> 신한 DS SW Academy 팀 프로젝트 **MARIA**(RIA 국내시장복귀계좌 백오피스)의 서버 배포 설정 · 2026.10 ~ 2026.11
> 원본 레포: [returns-7/maria-deploy](https://github.com/returns-7/maria-deploy) · 이 레포의 커밋은 전부 내가 작성했다.

앱 소스는 들어 있지 않다. 각 서비스 레포가 만든 이미지를 받아 앱 서버·DB 서버에 띄우는 compose, nginx, 배포 스크립트만 둔다.

## 구성

```
                 ┌──────────────── 앱 서버 (app/compose.yml) ────────────────┐
 Client ──HTTPS──▶ nginx ──edge──▶ maria (RIA 관리자·고객 API)                 │
                 │                   │ backend                               │
                 │                   ├─▶ mydata             ─┐               │
                 │                   ├─▶ return-securities  ─┤ API로만 연동  │
                 │                   ├─▶ risk (Python)       │               │
                 │                   └─▶ redis               │               │
                 └───────────────────────────────────────────┼───────────────┘
                                                             ▼
                 ┌──────────────── DB 서버 (db/compose.yml) ─────────────────┐
                 │  MariaDB 11.8 (ria_admin, securities) · PostgreSQL 15 (mydata) │
                 └────────────────────────────────────────────────────────────┘
```

| 서비스 | 이미지 |
|---|---|
| maria / mydata / return-securities / risk | `ghcr.io/returns-7/<서비스>:<커밋 SHA>` (태그는 `tags.env`에 기록) |
| nginx | `nginx:1.30-alpine` |
| redis | `redis:7-alpine` |

## 배포 흐름

```
서비스 레포 push → GitHub Actions: 테스트 → 이미지 빌드 → ghcr.io push
  → SSH로 앱 서버의 app/deploy.sh <서비스> <커밋 SHA> 호출
```

`app/deploy.sh`가 하는 일:

| 단계 | 내용 |
|---|---|
| 배치 시간 차단 | 00:00 ~ 02:30 KST(정산·동기화·세액 배치)에는 배포하지 않고 종료 |
| 동시 배포 방지 | `flock /tmp/maria-deploy.lock`으로 여러 레포의 배포를 서버에서 한 번에 하나만 실행 |
| 교체 | 그 서비스 하나만 `pull` → `up -d --no-deps` |
| risk 배치 경쟁 방지 | risk는 배치가 쓰는 `/data/prices.batch.lock`을 교체가 끝날 때까지 직접 잡는다. 못 잡으면 배치 실행 중이므로 아무것도 바꾸지 않고 종료 |
| 헬스체크 | 컨테이너 health가 `healthy`가 될 때까지 최대 3분 대기 |
| 성공 | `tags.env`에 새 태그 기록 |
| 실패 | 로그 50줄 출력 후 `tags.env`의 이전 태그로 되돌림 |

`tags.env`는 헬스체크를 통과한 태그만 기록하므로, 실패한 배포가 다음 배포의 기준 태그가 되지 않는다.

## 보안 설정

| 항목 | 내용 | 위치 |
|---|---|---|
| 내부 경로 차단 | `/actuator/`, `/api/internal/`(서비스 간 내부 API)은 외부 요청에 404 | `app/nginx/templates/default.conf.template` |
| 로그인 rate limit | 관리자 로그인 IP당 10회/분, burst 5, 초과 시 429 | 같은 파일 |
| DB 계정 최소 권한 | 앱마다 별도 계정. `securities_app`은 `securities.*` SELECT + `domestic_trade` INSERT만 | `db/init/mariadb/90_app_users.sh` |
| 비밀값 | 저장소에 두지 않음. 서버의 `.env`에서 주입 | `app/compose.yml`, `db/compose.yml` |
| DB 포트 | `${DB_BIND_IP}`에만 바인딩 | `db/compose.yml` |

## 장애 격리

- maria의 `depends_on`에 risk를 넣지 않았다. risk가 죽어도 maria는 기동된다.
- 각 앱은 `/actuator/health`(risk는 `/health`) 기반 healthcheck를 가지고, nginx·maria는 의존 서비스가 healthy일 때 기동한다.

## 시드 되돌리기

시연 기준 상태를 `db/backup/`에 저장해 두고, 시연·연습으로 데이터가 바뀌면 언제든 그 상태로 되돌린다. 백업 파일은 서버에만 둔다(gitignore).

- `db/seed-snapshot.sh`: 지금 DB(`ria_admin`·`securities`·`mydata`)와 업무시각을 저장한다. 덮어쓴다.
- `db/seed-restore.sh`: 저장한 상태로 되돌린다. 업무시계는 저장 당시 업무시각부터 다시 흐른다.

앱이 DB를 쓰는 중에 테이블을 다시 만들지 않도록 앱을 먼저 내린다.

```bash
# 1) 앱 서버 app 폴더: 앱 내리기
docker compose --env-file .env --env-file tags.env stop maria return-securities mydata risk
# 2) DB 서버 db 폴더: 되돌리기
./seed-restore.sh
# 3) 앱 서버 app 폴더: 캐시 비우고 앱 올리기
docker compose --env-file .env --env-file tags.env exec -T redis redis-cli FLUSHALL
docker compose --env-file .env --env-file tags.env start maria return-securities mydata risk
```

환경별 덮어쓰기가 필요하면 app 폴더 명령에 `-f compose.yml -f compose.academy.yml`처럼 override 파일을 붙인다.

## 실행

```bash
# DB 서버
cd db && docker compose --env-file .env up -d
# 앱 서버
cd app && docker compose --env-file .env --env-file tags.env up -d
```
