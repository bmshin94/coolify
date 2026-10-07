# Coolify 전수조사 & 활용 전략 리포트 (한국어)

> 작성일: 2026-10-07
> 작성: Claude Code (Karina persona)
> 대상 저장소: **https://github.com/bmshin94/coolify**
> 업스트림 원본: **https://github.com/coollabsio/coolify**
> 분석 브랜치: `claude/github-project-analysis-wth9i6`
> 분석 기준 버전: `4.3.22` (nightly `4.4-rc.1`)

---

## 목차

1. [프로젝트 정체 — 이게 뭐하는 건가](#1-프로젝트-정체--이게-뭐하는-건가)
2. [규모와 기술 스택](#2-규모와-기술-스택)
3. [폴더 구조 전수조사](#3-폴더-구조-전수조사)
4. [내장 MCP 서버 심층 분석](#4-내장-mcp-서버-심층-분석)
5. [AI 에이전트 지원 체계](#5-ai-에이전트-지원-체계)
6. [동작 원리 (배포 파이프라인)](#6-동작-원리-배포-파이프라인)
7. [설계상의 함정 2가지](#7-설계상의-함정-2가지)
8. [언제 쓰고, 언제 안 쓰나](#8-언제-쓰고-언제-안-쓰나)
9. [설치 및 사용법](#9-설치-및-사용법)
10. [플러그인 / 스킬 / MCP — 정체 규명](#10-플러그인--스킬--mcp--정체-규명)
11. [API 토큰](#11-api-토큰)
12. [AI 에이전트 구축에 주는 도움](#12-ai-에이전트-구축에-주는-도움)
13. [React / PHP 로 만들 수 있나](#13-react--php-로-만들-수-있나)
14. [유튜브 강의 제작 가능성](#14-유튜브-강의-제작-가능성)
15. [수익화 아이디어 13선](#15-수익화-아이디어-13선)
16. [추천 로드맵](#16-추천-로드맵)
17. [법적 체크리스트](#17-법적-체크리스트)

---

## 1. 프로젝트 정체 — 이게 뭐하는 건가

**Coolify = 오픈소스 셀프호스팅 PaaS.**
Heroku / Netlify / Vercel 의 자가 호스팅 대안이다. SSH 접속만 가능하면 VPS, 베어메탈, 라즈베리파이, 집에 남는 노트북까지 모두 배포 대상으로 관리할 수 있다.

핵심 철학: **"설정은 내 서버에 남는다."** Coolify 를 삭제해도 배포된 Docker 컨테이너는 계속 동작하며 수동으로 관리 가능하다. (vendor lock-in 없음)

주요 기능:

- 모든 애플리케이션 배포 — GitHub / GitLab / Bitbucket / Gitea + Nixpacks, Railpack, Dockerfile, Docker Compose, 프리빌트 이미지
- 데이터베이스 & 서비스 — 관리형 DB 9종 + 원클릭 서비스 342개, 영속 스토리지, 자격증명 자동 생성
- 배포 자동화 — Git push 자동 배포, PR 프리뷰 환경, 배포 웹훅, 이미지 롤백
- 네트워킹 — 커스텀 도메인, 자동 HTTPS, 리버스 프록시(Traefik/Caddy), 헬스체크
- 인프라 운영 — 멀티 서버, 배포/런타임 로그, 컨테이너 터미널, 상태 모니터링
- 보호 — DB/볼륨 백업(S3), 스케줄 작업, 환경변수/시크릿, 알림
- 통합 — 대시보드, REST API, CLI, **MCP**, 팀 기반 접근 제어

---

## 2. 규모와 기술 스택

### 규모

| 항목 | 수치 |
|---|---|
| `app/` PHP 파일 | 785 |
| Blade 템플릿 | 456 |
| Livewire 컴포넌트 | 232 |
| REST API 엔드포인트 | 297 |
| 원클릭 서비스 템플릿 | 342 (Compose YAML 371) |
| MCP 툴 | 45 (+ 리소스 2, 프롬프트 2) |
| OpenAPI 스펙 | `openapi.json` 1 MB |
| CHANGELOG | 322 KB |

엔터프라이즈급 대형 모놀리식 프로젝트.

### 기술 스택 (실측 `composer.json` / `package.json`)

| 레이어 | 기술 |
|---|---|
| Backend | PHP 8.4+, Laravel 12.65, Laravel Horizon 5.48 |
| UI | Livewire 3.8, Alpine.js, Blade, Tailwind CSS v4.3, Vite 8.2 |
| 에디터/터미널 | Monaco Editor, XTerm.js 6 |
| 실시간 | Soketi (WebSocket) |
| DB / 캐시 | PostgreSQL 15, Redis 7 |
| SSH | phpseclib 3.0 |
| MCP | `laravel/mcp` ^0.6.7 |
| 인프라 | Docker Compose, Traefik 2.11~3.7 / Caddy, S6 Overlay, Nginx |
| 패키지 | lorisleiva/laravel-actions, spatie/laravel-data, spatie/laravel-activitylog, spatie/laravel-schemaless-attributes |
| 테스트 | Pest 4, PHPUnit 12, Laravel Dusk 8, Playwright |
| 품질 | Pint, PHPStan 2, Rector 2 |
| 인증 | Laravel Fortify, Sanctum, Socialite |
| 관측 | Laravel Nightwatch, Pail |

> 주의: `TECH_STACK.md` 는 "Laravel 11 / PHP 8.4" 로 적혀 있으나 실제 코드는 Laravel 12 / PHP 8.4~8.5. 문서 드리프트.

---

## 3. 폴더 구조 전수조사

### `app/` — 애플리케이션 코어

| 폴더 | 파일수 | 역할 |
|---|---:|---|
| `Livewire/` | 232 | UI 전체. 전통 Controller 없이 Livewire 가 담당. `Server/`, `Project/`, `Settings/`, `Security/`, `Terminal/`, `Subscription/`, `Notifications/`, `SharedVariables/`, `Admin/`, `Boarding/`, `Dashboard/`, `Destination/`, `Source/`, `Storage/`, `Tags/`, `Team/`, `Profile/` |
| `Actions/` | 76 | 도메인 단위 작업. `Application/`, `Database/`, `Destination/`, `Development/`, `Docker/`, `Fortify/`, `Proxy/`, `Server/`, `Service/`, `Shared/`, `Stripe/`, `Team/`, `User/`, `CoolifyTask/`. `lorisleiva/laravel-actions` 의 `AsAction` — 하나의 클래스를 객체·잡·컨트롤러·커맨드로 재사용 |
| `Http/` | 70 | API 컨트롤러 + 미들웨어 (`ApiAbility`, `ApiSensitiveData`, `CheckForcePasswordReset`, `DecideWhatToDoWithUser`, `EnsureMcpEnabled`, `EnsureTeamMcpEnabled`) |
| `Jobs/` | 56 | 배포 핵심. `ApplicationDeploymentJob`, 백업, Docker 청소, 서버 관리, 프록시 설정. Redis 큐 + Horizon |
| `Models/` | 56 | `BaseModel`(자동 CUID2 UUID) 상속. `Server`, `Application`, `Service`, `Project`, `Environment`, `Team` + 독립 DB 9종 (`StandalonePostgresql`, `StandaloneMysql`, `StandaloneMariadb`, `StandaloneMongodb`, `StandaloneRedis`, `StandaloneKeydb`, `StandaloneDragonfly`, `StandaloneClickhouse`) |
| `Mcp/` | 54 | Coolify 내장 MCP 서버 (4장 참조) |
| `Notifications/` | 45 | Email, Discord, Slack, Telegram, Pushover, Webhook |
| `Console/` | 36 | Artisan 커맨드 + 스케줄러 |
| `Policies/` | 27 | 약 15개 모델↔정책 매핑. 팀 멀티테넌시 |
| `Services/` | 23 | `ConfigurationGenerator`, `ConfigurationRepository`, `ContainerStatusAggregator`, `DockerImageParser`, `CoolifyUpgradeStatus`, `ProxyDashboardCacheService`, `RestartCountTracker`, `SchedulerLogParser`, `ChangelogService`, `HetznerService`, `DigitalOceanService`, `VultrService`, `ServerTransfer/`, `DeploymentConfiguration/` |
| `Events/` | 22 | `ApplicationStatusChanged`, `ServiceStatusChanged`, `DatabaseStatusChanged`, `ProxyStatusChanged` 등 브로드캐스트 |
| `Traits/` | 17 | `HasConfiguration`, `HasMetrics`, `HasSafeStringAttribute`, `ClearsGlobalSearchCache` |
| `View/` | 14 | View 컴포넌트 |
| `Rules/` | 12 | `ValidGitRepositoryUrl`, `ValidServerIp`, `ValidHostname`, `DockerImageFormat` |
| `Enums/` | 11 | `Role`(MEMBER 1 < ADMIN 2 < OWNER 3), `ProcessStatus`, `BuildPackTypes`, `ProxyTypes`, `ContainerStatusTypes`, `ApplicationDeploymentStatus`, `NewDatabaseTypes`, `NewResourceTypes`, `RedirectTypes`, `StaticImageTypes`, `ActivityTypes` |
| `Support/` | 10 | 지원 클래스 |
| `Providers/` | 9 | 서비스 프로바이더 (Laravel 10 구조) |
| `Exceptions/` | 6 | `app/Exceptions/Handler.php` 포함 |
| `Helpers/` | 3 | `bootstrap/helpers/` 의 글로벌 헬퍼와 연계 (`shared.php`, `constants.php`, `versions.php`, `subscriptions.php`, `domains.php`, `docker.php`, `services.php`, `github.php`, `proxy.php`, `notifications.php`) |
| `Listeners/`, `Casts/`, `Contracts/`, `Data/`, `Repositories/` | 각 1~2 | 보조 |

### 그 외 주요 디렉터리

| 경로 | 내용 |
|---|---|
| `routes/` | `api.php` (42 KB, 297 엔드포인트), `web.php` (34 KB), `webhooks.php`, `ai.php` (MCP), `channels.php`, `console.php` |
| `templates/` | `service-templates-latest.json` (342개 앱), `compose/*.yaml` (371개) — n8n, Supabase, Appwrite, Authentik, AnythingLLM, Appsmith, Budibase, Argilla, Browserless, Bookstack 등 |
| `docker/` | `coolify-helper` (SSH 실행 컨테이너), `coolify-realtime` (Soketi), `production`, `development`, `lima`, `testing-host` |
| `scripts/` | `install.sh` (원클릭 설치), `upgrade.sh`, `cloud_upgrade.sh`, `upgrade-postgres.sh`, `dev-instances` (로컬 Coolify 2개 동시 기동), `dev-helper`, `railpack-smoke.sh`, `seed-transfer-demo.php` |
| `docs/v5/ui/` | **차기 V5 UI 프로토타입 아카이브** — React + TypeScript + shadcn/ui + 캔버스형 인프라 다이어그램 (`canvas-toolbar.tsx`, `connection-lines.tsx`, `application-card.tsx`, `application-inspector-sheet.tsx`). `.txt` 확장자로 빌드 제외. README: *"Do not import or serve these files. V5 will use a different implementation."* |
| `docs/superpowers/specs/` | 설계 스펙 문서 |
| `tests/` | Pest 4. `tests/v4/Browser/` 가 신규 브라우저 테스트 위치 (Playwright + in-process amphp HTTP 서버) |
| `lang/`, `svgs/`, `other/logos/` | 다국어, 아이콘, 서비스 로고 |
| `openapi.json` / `openapi.yaml` | OpenAPI 3.0 자동 생성 스펙 |

### Docker Compose 파일 구성

| 파일 | 용도 |
|---|---|
| `docker-compose.yml` | 베이스 |
| `docker-compose.dev.yml` | 개발 (app, postgres, redis, soketi, vite, testing-host, mailpit, minio) |
| `docker-compose.dev-multi.yml` | 인스턴스 2개 동시 (a:8000 / b:8001) |
| `docker-compose.prod.yml` | 프로덕션 |
| `docker-compose.windows.yml` | Windows Docker Desktop |
| `docker-compose-maxio.dev.yml` | 결제 연동 개발 |

---

## 4. 내장 MCP 서버 심층 분석

Coolify 는 **자기 자신을 AI 에이전트에게 노출하는 MCP 서버를 내장**하고 있다.

### 엔드포인트

```php
// routes/ai.php
Mcp::web('/mcp', CoolifyServer::class)
    ->middleware(['mcp.enabled', 'auth:sanctum', 'api.token.team', 'mcp.team.enabled']);
```

2단 활성화 스위치: **인스턴스 전역**(`mcp.enabled`) + **팀별**(`mcp.team.enabled`).

### 서버 정의 — `app/Mcp/Servers/CoolifyServer.php`

- `name = 'Coolify'`, `version = '0.2.0'`
- `maxPaginationLength = 100`, `defaultPaginationLength = 100` (패키지 기본 15 → 전체 툴 1페이지 노출)

### 툴 45개

| 그룹 | 툴 |
|---|---|
| 진입/탐색 | `coolify_help`, `get_infrastructure_overview`, `search_resources`, `list_unhealthy_resources` |
| 팀 | `get_current_team`, `list_team_members` |
| 서버 | `list_servers`, `get_server`, `get_server_domains`, `get_server_resources` |
| 목적지 | `list_destinations`, `get_destination` |
| 프로젝트 | `list_projects`, `get_project`, `get_environment`, `list_resources` |
| 애플리케이션 | `list_applications`, `get_application`, `list_application_previews` |
| 데이터베이스 | `list_databases`, `get_database`, `list_database_backups`, `list_backup_executions` |
| 서비스 | `list_services`, `get_service`, `list_service_applications`, `get_service_application`, `list_service_databases`, `get_service_database` |
| 배포/로그 | `list_deployments`, `get_deployment`, `get_logs` |
| 환경변수 | `list_env_keys`, `list_shared_env_keys` (**이름만, 값은 절대 반환 안 함**) |
| 스토리지/태그 | `list_storages`, `list_resource_tags`, `list_tags` |
| 스케줄 | `list_scheduled_tasks`, `list_scheduled_task_executions` |
| GitHub | `list_github_apps`, `list_github_repositories`, `list_github_branches` |
| **라이프사이클 (deploy 권한 필요)** | `control` (start/stop/restart, stop 은 `confirm=true` 필수), `deploy`, `cancel_deployment` |

- 리소스 2개: `coolify://overview`, `coolify://application/{uuid}`
- 프롬프트 2개: `troubleshoot_application`, `explain_failed_deploy`
- 공용 Concern 4개: `BuildsResponse`, `McpStatusFilters`, `ResolvesResource`, `ResolvesTeam`

### 응답 규약

```
{ data, _actions?, _pagination? }
```

> "Env values, configuration snapshots, and full deploy logs are **never** returned. Optional deploy log summaries are best-effort redacted only."

### 배울 만한 설계 패턴 8가지

1. **툴 호출 순서를 instructions 로 가르친다** — "Start here (prefer these before deep get_*)" → AI 의 무작정 전체 조회 방지, 토큰 절약
2. **응답에 `next_tools` 를 넣어 AI 를 유도** — `['tool' => 'get_deployment', 'args' => [...], 'hint' => 'Poll status / failure summary']`
3. **싼 툴 → 비싼 툴 계층** — `sample_only=true` 로 맛보기 후 상세 조회, 컨텍스트 폭발 방지
4. **노출 금지 라인 명시** — 환경변수 값 / 설정 스냅샷 / 전체 배포 로그
5. **파괴적 작업에 확인 요구** — `stop` 은 `confirm=true`, 라이프사이클은 `deploy` ability 분리
6. **모든 쿼리를 테넌트로 스코프** — `Application::ownedByCurrentTeamAPI($teamId)->where('uuid', $uuid)`
7. **감사 로그** — `auditLog('mcp.deploy', [...])`
8. **실패 시 루프 방지** — "on failure use reason + next_tools (do not loop)"

> 정독 추천 파일: `app/Mcp/Tools/CoolifyHelp.php`, `GetLogs.php`, `ListUnhealthyResources.php`, `Deploy.php`

### Claude 연결 설정

```jsonc
{
  "mcpServers": {
    "coolify": {
      "type": "http",
      "url": "https://coolify.example.com/mcp",
      "headers": { "Authorization": "Bearer <API_TOKEN>" }
    }
  }
}
```

---

## 5. AI 에이전트 지원 체계

이 저장소는 AI 코딩 에이전트 지원이 이례적으로 충실하다.

| 파일/폴더 | 용도 |
|---|---|
| `CLAUDE.md` (31 KB) | Claude Code 용 프로젝트 가이드 (+ Karina 페르소나) |
| `AGENTS.md` (28 KB) | 범용 에이전트 가이드 |
| `GEMINI.md` | Gemini 용 |
| `DESIGN.md` (36 KB) | UI/UX 디자인 단일 진실 공급원 |
| `.claude/skills/` | 스킬 11개 |
| `.agents/skills/` | 동일 스킬 (범용) + `shadcn` 스킬 |
| `.cursor/skills/` | Cursor 용 |
| `.codex/config.toml` | OpenAI Codex 용 |
| `.mcp.json` | `laravel-boost` MCP 클라이언트 설정 |
| `opencode.json` | OpenCode 용 |
| `boost.json` | Laravel Boost — 에이전트 4종(`cursor`, `claude_code`, `codex`, `opencode`), 스킬 11개 등록 |
| `skills-lock.json` | 외부 스킬 해시 고정 (`shadcn/ui`) |
| `.ai/lessons.md` | **AI 가 과거에 실수한 교훈 누적 기록** |
| `jean.json` | Jean 샌드박스 실행 설정 |

### 스킬 11개

`fortify-development`, `laravel-best-practices`(하위 규칙 20여 개), `configuring-horizon`, `mcp-development`, `configure-nightwatch`, `socialite-development`, `livewire-development`, `pest-testing`, `tailwindcss-development`, `laravel-actions`, `debugging-output-and-previewing-html-using-ray`

### `.ai/lessons.md` 교훈 요약

- 코드 수정 전에 베이스라인에서 회귀를 먼저 재현하라
- 유닛 테스트 통과 / 빌드 성공 / 프로세스 정상을 UI 버그 수정의 증거로 쓰지 말라
- 요청 범위를 넘는 과금 제한, 실시간 정합, 폴백을 추가하지 말라
- 동적 리스트는 불변 식별자로 키를 잡아라 (count / index 금지)
- 부모와 자식을 같은 작업에서 동시에 refresh 하지 말라
- 모달은 부모 페이지가 소유하고 파괴적 액션은 푸터 왼쪽에 둬라
- 표시되는 기본값은 상속 상태를 유지하고, 사용자가 바꿀 때만 override 저장
- 컨테이너 이미지 변경은 Compose 와 모든 Dockerfile 빌드 스테이지를 끝까지 추적하라. `latest` 금지, 안정 릴리스 태그 고정

---

## 6. 동작 원리 (배포 파이프라인)

```
git push origin main
      ↓
GitHub Webhook → POST /webhooks/...        (routes/webhooks.php)
      ↓
ApplicationDeploymentJob 을 Redis 큐에 투입
      ↓
Horizon 워커가 잡 실행
      ↓
phpseclib 으로 대상 서버 SSH 접속
      ↓
docker build → docker run → 헬스체크
      ↓
Traefik 이 라벨 읽고 라우팅 + Let's Encrypt HTTPS 자동 발급
      ↓
Soketi WebSocket 으로 브라우저에 로그 실시간 스트리밍
      ↓
Discord / Slack / Telegram / Email 알림
```

**핵심**: Coolify 는 대상 서버에 **에이전트를 설치하지 않는다.** SSH 로 접속해 `docker` 명령을 실행할 뿐이다. 그래서 SSH 가능한 모든 기계를 관리할 수 있다.

---

## 7. 설계상의 함정 2가지

### 7.1 `id = 0` 센티널

Coolify 는 "인스턴스 자신"을 PK `0` 으로 저장한다. 이는 자동증가 id 가 아니라 **센티널**이다.

| 레코드 | 조회 | 의미 |
|---|---|---|
| 루트 팀 | `Team::find(0)`, `team_id === 0` | 인스턴스/루트 팀. 클라우드 과금·스킵 체크가 면제 |
| localhost 서버 | `Server::find(0)` | Coolify 가 실행 중인 그 기계. 업그레이드/인스턴스 백업 대상 |
| 인스턴스 설정 | `InstanceSettings` `id = 0` | 싱글톤 설정. 테스트는 `InstanceSettings::create(['id' => 0])` 시딩 필수 |
| 인스턴스 Postgres | `StandalonePostgresql` `id = 0`, name `coolify-db` | Coolify 자체 DB |
| 로컬 Docker 목적지 | `StandaloneDocker` `id = 0` | localhost 서버의 목적지 |

**주의사항**
- 새 row 나 비-인스턴스 row 에 `id = 0` 을 절대 할당하지 말 것
- `ScheduledDatabaseBackup` / `ScheduledTask` 는 일반 스케줄. 레거시 설치의 `coolify-db` 백업은 `coolify-db` 관계/uuid 로 해석할 것 (`find(0)` 금지)
- PHP 지뢰: `empty(0) === true`. 키셋 페이지네이션 `where('id', '>', $cursor)` 가 0부터 시작하면 해당 row 를 건너뜀 → 첫 페이지에 `id = 0` 포함, 가능하면 `chunkById()` 사용

### 7.2 Livewire 스냅샷 레이스

동적 리스트에서 추가/삭제/변환 후 컨트롤이 먹통이 되고 콘솔에 `Snapshot missing on Livewire component` 가 뜨는 문제.

- 루프 내 모든 컴포넌트에 **레코드 ID / UUID / 파일명 등 불변 식별자**로 key 부여. `$loop->index`, 컬렉션 count, 재인덱싱된 배열 위치 금지
- edit/delete 액션에도 같은 불변 식별자 전달. `removeItem($index)` 류는 첫 삭제 후 엉뚱한 행을 가리킴 → 서버 측에서 ID/UUID 로 현재 행을 해석
- 같은 작업에서 부모 리스트와 (부모가 삽입/삭제/숨길 수 있는) 자식에게 하나의 refresh 이벤트를 동시에 브로드캐스트하지 말 것
- refresh 책임 분리. `$this->dispatch('event')->to(Component::class)` 로 타겟팅
- 참조 구현: `Project\Service\Storage` 가 `storageCountsChanged` 처리, `Project\Shared\Storages\All` 이 `refreshVolumeList` 처리

---

## 8. 언제 쓰고, 언제 안 쓰나

### 적합

1. Vercel / Heroku 요금 부담 — VPS 월 $5~10 에 앱 무제한
2. Git push 자동 배포 + PR 프리뷰 URL 필요
3. DB 를 클릭으로 띄우고 S3 자동 백업
4. n8n, Supabase, Grafana, AnythingLLM 등 342개 원클릭 설치
5. HTTPS 인증서 자동 발급/갱신
6. 데이터 주권이 중요 (금융/의료/공공, GDPR)
7. 라즈베리파이 홈서버 ~ 베어메탈까지 통합 관리

### 부적합

- 트래픽이 급변해 오토스케일링이 필수인 서비스
- 서버 운영을 전혀 하고 싶지 않은 경우
- 멀티리전 / 글로벌 CDN 이 필수인 경우

### 비용 비교

```
Vercel Pro + Supabase Pro + Railway        월 $70~100+
Hetzner CPX31 (4 vCPU / 8 GB) 1대          월 €14 (약 2만원)
                                            → 80~90% 절감
```

---

## 9. 설치 및 사용법

### 9.1 프로덕션 설치

**요구사항**: VPS 1대 (최소 2 vCPU / 2 GB / 30 GB, 권장 4 vCPU / 8 GB), Ubuntu 22.04 또는 24.04, 루트 SSH

```bash
ssh root@<SERVER_IP>
curl -fsSL https://cdn.coollabs.io/coolify/install.sh | bash
```

> 실행 전 스크립트 검토 권장 — 이 저장소의 `scripts/install.sh` 가 해당 파일이다.

**초기 설정**

1. 브라우저로 `http://<SERVER_IP>:8000`
2. 첫 계정 생성 (= 루트 관리자, Team `id=0`)
3. 온보딩: localhost 서버 자동 등록 (`Server id=0`) 또는 다른 서버 SSH 등록
4. Settings → Instance Domain 에 도메인 입력 → HTTPS 자동
5. Settings → Notifications 에서 Discord / Slack / Telegram webhook 등록

**첫 앱 배포**

```
+ New → Project → Environment 선택 → + New Resource
  ├ Public Repository         (URL 붙여넣기, 가장 간단)
  ├ Private Repo (GitHub App) (자동배포 + PR 프리뷰)
  ├ Dockerfile / Docker Compose
  ├ Docker Image
  └ Service                   (342개 템플릿)
→ Build Pack 선택 (Nixpacks / Railpack / Dockerfile / Compose)
→ Domain 입력 → Deploy
```

### 9.2 업그레이드 테스트 절차

```bash
bash upgrade.sh sha-<commit>
docker exec -e COOLIFY_VERSION=4.3.0 coolify php artisan config:cache
# UI 에서 Check for Updates → Upgrade
```

### 9.3 개발 환경

```bash
git clone https://github.com/bmshin94/coolify.git
cd coolify
cp .env.development.example .env

docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
# http://localhost:8000
docker compose -f docker-compose.yml -f docker-compose.dev.yml down
```

**인스턴스 2개 동시 (서버 이전 / 멀티 컨트롤플레인 테스트)**

```bash
./scripts/dev-instances up                 # a:8000 + b:8001
./scripts/dev-instances up a --with vite   # 단일 인스턴스일 때만 HMR
./scripts/dev-instances urls
./scripts/dev-instances down
# 설정: docker-compose.dev-multi.yml, .dev-instances/{a,b}.env
# 듀얼 Vite HMR 미지원 (public/hot 공유) → 멀티는 항상 public/build
```

**상용 명령어**

```bash
php artisan test --compact                             # 전체 테스트
php artisan test --compact --filter=<testName>         # 단일 테스트
php artisan test --compact tests/Feature/SomeTest.php  # 파일 지정
php artisan test --compact tests/v4/Browser/           # 브라우저 테스트
vendor/bin/pint --dirty --format agent                 # 포맷팅 (필수)
npm run dev                                            # Vite HMR
npm run build                                          # 프로덕션 빌드
php artisan horizon                                    # 큐 모니터
php artisan route:list --path=api                      # API 라우트
```

**브라우저 테스트 주의점**

- 신규 테스트는 `tests/v4/Browser/` 에. 레거시 `tests/Browser/` 는 참고하지 말 것
- `visit()` 은 dev 앱(:8000)이나 ChromeDriver(:4444)를 쓰지 않는다. Playwright 서버 + 테스트 프로세스 내 amphp HTTP 서버를 띄운다
- `beforeEach` 에 `InstanceSettings::create(['id' => 0])`, `config()->set('app.maintenance.store', 'array')`, `Team::query()->update(['show_boarding' => false]); Cache::flush();`
- 런이 멈추면 `pkill -f "playwright run-server"`
- 브라우저 테스트 파일은 별도 `php artisan test` 호출로 실행

**기여 브랜치 규칙**

```
main = 프로덕션  → 버그 수정 PR
next = 개발      → 신기능 PR
```

---

## 10. 플러그인 / 스킬 / MCP — 정체 규명

**결론: 셋 다 아니다. 독립 실행형 웹 애플리케이션(SaaS 플랫폼)이다.** 단, 내부에 MCP 서버와 Skill 을 포함한다.

| 분류 | 해당 | 설명 |
|---|:---:|---|
| 독립 웹 앱 | **O** | Laravel 모놀리식. Docker 로 서버에 설치해 상시 구동 |
| MCP 서버 (내장) | O | `app/Mcp/` 45개 툴. AI 가 Coolify 조작 |
| MCP 클라이언트 | O | `.mcp.json` → 개발 시 `laravel-boost` 사용 |
| Claude Skill | 부분 | `.claude/skills/` 11개 — **Coolify 개발용**, 사용자용 기능 아님 |
| Claude Code 플러그인 | X | `.claude-plugin/` 없음, 마켓플레이스 미등록 |
| 브라우저/IDE 확장 | X | |
| CLI 도구 | 부분 | `coolify-cli` 는 별도 저장소 |

### 3단 구조

```
1층) Coolify 본체 = 웹 애플리케이션        ← 제품
2층) 내장 MCP 서버 (app/Mcp/)              ← AI 용 리모컨
3층) .claude/skills/ 11개                  ← 소스 수정용 개발 가이드
```

---

## 11. API 토큰

### 용도별 필요 여부

| 용도 | 토큰 |
|---|---|
| 브라우저 UI 사용 | 불필요 (세션 로그인) |
| REST API (297개) | **필수** |
| MCP 연결 | **필수** |
| Git push 자동배포 | 불필요 (Webhook secret) |
| Deploy Webhook 수동 호출 | **필수** |
| 소스 개발/학습 | 불필요 |

### 발급

```
로그인 → 프로필 → Keys & Tokens → API tokens → 이름 + 권한 → Create
(토큰은 생성 시 1회만 표시)
```

### 권한(ability) 6종

| 권한 | 설명 |
|---|---|
| `read` | 읽기 전용 (기본값) |
| `read:sensitive` | 민감정보 읽기 (선택 시 `read` 자동 추가) |
| `write` | 생성/수정/삭제 |
| `write:sensitive` | 민감정보 쓰기 |
| `deploy` | 배포/시작/정지/재시작 전용 (선택 시 다른 권한 전부 제거) |
| `root` | 전체 권한 (선택 시 `['root']` 만 남음) |

Policy 2차 검증: `useRootPermissions`, `useWritePermissions`, `useDeployPermissions`
미들웨어: `ApiAbility` (권한 체크), `ApiSensitiveData` (민감 필드 마스킹)

### 사용 예시

```bash
TOKEN="<API_TOKEN>"
BASE="https://coolify.example.com/api/v1"

curl -H "Authorization: Bearer $TOKEN" $BASE/servers
curl -H "Authorization: Bearer $TOKEN" $BASE/applications
curl -H "Authorization: Bearer $TOKEN" "$BASE/deploy?uuid=<APP_UUID>&force=false"
curl -X POST -H "Authorization: Bearer $TOKEN" $BASE/applications/<APP_UUID>/restart
curl -X POST -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
  -d '{"key":"API_KEY","value":"xxx","is_preview":false}' \
  $BASE/applications/<APP_UUID>/envs
```

### 보안 권고

1. CI/CD 용은 `deploy` 권한만 발급 — 유출 시 데이터 노출 없음
2. 용도별 토큰 분리 (GitHub Actions / 모니터링 / AI)
3. 로그에 토큰이 남지 않도록 주의
4. API IP 화이트리스트 활용
5. 전체 스펙은 `openapi.json` 또는 `/docs/api`

---

## 12. AI 에이전트 구축에 주는 도움

### 12.1 에이전트의 "손발" — Coolify 를 툴로 사용

```
내 AI 에이전트 --MCP/HTTP--> Coolify --SSH--> 실제 서버
```

구현 가능한 에이전트:

- **DevOps 자율 에이전트** — 배포 실패 감지 → 로그 분석 → 원인 추정 → 롤백/재배포 → Slack 보고
- **온콜 1차 대응 봇** — 앱 다운 감지 → `list_unhealthy_resources` → `control(restart)` → 실패 시 사람 호출
- **AI 생성 앱의 자동 배포** — 코드 생성 에이전트 → GitHub push → Coolify API 배포 → 프리뷰 URL 반환
- **비용 최적화 에이전트** — 유휴 리소스 탐지 및 정지 제안
- **셀프서비스 봇** — "스테이징에 Postgres 하나" → 자동 프로비저닝 + 접속정보 DM

### 12.2 에이전트의 "몸" — AI 에이전트를 Coolify 에 배포

템플릿에 이미 AI 관련 서비스가 다수 존재:

| 템플릿 | 용도 |
|---|---|
| `anythingllm` | RAG 챗봇 |
| `n8n` | 에이전트 워크플로우 오케스트레이션 |
| `appsmith` / `budibase` | 어드민 UI |
| `argilla` | 데이터 레이블링 |
| `browserless` | 헤드리스 브라우저 (에이전트 웹 조작) |
| `authentik` | SSO / 인증 |
| Postgres(pgvector), Redis, ClickHouse | 벡터DB / 캐시 / 분석 |

VPS 1대에 AI 에이전트 풀스택 전체를 월 2~3만원 수준으로 구성 가능 (LLM API 비용 별도).

### 12.3 에이전트의 "교과서" — 설계 패턴

`app/Mcp/` 는 프로덕션급 MCP 서버 설계의 드문 대형 오픈소스 예제다. 4장의 8가지 패턴 참조.
추가로 `laravel/mcp ^0.6.7` 실전 코드와 `.claude/skills/mcp-development/SKILL.md` 가이드가 함께 들어있어, PHP 로 MCP 를 만들 때 최상의 레퍼런스가 된다.

---

## 13. React / PHP 로 만들 수 있나

### 13.1 PHP — 이미 PHP다

| 난이도 | 작업 | 소요 | 대상 |
|:---:|---|---|---|
| ★ | 새 서비스 템플릿 추가 | 30분 | `templates/compose/*.yaml` |
| ★ | 한국어 번역 개선 | 1시간 | `lang/` |
| ★★ | 새 알림 채널 (카카오톡 등) | 반나절 | `app/Notifications/` + `app/Models/*NotificationSettings.php` |
| ★★ | 새 MCP 툴 추가 | 2시간 | `app/Mcp/Tools/` + `CoolifyServer.php` 등록 |
| ★★ | 새 API 엔드포인트 | 2시간 | `routes/api.php` + `app/Http/Controllers/Api/` |
| ★★★ | 새 DB 타입 지원 | 2~3일 | `app/Models/Standalone*.php` + Actions + Livewire + 마이그레이션 |
| ★★★ | 대시보드 위젯 | 1~2일 | `app/Livewire/Dashboard.php` + Blade |
| ★★★★ | 새 클라우드 프로바이더 | 1~2주 | `app/Services/*Service.php` (Hetzner/DO/Vultr 참고) |
| ★★★★★ | 새 빌드팩 | 2~4주 | `app/Jobs/ApplicationDeploymentJob.php` |

**MCP 툴 추가 스켈레톤**

```php
<?php

namespace App\Mcp\Tools;

use App\Mcp\Concerns\BuildsResponse;
use App\Mcp\Concerns\ResolvesTeam;
use Laravel\Mcp\Request;
use Laravel\Mcp\Response;
use Laravel\Mcp\Server\Tool;

class MyAwesomeTool extends Tool
{
    use BuildsResponse;
    use ResolvesTeam;

    protected string $name = 'my_awesome_tool';

    protected string $description = 'AI 가 읽는 설명문 — 호출 조건과 반환값을 명확히.';

    public function handle(Request $request): Response
    {
        $teamId = $this->resolveTeamId($request);
        if (is_null($teamId)) {
            return $this->mcpError($request, 'Invalid token.');
        }

        return $this->mcpSuccess($request, $this->respond(['ok' => true]));
    }
}
```

이후 `CoolifyServer::$tools` 배열에 클래스 등록.

**필수 규칙**

```bash
php artisan make:class --no-interaction        # 파일 생성은 artisan 으로
vendor/bin/pint --dirty --format agent         # 포맷팅 (미실행 시 CI 실패)
php artisan test --compact --filter=<name>     # 테스트 필수
```

- `DB::` 파사드 대신 `Model::query()` 와 Eloquent 관계 사용
- Form Request 대신 인라인 `Validator` + `app/Rules/` 커스텀 규칙
- PHP 8.4+: 생성자 프로퍼티 승격, 명시적 반환 타입, 타입 힌트
- 모든 변경에 테스트. 버그 수정은 TDD (실패 테스트 먼저)
- 원격 셸 명령 추가 시 비-root SSH 사용자 고려 — `parseCommandsByLineForSudo()` 를 통과하므로 파이프/리다이렉트/치환/`sh -c` 스크립트를 sudo 파서로 검증

### 13.2 React — 3가지 경로

**경로 1: Coolify API 를 소비하는 React 앱 (권장)**

```
React / Next.js (신규 개발) --fetch--> Coolify REST API(297) + MCP --> 서버들
```

구현 가능한 것:
- 모바일 최적화 대시보드
- 다중 Coolify 인스턴스 통합 관제탑
- 경영진용 비용/상태 리포트
- AI 챗 기반 인프라 운영 UI
- React Native 모바일 앱

Coolify 본체를 건드리지 않으므로 업스트림 업데이트 충돌이 없고 수익화도 가장 쉽다.

**경로 2: Coolify 내부에 React 심기 (비권장)**

`docs/v5/ui/` 에 React + TypeScript + shadcn/ui 캔버스 UI 프로토타입이 보관돼 있으나, README 에 *"Do not import or serve these files. V5 will use a different implementation."* 로 명시. Inertia.js + React 를 붙여 Livewire 232개와 공존시키는 비용이 크다.

**경로 3: 처음부터 React 로 클론 (비현실적)**

785개 PHP 파일 + 297개 API + 342개 템플릿 + SSH/Docker/Traefik 오케스트레이션 → 최소 1~2년 + 팀 필요. 학습용 "미니 Coolify"(단일 서버, Docker 배포만)는 좋은 연습 과제.

**권장 조합**

```
1단계 Coolify 그대로 설치·운영 (1주)
2단계 PHP 로 소규모 기능 추가 — MCP 툴 1개, 알림 채널 1개 (2주)
3단계 React 로 API 소비 커스텀 대시보드 (1개월)   ← 수익화 지점
4단계 AI 에이전트 레이어 (2개월)                   ← 차별화
```

---

## 14. 유튜브 강의 제작 가능성

**가능하며 주제성이 매우 좋다.** 라이선스는 **Apache 2.0** 으로 상업적 강의 제작에 제약이 없다.

### 주제가 좋은 이유

1. "비용 절감" 주제 = 높은 조회수
2. 한국어 콘텐츠 희소 = 블루오션
3. 결과가 시각적 (클릭 → 배포 → HTTPS)
4. 타겟 명확 (개발자, 인디해커, 스타트업 CTO)
5. 반복 소비 (레퍼런스로 재방문)
6. 수익화 연결 용이 (VPS 제휴 + 컨설팅 유입)

### 커리큘럼 12편

| # | 제목 | 길이 | 핵심 |
|:---:|---|---|---|
| 0 | "Vercel 요금 90% 줄이는 방법" (후킹) | 8분 | 요금 비교, 결과 선공개 |
| 1 | Coolify 개념 완전정리 | 12분 | 비유 설명, 장단점 |
| 2 | VPS 고르기 + 5분 설치 | 15분 | Hetzner/Contabo/Vultr 비교 (제휴) |
| 3 | 첫 Next.js 배포 + 도메인 + HTTPS | 18분 | 최다 수요 예상 |
| 4 | GitHub 자동배포 + PR 프리뷰 | 15분 | GitHub App 연동 |
| 5 | DB + S3 자동백업 | 20분 | Postgres + MinIO/R2 |
| 6 | 원클릭 342개 서비스 (n8n, Supabase) | 18분 | 템플릿 쇼케이스 |
| 7 | Docker Compose + 멀티서버 | 20분 | 중급 |
| 8 | **Coolify MCP 로 Claude 가 서버 관리** | 20분 | **차별화 핵심** |
| 9 | API 로 CI/CD 자동화 | 18분 | 토큰 권한 설계 |
| 10 | 보안 점검 (방화벽/SSH/백업/2FA) | 20분 | 신뢰도 |
| 11 | 실전 장애 대응 삽질기 5가지 | 15분 | 최고 신뢰도 |

### 고급 시리즈 (희소성 높음)

- Laravel 12 + Livewire 3 실무 패턴 해부 (Coolify 코드 리딩)
- MCP 서버 제대로 설계하기 — Coolify 45개 툴 분석
- PHP 로 MCP 툴 만들기 라이브 코딩
- React 로 Coolify 커스텀 대시보드 만들기

### 수익 루트

| 루트 | 예상 |
|---|---|
| 애드센스 | 10만 조회 ≈ 30~80만원 |
| **VPS 제휴** (Hetzner, Contabo, Hostinger, Vultr, DO) | 건당 $10~50 — 메인 |
| 유료 강의 (인프런/클래스101) | 코스당 10~30만원 |
| 컨설팅 유입 | 영상 → 설치 대행 문의 |
| 멤버십 | 월 구독, 우선 질의 |
| 템플릿/스타터킷 | 디지털 상품 |

### 주의사항

- 영상에 API 토큰 / SSH 키 / 실도메인 노출 금지 (블러)
- 버전 명시 (`4.3.22` 기준). 업데이트 속도가 빨라 영상이 빠르게 구버전화
- "Coolify 공식 채널 아님" 명시, 로고/상표 사용 신중
- 단점(오토스케일링 없음, 운영 책임 본인) 솔직히 언급 → 신뢰도 상승
- 제휴링크는 광고 표기 필수 (유튜브 정책 + 공정위 추천·보증 심사지침)
- 터미널 글꼴 확대, 챕터 타임스탬프, 결과 선공개

---

## 15. 수익화 아이디어 13선

### Tier 1 — 즉시 시작 가능 (초기비용 ≈ 0)

#### 1. Coolify 설치·구축 대행

| 항목 | 내용 |
|---|---|
| 고객 | 1인 개발자, 디자이너, 소규모 쇼핑몰, 비개발 창업자 |
| 가격 | 기본 설치 30~50만원 / 마이그레이션 포함 80~200만원 |
| 소요 | 기본 2~4시간, 마이그레이션 1~3일 |
| 원가 | 0원 |
| 채널 | 크몽, 숨고, 위시켓, 당근 전문가, 개발자 커뮤니티 |

```
베이직  (30만원)  VPS 세팅 + 설치 + 도메인/HTTPS + 앱 1개 + 1시간 교육
스탠다드(80만원)  + DB 2개 + S3 백업 + 알림 + GitHub 자동배포 + 보안 기본 + 1개월 지원
프리미엄(200만원+) + 무중단 마이그레이션 + 멀티서버 + Grafana + 운영 매뉴얼 + 3개월 지원
```

설치 자동화 스크립트를 만들어두면 2시간 → 30분으로 단축되어 시간당 단가가 급증한다.

#### 2. 콘텐츠 + 제휴 마케팅

14장 참조. 유튜브는 직접 수익보다 **신뢰 자산**으로서의 가치가 크다. 영상 설명란에 설치 대행 문의 링크 필수.

#### 3. 한국형 서비스 템플릿 번들

342개 템플릿은 전부 해외 서비스. 한국 환경 템플릿을 제작해 판매.

- 토스페이먼츠 / 포트원 결제 연동 스타터
- 카카오·네이버 소셜로그인 포함 Laravel / Next.js 스타터
- 네이버 클라우드 / NHN Cloud Object Storage 백업 설정
- 한국어 최적화 Ghost / WordPress
- 카카오톡 알림톡 연동
- 국내 쇼핑몰(카페24/고도몰) 연동 미들웨어

| 상품 | 가격 |
|---|---|
| 개별 템플릿 | 1~3만원 |
| 번들 10개 | 5~10만원 |
| 평생 업데이트 번들 | 15~30만원 |

채널: Gumroad, Lemon Squeezy, 자체 사이트, 크몽. 원가 0, 패시브 인컴.

#### 4. 트러블슈팅 / 긴급 대응

```
시간당 상담       10~15만원
긴급 대응 (24h)   30~50만원
월 유지보수 구독  월 20~50만원
```

월 구독이 핵심. 고객 10명 × 월 30만원 = 월 300만원 안정 수익. Discord 커뮤니티 무료 지원으로 신뢰를 쌓고 유료 전환.

### Tier 2 — 1~3개월 준비

#### 5. 중소기업 서버 관리 아웃소싱

| 항목 | 내용 |
|---|---|
| 고객 | 직원 10~50명, 개발자 0~1명 |
| 가격 | 월 50~200만원 / 회사 |
| 제공 | Coolify 기반 사내 시스템 운영 + 백업 + 모니터링 + 장애대응 + 월간 리포트 |

셀링 포인트: "개발자 1명 채용 = 연 5천만원. 월 100만원에 서버 관리 전담." / "AWS 월 300만원 → 인프라 30 + 관리 100 = 130만원."

```
VPS 3대          월 10만원
고객 5개사       월 500만원 매출 - 50만원 원가 = 순익 450만원
```

가장 현실적이고 규모가 큰 모델. 영업 채널 필요 (상공회의소, 중소기업 지원센터).

#### 6. 관리형 Coolify 호스팅

| 플랜 | 가격 | 내용 |
|---|---|---|
| 스타터 | 월 1.5만원 | 앱 3개, 공용 2 GB |
| 프로 | 월 4만원 | 앱 10개, 전용 4 GB |
| 비즈 | 월 10만원 | 전용 8 GB, 백업, 우선지원 |

차별점: 한국어 지원, 국내 리전, 카카오톡 상담.
주의: 24시간 대응, 결제 시스템, 약관, 개인정보처리방침, SLA 필요. 초기에는 고객 5명 한정 베타 권장. Apache 2.0 이므로 상업적 호스팅은 가능하나 **Coolify 브랜드를 자사 제품명처럼 쓰면 안 된다** ("Coolify 기반" 표기).

#### 7. AI DevOps 에이전트 SaaS (가장 트렌디)

```
내 SaaS: AI 인프라 운영 비서
  - 장애 자동 감지 → 원인 분석 → 자동 복구
  - 일일/주간 인프라 리포트 (AI 작성)
  - Slack/카톡 채팅으로 인프라 조작
  - 비용 최적화 제안
  - 배포 실패 원인 분석 + 수정 PR 제안
        ↓ MCP (45개 툴 그대로 활용)
  고객의 Coolify 인스턴스
```

| 플랜 | 가격 |
|---|---|
| 개인 | 월 2~3만원 |
| 팀 | 월 10만원 |
| 기업 | 월 30만원+ |

지금 해야 하는 이유: MCP 생태계 초기 = 선점 기회 / Coolify MCP 45개 툴로 인프라 개발 불필요 / "AI 에이전트 + 셀프호스팅" 조합의 경쟁자 거의 없음.
원가는 LLM API 비용 (경량 모델로 대부분 처리 시 고객당 월 수천원), 마진 80%+.

#### 8. AI 에이전트 스타터킷 판매

```
AI Agent Stack Starter Kit
  Coolify 설치 스크립트
  Postgres + pgvector / Redis / n8n / browserless
  AnythingLLM / Langfuse·Grafana
  FastAPI 에이전트 보일러플레이트
  설정 가이드 PDF + 영상
```

| 상품 | 가격 |
|---|---|
| 스타터킷 (코드+문서) | 10~30만원 |
| + 1:1 세팅 지원 | 50만원 |
| + 커스터마이징 | 100만원+ |

### Tier 3 — 장기

#### 9. 유료 강의

```
Coolify 완전정복: 셀프호스팅 마스터          10~15만원
Laravel 12 + Livewire 3 실무 패턴            20~30만원
MCP 서버 설계 마스터클래스                   25~40만원  ← 희소성 최고
```

14장의 유튜브 시리즈를 재편집해 유료 코스화하면 제작비 절감.

#### 10. 기술 문서 / 전자책

- 한국 개발자를 위한 Coolify 완전 가이드 (2~3만원)
- 셀프호스팅 비용 절감 플레이북 (기업용, 10만원)
- Coolify 공식 문서 한국어 번역 → 직접 수익은 없으나 브랜딩 효과 최대

#### 11. 특수 시장 — 데이터 주권

| 시장 | 셀프호스팅이 필요한 이유 |
|---|---|
| 의료/병원 | 환자 데이터 외부 유출 불가 (의료법) |
| 금융/핀테크 | 금융권 클라우드 규제 |
| 공공기관 | 국내 서버 의무, CSAP |
| 법무법인 | 고객 비밀유지 의무 |
| 교육기관 | 학생 개인정보 보호 |
| 유럽 진출 기업 | GDPR 데이터 거주 요건 |

```
구축 500만원~ + 월 유지보수 100~300만원
```

단가는 Tier 1 의 10배. 대신 보안 감사 대응, 문서화, 책임보험 필요. 진입장벽이 높아 경쟁이 적다.

#### 12. 멀티 Coolify 통합 관제 플랫폼 (React)

```
React 대시보드 (자사 제품) --API 297개 × N 인스턴스--> 고객사 A/B/C Coolify
```

고객: 에이전시, MSP, 다수 고객사를 관리하는 프리랜서
가격: 인스턴스당 월 2~5만원 또는 에이전시 라이선스 월 30만원
장점: Coolify 본체 미수정 → 업데이트 충돌 없음, React 로 구현 가능

#### 13. 플러그인 / 확장 마켓

Coolify 에 공식 플러그인 생태계가 아직 없다. 선점 기회.

- 네이버클라우드 / NHN Cloud / 카카오클라우드 프로바이더 모듈
- 카카오톡 알림톡 알림 채널
- 한국 결제사 연동 모듈
- 사내 LDAP / AD 인증 연동

업스트림 기여를 통해 "Coolify 한국 전문가" 포지션 확보 → 다른 모든 수익의 신뢰 기반.

---

## 16. 추천 로드맵

```
1개월차 — 기반
  VPS 1대 + Coolify 설치, 본인 프로젝트 2~3개 운영
  삽질 기록 블로그 포스팅 (SEO 자산)
  Coolify Discord 한국 유저 무료 지원 (신뢰 구축)
  수익: 0원

2~3개월차 — 첫 수익
  유튜브 0~3편 업로드 (후킹 + 설치 + 첫 배포)
  크몽/숨고에 설치 대행 등록
  VPS 제휴 프로그램 가입
  수익: 월 30~100만원

4~6개월차 — 확장
  유튜브 10편 완성 (MCP 편 포함)
  한국형 템플릿 번들 제작 → Gumroad 판매
  중소기업 1~2곳 월 유지보수 계약
  React 커스텀 대시보드 프로토타입
  수익: 월 200~400만원

7~12개월차 — 스케일
  AI DevOps 에이전트 SaaS 베타 출시
  인프런 유료 강의 런칭
  중소기업 유지보수 5곳
  "Coolify 한국 전문가" 브랜드 확립
  수익 목표: 월 500~1000만원
```

**권장 조합**: ① 설치 대행 (즉시 현금) + ② 유튜브 (신뢰 자산) → ⑦ AI DevOps SaaS (스케일)

이유: 대행으로 현금흐름을 만들며 실전 노하우 축적 → 유튜브가 영업을 대체 → 실제 고객 문제를 아는 상태로 SaaS 제작 → 성공률 상승. 특히 ⑦은 MCP 생태계 초기라 타이밍이 중요하다.

---

## 17. 법적 체크리스트

| 항목 | 내용 |
|---|---|
| 라이선스 | **Apache 2.0** — 상업적 이용 / 수정 / 재배포 허용 |
| 고지 의무 | 코드 재배포 시 저작권 고지 + 라이선스 사본 포함, 변경사항 명시 |
| 상표권 | "Coolify" 이름/로고를 자사 제품명처럼 사용 금지. "Coolify 기반", "Coolify 전문" 식 표현 |
| 호스팅 사업 | 부가통신사업자 신고 여부 확인 |
| 개인정보 | 고객 데이터 취급 시 개인정보처리방침 + 안전조치 의무 |
| 제휴링크 | 유튜브/블로그 광고 표기 필수 (공정위 추천·보증 심사지침) |
| SLA | 유지보수 계약서에 책임 범위 명시 (무한 책임 회피) |
| 보험 | 기업 상대 사업 시 IT 배상책임보험 검토 |

---

## 부록 A. 참고 링크

| 항목 | URL |
|---|---|
| **이 저장소 (포크)** | https://github.com/bmshin94/coolify |
| 업스트림 원본 | https://github.com/coollabsio/coolify |
| 공식 웹사이트 | https://coolify.io |
| 공식 문서 | https://coolify.io/docs |
| 설치 가이드 | https://coolify.io/docs/installation |
| Coolify Cloud | https://app.coolify.io |
| Discord | https://coollabs.io/discord |
| GitHub Discussions | https://github.com/coollabsio/coolify/discussions |
| 철학 | https://coolify.io/philosophy |
| 스폰서십 | https://coolify.io/sponsorships |
| 설치 스크립트 | https://cdn.coollabs.io/coolify/install.sh |
| 버전 정보 JSON | https://cdn.coollabs.io/coolify/versions.json |

## 부록 B. 저장소 내 필독 파일

| 파일 | 왜 읽어야 하나 |
|---|---|
| `CLAUDE.md` / `AGENTS.md` | 아키텍처, 관례, 함정 총정리 (31 KB / 28 KB) |
| `DESIGN.md` | UI/UX 디자인 단일 진실 공급원 (36 KB) |
| `CONTRIBUTING.md` | 기여 절차, 브랜치 규칙 |
| `DEVELOPMENT.md` | 개발 환경 상세 |
| `RELEASE.md` | 릴리스 프로세스 |
| `SECURITY.md` / `SECURITY_ADVISORY.md` | 보안 정책 |
| `.ai/lessons.md` | AI 가 실수한 교훈 누적 기록 |
| `app/Mcp/Servers/CoolifyServer.php` | MCP 서버 전체 구성 |
| `app/Mcp/Tools/Deploy.php` | 권한·감사로그·next_tools 패턴 |
| `app/Mcp/Tools/CoolifyHelp.php` | 의도 기반 툴 카탈로그 설계 |
| `app/Jobs/ApplicationDeploymentJob.php` | 배포 파이프라인의 심장 |
| `routes/api.php` | 297개 엔드포인트 전체 |
| `templates/service-templates-latest.json` | 342개 서비스 템플릿 |
| `scripts/install.sh` | 원클릭 설치 스크립트 (실행 전 검토) |
| `docs/v5/ui/README.md` | 차기 V5 UI 방향성 |

---

*이 문서는 `bmshin94/coolify` 저장소를 전수조사한 결과를 정리한 것이다. 분석 시점의 버전은 `4.3.22` 이며, Coolify 는 업데이트가 빈번하므로 최신 정보는 공식 문서를 함께 확인할 것.*
