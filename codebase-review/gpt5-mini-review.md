# API Technical Review — apps/api

## 1. Executive Summary
This repository provides a strong production-ready foundation: clear modular structure, telemetry, caching, queueing, global validation, structured logging, idempotency, and audit-ready error handling. It is not yet safe to treat this as “production-ready baseline” without a small set of fixes and documentation/CI additions (see Gaps & Risks). With the fixes below, future work can be limited to the feature folders under [`apps/api/src/features`](apps/api/src/features:1).

## 2. Scorecard
Category | Score (0–5) | Key Observations
--- | ---: | ---
Architecture & Modularity | 4 | Clear separation (see root module imports: [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:56))
Persistence & Data Layer | 3 | TypeORM/migrations present (`apps/api/typeorm.config.ts:20`,`apps/api/typeorm.config.ts:34`); migrations not auto-run (`apps/api/typeorm.config.ts:38`)
Security | 3 | Multi-method auth (JWT/JWKS + API keys + Dual guard) — see [`apps/api/src/lib/auth/auth.module.ts`](apps/api/src/lib/auth/auth.module.ts:64) and [`apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts`](apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts:41); but there are dev fallbacks (`apps/api/src/lib/auth/auth.module.ts:57`, `apps/api/src/lib/auth/authentication/guards/api-key.guard.ts:190`)
Rate Limiting & Abuse Prevention | 3 | Redis-backed sliding-window guard exists (`apps/api/src/common/guards/rate-limit.guard.ts:22`) but Nest throttler TTL misconfigured (`apps/api/src/app.module.ts:92-99`)
Performance & Scalability | 4 | Redis cache with in-memory fallback (`apps/api/src/lib/cache/cache.module.ts:34`), queue + telemetry integrated (`apps/api/src/lib/telemetry/telemetry.service.ts:44`), DB pooling in DatabaseConfigService (verify)
Error Handling & Logging | 5 | Global exception filter + structured logging + enhanced validation pipe (`apps/api/src/common/filters/global-exception.filter.ts:88`, `apps/api/src/common/interceptors/logging.interceptor.ts:35`, `apps/api/src/common/pipes/validation.pipe.ts:22`)
Testing & CI/CD | 3 | Vitest present (`apps/api/package.json:17`) but no visible CI workflows or enforced coverage gates
Documentation & DevEx | 4 | Swagger setup and env docs; Swagger enabled in non-prod (`apps/api/src/main.ts:204`), env docs present
Maintainability & Future-Proofing | 4 | Consistent modular pattern, Biome linting in package scripts (`apps/api/package.json:16`), but tighten env fail-fast and remove dev fallbacks

