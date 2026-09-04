# Contributing to Scalable E-Commerce Microservices

First off, thanks for taking the time to contribute — this project spans multiple services and a fair amount of infrastructure, so this guide exists to help you get productive quickly and keep the codebase consistent.

## Table of Contents

- [Code of Conduct](#code-of-conduct)
- [Getting Started](#getting-started)
- [Development Workflow](#development-workflow)
- [Branching & Commit Conventions](#branching--commit-conventions)
- [Coding Standards](#coding-standards)
- [Testing Requirements](#testing-requirements)
- [Submitting a Pull Request](#submitting-a-pull-request)
- [Reporting Bugs](#reporting-bugs)
- [Suggesting Features](#suggesting-features)
- [Service-Specific Notes](#service-specific-notes)

---

## Code of Conduct

This project follows a standard [Code of Conduct](CODE_OF_CONDUCT.md). By participating, you're expected to uphold it — be respectful, assume good intent, and keep feedback constructive.

---

## Getting Started

1. **Fork** the repository and clone your fork:

   ```bash
   git clone https://github.com/<your-username>/scalable-ecommerce-microservices.git
   cd scalable-ecommerce-microservices
   ```

2. **Add the upstream remote** so you can keep your fork in sync:

   ```bash
   git remote add upstream https://github.com/your-org/scalable-ecommerce-microservices.git
   ```

3. Follow the [Local Setup & Quickstart](README.md#-local-setup--quickstart) section in the README to get the full stack running via Docker Compose.

4. Install dependencies for the service(s) you're working on:

   ```bash
   cd services/<service-name>
   npm install
   ```

---

## Development Workflow

1. Sync your fork before starting new work:

   ```bash
   git checkout main
   git pull upstream main
   ```

2. Create a feature branch off `main` (see naming convention below).

3. Make your changes, keeping commits focused and scoped to a single logical change.

4. Run linting and tests locally **before** pushing (see [Testing Requirements](#testing-requirements)).

5. Push your branch and open a pull request against `main`.

---

## Branching & Commit Conventions

### Branch naming

| Prefix | Use case | Example |
|---|---|---|
| `feature/` | New functionality | `feature/order-refund-flow` |
| `fix/` | Bug fixes | `fix/cart-ttl-expiry` |
| `chore/` | Tooling, deps, config | `chore/upgrade-node-20` |
| `docs/` | Documentation only | `docs/update-api-gateway-readme` |
| `refactor/` | Non-behavioral code changes | `refactor/extract-payment-adapter` |

### Commit messages

This project follows [Conventional Commits](https://www.conventionalcommits.org/):

```
<type>(<scope>): <short summary>

[optional body]

[optional footer(s)]
```

**Examples:**

```
feat(order-service): add idempotency key validation to POST /orders
fix(cart-service): correct TTL calculation on guest cart merge
docs(readme): clarify RabbitMQ env var requirements
test(catalog-service): add integration tests for search filters
```

Common types: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`, `perf`, `ci`.

---

## Coding Standards

- **Linting:** Run `npm run lint` before committing. CI will reject PRs with lint errors.
- **Formatting:** Prettier config is enforced repo-wide — run `npm run format` to auto-fix.
- **Style conventions:**
  - Use `async/await` over raw Promise chains.
  - No business logic in route handlers — keep it in service/controller layers.
  - New public-facing resource IDs must use **UUIDv7**, not auto-increment integers (see README's [Architectural Decisions](README.md#-architectural--security-decisions)).
  - Side effects that don't need to block the request/response cycle (emails, analytics, non-critical writes) should be published as events to RabbitMQ, not called synchronously.
  - Never commit secrets, API keys, or `.env` files — use `.env.example` to document new required variables.
- **Per-service isolation:** Do not add cross-service database access. Services communicate only via their public REST APIs or the event bus.

---

## Testing Requirements

Every PR that changes behavior must include or update tests. At minimum:

```bash
# Unit tests for the service you touched
npm run test:unit

# Integration tests if you touched data access, queues, or cross-service contracts
npm run test:integration
```

For larger changes touching multiple services, run the full suite before opening a PR:

```bash
npm run test:ci
```

PRs with failing tests or a drop in coverage will not be merged. See [Testing & QA](README.md#-testing--qa) in the README for the complete command reference.

---

## Submitting a Pull Request

1. Ensure your branch is rebased on the latest `main`:

   ```bash
   git fetch upstream
   git rebase upstream/main
   ```

2. Push and open a PR with:
   - A clear title following the Conventional Commits format
   - A description of **what** changed and **why**
   - Linked issue(s), if applicable (`Closes #123`)
   - Screenshots or sample requests/responses for API-facing changes

3. Fill out the PR template checklist (tests added, docs updated, lint passing).

4. Address review feedback with additional commits — please don't force-push over a review in progress, since it makes re-reviewing harder. Squash before merge is handled by the maintainers.

5. A maintainer will merge once at least one approval is given and CI is green.

---

## Reporting Bugs

Open an issue using the **Bug Report** template and include:

- Steps to reproduce
- Expected vs. actual behavior
- Service(s) affected (e.g., `order-service`, `api-gateway`)
- Relevant logs (redact any secrets/PII) — Kibana output is helpful if available
- Environment details (OS, Docker version, Node version)

---

## Suggesting Features

Open an issue using the **Feature Request** template and include:

- The problem the feature solves (not just the solution)
- Which service(s) or layer it would affect
- Any relevant prior art, trade-offs, or alternatives considered

For larger architectural proposals (new services, new infrastructure dependencies), please open a discussion thread first so the approach can be validated before implementation work begins.

---

## Service-Specific Notes

| Service | Notes for contributors |
|---|---|
| `auth-service` | Changes to token expiry or refresh logic require security review before merge |
| `catalog-service` | Cache invalidation logic is sensitive — see the cache-aside strategy in the README before modifying |
| `cart-service` | TTL and guest-merge logic have edge cases covered by integration tests — don't remove without replacing coverage |
| `order-service` | All write endpoints must remain idempotent — do not remove the `Idempotency-Key` requirement |
| `notification-service` | Should never be called synchronously from another service — consume events only |

---

Thanks again for contributing — every fix, test, and doc improvement helps keep this project maintainable as it grows.
