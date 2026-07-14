# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository layout

The repo root holds three trees plus supporting docs:

- `vhr/` — the Java backend. A Maven multi-module project rooted at `vhr/pom.xml` (Spring Boot `2.1.8.RELEASE`, Java 1.8). Two top-level modules:
  - `vhr/vhrserver/` (packaging `pom`) — the main HR server, further split into four modules that must be built in dependency order: **`vhr-model`** (POJOs / DTOs) → **`vhr-mapper`** (MyBatis mappers + Flyway + Druid) → **`vhr-service`** (business services, Redis cache, RabbitMQ producer, POI, `@EnableCaching` config) → **`vhr-web`** (Spring MVC controllers, Spring Security, WebSocket, FastDFS, static frontend). Only `vhr-web` is a runnable Spring Boot app (`org.javaboy.vhr.VhrApplication`, `@MapperScan("org.javaboy.vhr.mapper")`, `@EnableScheduling`, `@EnableCaching`, port `8081`).
  - `vhr/mailserver/` — a standalone Spring Boot app (`org.javaboy.mailserver.MailserverApplication`, port `8082`) that consumes RabbitMQ messages and sends emails via Thymeleaf templates + `spring-boot-starter-mail`. Depends on `vhr-model` for the shared message payload.
- `vuehr/` — Vue 2 + ElementUI frontend built with vue-cli 3. During dev, `vue.config.js` proxies `/` → `http://localhost:8081` and `/ws` → `ws://localhost:8081`, so the backend must be running for the SPA to work.
- `vhr.sql` — legacy full dump. **Do not import it manually** — Flyway runs `vhr/vhrserver/vhr-web/src/main/resources/db/migration/V1__vhr.sql` against an empty `vhr` database on first startup of `vhr-web`.

## Commands

Backend (run from `vhr/`):

```bash
# Full build of both top-level modules (vhrserver + mailserver) — installs vhr-model / vhr-mapper / vhr-service into the local Maven repo so vhr-web and mailserver can resolve them.
mvn -f vhr/pom.xml clean install -DskipTests

# Run the main HR server (must be started AFTER Redis + RabbitMQ + MySQL are up; Flyway auto-migrates on first run).
mvn -f vhr/vhrserver/vhr-web/pom.xml spring-boot:run

# Run the mail worker.
mvn -f vhr/mailserver/pom.xml spring-boot:run

# Tests for a single module.
mvn -f vhr/vhrserver/vhr-web/pom.xml test
# Single test class / method.
mvn -f vhr/vhrserver/vhr-web/pom.xml test -Dtest=SomeTest
mvn -f vhr/vhrserver/vhr-web/pom.xml test -Dtest=SomeTest#methodName
```

There is a Maven wrapper under `vhr/mailserver/` (`./mvnw`) but **not** at the repo root or under `vhrserver/`; use system `mvn` for cross-module builds.

Frontend (run from `vuehr/`):

```bash
npm install
npm run serve   # dev server on :8080, proxies to :8081
npm run build   # emits dist/; copy dist/static and dist/index.html to vhr-web/src/main/resources/static/ for prod bundling
```

## Runtime dependencies

`vhr-web` will not start without all of these reachable at the addresses in `vhr/vhrserver/vhr-web/src/main/resources/application.properties`:

- **MySQL** on `localhost:3306`, empty database named `vhr` (schema is managed by Flyway; do not hand-run `vhr.sql`).
- **Redis** on `localhost:6379` — used by `spring.cache.cache-names=menus_cache` and for chat/session state.
- **RabbitMQ** on `localhost:5672` — the exchange/queue/routing-key names are constants in `vhr-model` (`MailConstants`) and must be identically configured in both `vhr-web` and `mailserver` `application.properties`.
- **FastDFS + Nginx** — only required if exercising avatar / file-upload flows; endpoint is `fastdfs.nginx.host`.

The mail worker additionally needs an SMTP account configured in `vhr/mailserver/src/main/resources/application.properties` (`spring.mail.*`, currently a QQ Mail authorization code — replace before running).

## Architecture notes worth loading before editing

- **Module dependency direction is one-way**: `model ← mapper ← service ← web`, and `mailserver` also depends on `model`. Adding a class in the wrong module (e.g. a controller in `service`) will silently break the packaging; keep controllers in `vhr-web`, `@Service`/`@Cacheable` beans in `vhr-service`, MyBatis `*Mapper` interfaces + their sibling `*Mapper.xml` files together under `vhr-mapper/src/main/java/org/javaboy/vhr/mapper/` (both `pom.xml` files declare `src/main/java` as a resource root so the XMLs ship alongside the interfaces).
- **Flyway migrations live under `vhr-web`**, not `vhr-mapper`, at `src/main/resources/db/migration/`. Any schema change is a new `V<n>__desc.sql` file there — never edit `V1__vhr.sql`.
- **Auth / authorization is dynamic**: `SecurityConfig` (in `vhr-web/config/`) wires `CustomFilterInvocationSecurityMetadataSource` (loads menu→role rules from DB) and `CustomUrlDecisionManager` (evaluates them per request), plus a captcha filter (`VerificationCodeFilter` backed by `VerificationCode`). Role/menu data drives the frontend menu tree too — the same `MenuService` endpoint feeds both authorization and UI.
- **Employee onboarding email flow**: adding an employee in `EmployeeService` publishes to `MailConstants.MAIL_EXCHANGE_NAME` with routing key `MailConstants.MAIL_ROUTING_KEY_NAME`; `mailserver`'s `MailReceiver` (`@RabbitListener`) picks it up, renders a Thymeleaf template, and sends via JavaMailSender. A `mail_send_log` row is written before publishing; `MailSendTask` (scheduled) resends unacknowledged entries. Any change to the payload class **must be made in `vhr-model`** so both sides deserialize the same shape.
- **WebSocket / chat**: STOMP endpoints are configured in `WebSocketConfig`; `ChatController` / `WsController` handle per-user messaging and admin system-notifications. The Vue dev proxy forwards `/ws` specifically because of this.
- **Frontend↔backend contract**: the Vue app assumes it is served from the same origin as the API (either via the dev proxy or by copying the built `dist/` into `vhr-web`'s `resources/static/`). Cross-origin deploys are not configured — CORS is not enabled in `SecurityConfig`.

## Working with configuration

`application.properties` files under `vhr-web/` and `mailserver/` contain committed local credentials (DB password, SMTP authorization code, RabbitMQ guest/guest). Treat them as developer defaults, not secrets to reuse. If you need to change service endpoints for a task, edit both files consistently — the mail flow silently drops messages when the two RabbitMQ configs disagree.