## 3. Strengths
- Modular infrastructure vs features separation: root module imports infrastructure modules and features ([`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:56)).
- Robust telemetry and metrics: telemetry initialized at bootstrap ([`apps/api/src/main.ts`](apps/api/src/main.ts:83)) and implemented in [`apps/api/src/lib/telemetry/telemetry.service.ts`](apps/api/src/lib/telemetry/telemetry.service.ts:44).
- Redis-backed caching with in-memory fallback: [`apps/api/src/lib/cache/cache.module.ts`](apps/api/src/lib/cache/cache.module.ts:34).
- Flexible authentication guards and JWKS-based JWT validation: see [`apps/api/src/lib/auth/auth.module.ts`](apps/api/src/lib/auth/auth.module.ts:64) and [`apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts`](apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts:41).
- Production-minded error handling & logging: [`apps/api/src/common/filters/global-exception.filter.ts`](apps/api/src/common/filters/global-exception.filter.ts:88), [`apps/api/src/common/interceptors/logging.interceptor.ts`](apps/api/src/common/interceptors/logging.interceptor.ts:35), [`apps/api/src/common/pipes/validation.pipe.ts`](apps/api/src/common/pipes/validation.pipe.ts:22).

## 4. Gaps & Risks
- Throttler TTL units bug: [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:92-99) sets `ttl: 15 * 60 * 1000` (milliseconds) instead of seconds — this invalidates window semantics.
- Hardcoded/dev fallback secrets: JWT fallback secret present (`apps/api/src/lib/auth/auth.module.ts:57`) and default development API key inserted when none configured (`apps/api/src/lib/auth/authentication/guards/api-key.guard.ts:190`).
- Migrations not auto-run and limited visible pooling config: `migrationsRun: false` in [`apps/api/typeorm.config.ts`](apps/api/typeorm.config.ts:38); verify `DatabaseConfigService` for pooling/timeouts.
- Missing CI enforcement: tests and coverage exist in scripts (`apps/api/package.json:17`,`apps/api/package.json:19`) but no workflow present to gate merges.
- CSP vs Swagger tradeoff: Helmet CSP allows `unsafe-inline`/`unsafe-eval` for Swagger in non-prod (`apps/api/src/main.ts:151`) — must be gated for production.

## 5. Recommendations
Short-term fixes (quick wins)
- Fix throttler TTL units in [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:92): use seconds (e.g., `ttl: 15 * 60`) or pass correct unit expected by library.
- Remove fallback secrets and fail fast:
  - Require `JWT_SECRET` at startup and remove `'fallback-secret'` in [`apps/api/src/lib/auth/auth.module.ts`](apps/api/src/lib/auth/auth.module.ts:57).
  - Remove default dev API key (`'dev-key-12345'`) and fail startup if keys are required (`apps/api/src/lib/auth/authentication/guards/api-key.guard.ts:190`).
- Add Zod-based env validation early in bootstrap (ensure `ConfigModule` validation is strict).
- Add CI workflow to run `pnpm lint`, `pnpm test`, `pnpm test:cov`, and migration checks before merges (scripts in [`apps/api/package.json`](apps/api/package.json:8-29)).
- Move Swagger to a docs subdomain or behind a flag and remove `unsafe-*` CSP directives in production (`apps/api/src/main.ts:151`).

Long-term improvements (strategic)
- Ensure `DatabaseConfigService` exposes explicit pooling/timeouts and instrument DB connection metrics; document PgBouncer recommendations.
- Harden dependency/secret pipeline: enable Dependabot/GitHub CodeQL, SAST scans, and secret scanning in CI.
- Consolidate rate limiting: make Redis sliding-window guard canonical and remove/align Nest `ThrottlerModule` duplication.
- Add integration tests for auth (JWT+API key), idempotency, rate-limiter behavior, and migration safety in CI.
- Enforce minimum coverage and merge gating in CI.

## Appendix — Key files referenced
- Bootstrap / telemetry / helmet / swagger: [`apps/api/src/main.ts`](apps/api/src/main.ts:83), [`apps/api/src/main.ts`](apps/api/src/main.ts:151), [`apps/api/src/main.ts`](apps/api/src/main.ts:204)
- Root module & throttler: [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:56), [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:92)
- Auth & guards: [`apps/api/src/lib/auth/auth.module.ts`](apps/api/src/lib/auth/auth.module.ts:64), [`apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts`](apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts:41), [`apps/api/src/lib/auth/authentication/guards/api-key.guard.ts`](apps/api/src/lib/auth/authentication/guards/api-key.guard.ts:190)
- DB / migrations: [`apps/api/typeorm.config.ts`](apps/api/typeorm.config.ts:20), [`apps/api/typeorm.config.ts`](apps/api/typeorm.config.ts:34), [`apps/api/typeorm.config.ts`](apps/api/typeorm.config.ts:38)
- Cache: [`apps/api/src/lib/cache/cache.module.ts`](apps/api/src/lib/cache/cache.module.ts:34)
- Telemetry service: [`apps/api/src/lib/telemetry/telemetry.service.ts`](apps/api/src/lib/telemetry/telemetry.service.ts:44)
- Error handling & validation: [`apps/api/src/common/filters/global-exception.filter.ts`](apps/api/src/common/filters/global-exception.filter.ts:88), [`apps/api/src/common/interceptors/logging.interceptor.ts`](apps/api/src/common/interceptors/logging.interceptor.ts:35), [`apps/api/src/common/pipes/validation.pipe.ts`](apps/api/src/common/pipes/validation.pipe.ts:22)
- Scripts & test tooling: [`apps/api/package.json`](apps/api/package.json:8-29)

---
Final verdict: This codebase is a high-quality, near-production foundation — implement the short-term fixes (throttler TTL, remove fallback secrets/dev keys, add env fail-fast, add CI) and then treat apps/api as the reusable baseline for future feature work.
