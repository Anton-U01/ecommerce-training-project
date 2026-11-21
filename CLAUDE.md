# E-commerce Microservices — Training Project

A **learning project**. The developer wants hands-on practice with Kafka, RabbitMQ, the Spring data layer (JPA + PostgreSQL) and CI/CD. The roadmap and all tickets are in `docs/ROADMAP.md`. The user will say which ticket we are on.

## Starting a ticket ("let's start with ticket T-XX")
When the user starts a ticket, the **first response is a step-by-step implementation guide only**. Do not create, edit or delete files, and do not run commands that change anything. Read-only checks, such as reading files or checking tool versions, are fine.

The guide contains:
1. **Goal**: one or two sentences on what the ticket delivers, taken from `docs/ROADMAP.md`.
2**Steps**: numbered and small, each with *what* to do, *where* (file/path) and *why* (the concept behind it). Name the Spring annotations, classes, config keys and commands involved, but don't write full implementations.
3**How to verify**: the commands or requests with all the main test cases that prove the ticket's "Done when".

End with a question asking which working mode to use. Make changes only after the user picks a mode or explicitly asks for them.

## Working mode (decided per ticket)
The user does most of the work by hand to keep full ownership of the code. At the start of each ticket, they say which mode to use:
- **I code, you review:** don't write implementation code unless asked. Give hints, point to docs, review their code, explain the *why*, and ask the ticket's theory questions.
- **Build together / you implement:** implement in small steps and explain the decisions as you go.

If the mode isn't stated, ask before writing code. Answer questions in a focused way, tied to this project's code where possible.

## Scope discipline
- Build only the **MVP** described in the roadmap. Don't add backlog features such as Flyway, pagination, the outbox, Resilience4j or tracing unless the user starts that ticket.
- Keep solutions simple and readable. Readability matters more than cleverness, because this is for learning.

## Stack and decisions
- Java 21, Spring Boot 3.x, **Maven multi-module** with the Maven Wrapper (`./mvnw`)
- **Monorepo, not a monolith:** each service is its own Spring Boot app → its own jar, Docker image, container/K8s Deployment and database. Services communicate only via HTTP, Kafka or RabbitMQ, never through shared Java code. The only exception is `libs/events`, which holds event DTOs (records) only.
- PostgreSQL with one database per service (`product_db`, `order_db`, `inventory_db`, `notification_db`). In the MVP, the schema comes from `spring.jpa.hibernate.ddl-auto=update`.
- Kafka (KRaft mode) for business events between services, with `orderId` as the message key.
- RabbitMQ for background email tasks inside notification-service.
- Mailpit as the local SMTP server.
- Spring Cloud Gateway as the single entry point (`/api/products/**`, `/api/orders/**`, `/api/inventory/**`).
- Tests: JUnit 5 + Mockito for unit tests, `@SpringBootTest` + Testcontainers (`@ServiceConnection`) for integration tests against real Postgres. No H2.
- CI/CD: GitHub Actions on **GitHub-hosted runners only**. The repo is public, so never use self-hosted runners. `ci.yml` builds and tests PRs. `deploy.yml` runs on merge to `main`: it builds images tagged with the git SHA, deploys them to an **ephemeral kind cluster** in the job, runs smoke/E2E tests through the gateway, and pushes the images to GHCR only if the tests pass.
- The repo is **public**. `docs/ROADMAP.md` and `docs/learning/` are git-ignored and must never be committed. Never commit secrets; use GitHub Actions secrets or `GITHUB_TOKEN`.

## Layout
```
pom.xml                  parent / aggregator
libs/events/             shared event records
services/<name>-service/ api-gateway, product, order, inventory, notification
infra/docker-compose.yml local infrastructure (+ services later)
infra/k8s/               Kubernetes manifests (kind)
.github/workflows/       ci.yml, deploy.yml
docs/ROADMAP.md          tickets (local only, git-ignored)
docs/learning/           learning notes, one file per phase (local only, git-ignored)
```

## Code conventions
- Package per service: `com.ecommerce.<service>` with sub-packages `controller`, `service`, `repository`, `entity`, `dto`, `exception` (and `client`, `messaging` where needed).
- DTOs are Java records. Never expose JPA entities from controllers.
- Money is `BigDecimal` and maps to `NUMERIC(10,2)`.
- Enums are stored with `@Enumerated(EnumType.STRING)`.
- Validate input with `@Valid` + Bean Validation. Each service has a `@RestControllerAdvice` for errors.
- Keep remote HTTP calls outside DB transactions.

## Build and run
- Build and test everything: `./mvnw verify`
- One service: `./mvnw -pl services/product-service -am verify`
- Run locally: `./mvnw -pl services/product-service spring-boot:run`
- Infrastructure: `docker compose -f infra/docker-compose.yml up -d`

## Project subagents (`.claude/agents/`)
- `code-reviewer`: reviews the ticket's changes and explains findings. Never edits.
- `tutor`: `questions` mode returns a quiz for a ticket; `grade` mode grades the answers and adds the ticket's section to its phase file in `docs/learning/` (e.g. `phase-1-data-layer.md`). For a quiz, call it in `questions` mode, show the questions to the user, collect the answers, then call it in `grade` mode.
- `debugger`: read-only root-cause investigation (logs, containers, Kafka/Rabbit, kind, CI).

None of them write implementation code. Delegate to them only when the user asks or the situation clearly fits.

## Learning notes
Notes are kept **one file per phase**. The file names are listed in `docs/learning/README.md`. At the end of a ticket, if the user asks, add a section for that ticket to its phase file (create the file if it doesn't exist yet) with what was built, the answers to the theory questions and the gotchas we found. Merge new insights into the file's shared "Key concepts" section rather than repeating them. Never overwrite earlier ticket sections. Follow the template in `docs/learning/README.md`.
