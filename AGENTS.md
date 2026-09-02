# AGENTS.md — Aplico

Job application tracker (kanban board of roles applied to). **This repo is a study artefact.**
The code exists to teach Java 25 / Spring Boot 4.1 / Angular 22 signals / a free CI-CD pipeline,
and to be defended out loud in an interview. Optimise for *"the author can explain this line"*,
not for cleverness. When a simpler option is defensible, take it and write down why.

## Stack — pinned, do not upgrade or substitute without asking
**Java 25 (LTS)** · Spring Boot 4.1.x (supports Java 17–26) · Maven **via `./mvnw` only — Maven
is not installed on this machine** · Spring Data JPA · Flyway · PostgreSQL 17 ·
Angular 22 + TypeScript strict (**Node ≥ 24.15**, the v22 floor) · Angular Material ·
JUnit 5 + Mockito + Testcontainers · Docker Engine + Compose v2 inside WSL (`docker compose`,
not `docker-compose`) · GitHub Actions · **Render (API) + Neon (Postgres) + Cloudflare Pages
(frontend)** — deployed as two separate artefacts, on purpose, so that CORS is real.

Local environment: WSL Ubuntu on Windows. Everything — build, tests, Docker — runs **inside
WSL**, never from Windows, or Testcontainers cannot reach the Docker socket.

⚠️ **Spring Boot 3.x is a different framework line.** 4.0 (Nov 2025) changed Jackson to 3.x,
Jakarta Validation to 3.1, and restructured modules; most blog posts and Stack Overflow answers
are 3.x. Rules: generate dependencies from start.spring.io, never paste a `<dependency>` block
from a blog, and check the version number in the URL of any docs page before trusting it.
Prefer official docs (docs.spring.io/spring-boot/4.1.x, angular.dev) over tutorials. Use
Context7 MCP for current docs when available.

## Hard rules
- **Timebox 3 weeks / ~18 h** (6 h/week). If a feature grows, cut scope — never extend the box.
- **Max 2 tables, 8 endpoints, 2 screens.** Adding one of anything requires removing one.
- **Free forever.** No paid tier, no trial that expires, no card. If a step looks like it needs
  payment, **stop and ask** — do not sign up.
- **Never push, and never commit to `main`.** Feature branch + PR; ask before every push.
- Never commit `.env`, credentials, or `docs/study/**`.

## Backend conventions
- All routes under `/api/`. Errors are RFC 9457 `ProblemDetail`, produced by one
  `@RestControllerAdvice`. Never leak stack traces, SQL, or class names to the client.
  Bean-validation failures → **400**. Conflict with domain state → **409**. Not found → **404**.
- Java **records** for request/response DTOs. **Never** serialise a JPA entity to JSON.
- **Constructor injection only.** No `@Autowired` on fields. No field/setter injection.
- Layers: `controller` → `service` → `repository`. Controllers hold no business logic;
  repositories are never called from a controller. Mapping entity↔DTO lives in the service
  (or a small static `Mapper`) — no MapStruct, no Lombok (write the boilerplate; it is study material).
- Flyway owns the schema. `spring.jpa.hibernate.ddl-auto: validate`, **never** `update`.
  Migrations are immutable once merged: fix forward with a new `V{n}__`.
- `@Enumerated(EnumType.STRING)` always, never `ORDINAL`. Enums are also `CHECK`-constrained in SQL.
- Timestamps are `timestamptz` / `Instant`. Calendar days are `date` / `LocalDate`.
- No SQL reserved words as column names (`position`, `user`, `order`, `end`…). `job_title`, not
  `position`.

## Data & privacy — two environments, two datasets
**The deployed instance never contains real job applications.** Her actual search — real company
names, salary expectations, "rejected by X", notes that may name people — stays on the laptop.

- **Public (Render + Neon):** fictional seed data only, loaded by a Flyway seed migration under
  the `demo` profile. Invented company names — never a real employer, even in a made-up
  rejection. A visible "Demo data" chip in the header.
- **Private (local):** `docker compose up` + local Postgres. This is the instance used daily.
- The two never share a database. **Never point the local app at the Neon connection string**,
  and never run the demo seed against a database holding real rows.
- `.env` is git-ignored and holds the real values; `.env.example` documents names only. The
  production connection string lives in Render's environment variables, never in the repo.
- The README states plainly that the live data is fictional.

## Security & CORS — the split deploy makes this real
- Single hardcoded user, HTTP Basic, one explicit `SecurityFilterChain` bean.
  `GET /api/**` and `/actuator/health` are public (recruiters must see the board);
  every write requires authentication.
- CORS is configured **once**, as a `CorsConfigurationSource` bean wired into the security chain
  via `http.cors(...)`. **Never `@CrossOrigin` annotations** scattered on controllers — and never
  a CORS filter outside the chain, or the preflight `OPTIONS` gets a 401 before CORS is applied.
  Allowed origins come from a property (`app.cors.allowed-origins`), different per environment:
  `http://localhost:4200` locally, the Cloudflare Pages origin in production. No wildcards.
- The Angular app sends the `Authorization: Basic …` header **explicitly** on writes, from
  credentials held in an in-memory signal. Override `BasicAuthenticationEntryPoint` so a 401
  returns a clean `ProblemDetail` **without** the `WWW-Authenticate` header — otherwise the
  browser hijacks the flow with its native popup.
- CSRF disabled with a written justification comment (stateless API, no cookie-borne credential).
- Username/password come from environment variables; the password is stored as a bcrypt hash,
  never plaintext, never in `application.yml`. `.env.example` documents names only.

## Frontend conventions — mobile-first is a hard requirement
- **Design at 360 px first, then widen.** A recruiter opens this on a phone. No horizontal page
  scroll at 360 px, ever. Breakpoints add columns; they never rescue a broken small screen.
- The board on a phone shows **one status column at a time**, chosen with a segmented control
  that carries the counts — not five columns squeezed side by side. On ≥1024 px the columns sit
  next to each other.
- **No drag-and-drop.** Status changes through a menu on the card: correct on touch, reachable by
  keyboard, and free. This is a design decision, not a shortcut — be able to say so.
- Tap targets ≥ 44 px. No hover-only affordances. Every interactive element reachable by keyboard
  with a visible focus ring.
- Standalone components, `ChangeDetectionStrategy.OnPush`, zoneless (Angular 21+ default).
- State is **signals**: `signal` for sources of truth, `computed` for anything derivable,
  `linkedSignal` where local state must reset when an input changes, `resource`/`httpResource`
  for server reads. **No derived value stored in a plain field and manually kept in sync.**
- `camelCase` variables/functions, `PascalCase` components, `kebab-case` files.
- TypeScript `strict: true`. No `any` without a comment on the same line saying why.
- Design tokens live in `tokens.css` and are mapped onto Angular Material's `--mat-sys-*`
  system variables. Never inline a colour, radius or spacing value in a component.
- Every network state is visible: loading skeleton, empty state, error state. The API is on a
  free tier that sleeps — a 60 s first load must look deliberate, not broken.

## Testing — honest tests only
The point of a test here is to catch a real regression and to be defensible in an interview.
A test that only proves the mocking framework works is worse than no test: it costs maintenance
and it will be picked apart the moment someone reads it.

- **Before writing a test, finish this sentence: "this test fails if ______."** If the blank
  can't be filled with a behaviour someone would notice, don't write the test.
- **Banned:** `verify(repository).save(any())` as the only assertion. Mocking the repository to
  check that the service called the repository. `assertNotNull(result)` as a whole test.
  Test names like `test1`, `shouldWork`, `happyPath`.
- **Mockito is allowed only where a decision is made in Java** — the status-transition state
  machine, the duplicate rule, entity↔DTO mapping. Pass-through CRUD gets no unit test; it is
  covered by the integration test, where it is actually exercised.
- **Integration tests assert HTTP status + specific body fields + database state.** Not "no
  exception was thrown".
- Sanity check, once per test class: delete the production line the test claims to cover and
  confirm the test goes red. If it stays green, the test is decoration.
- `XxxControllerIT` — `@SpringBootTest` + Testcontainers with **real PostgreSQL 17**, never H2.
  One reusable container per run.
- **Max 8 tests per class.** Happy path, missing field, invalid value, one edge case.
  Never happy-path-only. Names describe the scenario:
  `returns409WhenMovingFromRejectedToOffer`.
- Planned test classes for the whole project — exactly two:
  `ApplicationServiceTest` (transitions, duplicates) and `ApplicationControllerIT` (the API).
- Every bug fixed gets a test that fails before the fix.
- On Java 25, Mockito's dynamic agent loading emits a warning (JEP 451). Fix it once, in the
  surefire config, with `-XX:+EnableDynamicAgentLoading` — and know why the JVM complains:
  loading an agent into a running VM is being locked down. Do not silence it by downgrading Java.

## Git & workflow
- Conventional Commits: `feat(applications): add status transition endpoint`.
- One branch per feature: `feat/003-crud-board`. PR into `main`, CI green before merge.
- **Ask before every `git push`.** Never `--force`. Never commit generated build output.

## Comments — this is the deliverable, not decoration
- English. Explain **why**, not what.
- **The first time an annotation appears in the codebase, comment what it does and what it
  replaces.** (`@Transactional`, `@Valid`, `@ServiceConnection`, `@RestControllerAdvice`, …)
- If a decision had a plausible alternative, name the alternative and why it lost.

## Definition of done (a feature is not done until all six)
1. Code + honest tests pass locally and in CI.
2. `docs/study/learning-map.md` updated: every new tool/annotation/concept, why over the
   alternative, which file to read.
3. `docs/study/interview-cheatsheet.md` updated: the questions this feature invites, answered
   in 30–60 seconds of speech, with likely follow-ups.
4. Deployed, and **opened on a real phone**, not just a narrow browser window.
5. No horizontal scroll at 360 px; every action reachable by keyboard.
6. The author can explain the feature out loud with the laptop closed.

## Commands
```
docker compose up -d              # local Postgres 17
./mvnw spring-boot:test-run       # API on :8080 with the compose DB
./mvnw verify                     # unit + integration tests (needs the Docker daemon)
npm start          # (in web/) Angular dev server on :4200
npm run build
```
All of the above run inside WSL. There is no global `mvn` and no `docker-compose` (v1) here.

## Traps to refuse
- Adding a table, an endpoint, a screen, or a library "while we're here".
- JWT, refresh tokens, multi-user, roles, registration. Out of scope, on purpose.
- Lombok, MapStruct, ModelMapper, a mapper library of any kind.
- Drag-and-drop. Desktop-first CSS with a mobile "fix" bolted on afterwards.
- Copying a Spring Boot 3.x pattern from memory or a blog.
- Writing a test to reach a number instead of to catch a regression.
- Writing code before the study-doc entry that explains it.
