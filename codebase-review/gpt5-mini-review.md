# API Foundation Technical Review — apps/api

Generated on 2025-08-11 by Roo (automated technical review).

## 1. Executive Summary

- Verdict: The API repository implements a strong, production-oriented foundation — modular NestJS architecture, Zod-validated config, telemetry, structured error handling, JWT + API-key auth, caching, rate‑limit hooks, and a repository DAL. It is close to being a reliable baseline for future feature work, but a few important correctness, consistency, and hardening fixes (mostly small, high‑impact) are needed before declaring it production‑ready without further foundational changes.

## 2. Scorecard

| Category | Score (0–5) | Key Observations |
|---|---:|---|
| Architecture & Modularity | 4 | App is modular and feature-driven. See [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:56). Global interceptors/guards wired: [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:139). Note: DI anti-pattern in JWT auth: [`apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts`](apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts:38). |
| Persistence & Data Layer | 4 | TypeORM with pool and SnakeNamingStrategy: [`apps/api/src/config/database.config.ts`](apps/api/src/config/database.config.ts:26). BaseRepository with transactions and pagination: [`apps/api/src/lib/database/base.repository.ts`](apps/api/src/lib/database/base.repository.ts:55). Migrations in package.json: [`apps/api/package.json`](apps/api/package.json:23). |
| Security | 4 | Zod env validation: [`apps/api/src/config/env.validation.ts`](apps/api/src/config/env.validation.ts:8). ValidationPipe and sanitization: [`apps/api/src/common/pipes/validation.pipe.ts`](apps/api/src/common/pipes/validation.pipe.ts:22). Global exception filter: [`apps/api/src/common/filters/global-exception.filter.ts`](apps/api/src/common/filters/global-exception.filter.ts:88). JWKS + JWT validator: [`apps/api/src/lib/auth/authentication/jwt/jwks-client.ts`](apps/api/src/lib/auth/authentication/jwt/jwks-client.ts:50). Helmet CSP currently allows unsafe directives: [`apps/api/src/main.ts`](apps/api/src/main.ts:151). |
| Rate Limiting & Abuse Prevention | 3 | Per-endpoint decorator, guard and interceptor exist: [`apps/api/src/common/decorators/rate-limit.decorator.ts`](apps/api/src/common/decorators/rate-limit.decorator.ts:60), [`apps/api/src/common/guards/rate-limit.guard.ts`](apps/api/src/common/guards/rate-limit.guard.ts:136). Implementation uses read‑modify‑write (cache.get + set) and needs atomic Redis INCR/LUA for safe distributed operation. |
| Performance & Scalability | 4 | DB pooling and TypeORM config: [`apps/api/src/config/database.config.ts`](apps/api/src/config/database.config.ts:44). Cache service with TTL strategies and cache-aside helpers: [`apps/api/src/lib/cache/cache.service.ts`](apps/api/src/lib/cache/cache.service.ts:98). Telemetry initialized early: [`apps/api/src/main.ts`](apps/api/src/main.ts:83). |
| Error Handling & Logging | 4 | GlobalExceptionFilter provides consistent error shape: [`apps/api/src/common/filters/global-exception.filter.ts`](apps/api/src/common/filters/global-exception.filter.ts:88). LoggingInterceptor and correlation middleware are present: [`apps/api/src/common/interceptors/logging.interceptor.ts`](apps/api/src/common/interceptors/logging.interceptor.ts:34) and [`apps/api/src/common/middleware/correlation-id.middleware.ts`](apps/api/src/common/middleware/correlation-id.middleware.ts:31). |
| Testing & CI/CD | 3 | Vitest is configured (scripts): [`apps/api/package.json`](apps/api/package.json:17). Test scaffolding exists under `test/` but there is no evidence of CI gating enforcing coverage in the reviewed snapshot. |
| Documentation & DevEx | 4 | Swagger/OpenAPI is thorough and wired in dev: [`apps/api/src/config/swagger.config.ts`](apps/api/src/config/swagger.config.ts:7). Zod environment schema documents required envs programmatically: [`apps/api/src/config/env.validation.ts`](apps/api/src/config/env.validation.ts:8). |
| Maintainability & Future-Proofing | 4 | Feature-based layout and repository pattern enable predictable extension: [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:56), [`apps/api/src/lib/database/base.repository.ts`](apps/api/src/lib/database/base.repository.ts:55). Address DI anti-pattern and cache-store portability for improvement. |

## 3. Strengths

- Modular infrastructure and clear feature separation: see [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:56).
- Strong config validation with Zod: [`apps/api/src/config/env.validation.ts`](apps/api/src/config/env.validation.ts:8).
- Robust JWT/JWKS pipeline with caching and rotation: [`apps/api/src/lib/auth/authentication/jwt/jwks-client.ts`](apps/api/src/lib/auth/authentication/jwt/jwks-client.ts:50) and [`apps/api/src/lib/auth/authentication/jwt/jwt-validator.ts`](apps/api/src/lib/auth/authentication/jwt/jwt-validator.ts:45).
- Observability-first approach: OpenTelemetry initialization and TelemetryModule: [`apps/api/src/main.ts`](apps/api/src/main.ts:83) and [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:116).
- Centralized error handling and sanitization: ValidationPipe and GlobalExceptionFilter: [`apps/api/src/common/pipes/validation.pipe.ts`](apps/api/src/common/pipes/validation.pipe.ts:22), [`apps/api/src/common/filters/global-exception.filter.ts`](apps/api/src/common/filters/global-exception.filter.ts:88).

## 4. Gaps & Risks

1. DI anti-pattern in JwtAuthService:
   - Location: [`apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts`](apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts:38).
   - Impact: Harder to mock/replace, invalidates DI lifecycle and testing. Replace manual `new` with DI-provided instances.

2. Non-atomic rate-limiter implementation:
   - Location: [`apps/api/src/common/guards/rate-limit.guard.ts`](apps/api/src/common/guards/rate-limit.guard.ts:136) and [`apps/api/src/common/interceptors/rate-limit.interceptor.ts`](apps/api/src/common/interceptors/rate-limit.interceptor.ts:137).
   - Impact: Race conditions in distributed deployments; attackers could bypass limits. Use Redis atomic ops or Lua.

3. CSP/helmet overly permissive for production:
   - Location: [`apps/api/src/main.ts`](apps/api/src/main.ts:151).
   - Impact: Increased XSS exposure if Swagger/UI are enabled in production. Make unsafe directives conditional.

4. Cache TTL and store portability issues:
   - Location: [`apps/api/src/lib/cache/cache.service.ts`](apps/api/src/lib/cache/cache.service.ts:44) and [`apps/api/src/lib/cache/cache.config.service.ts`](apps/api/src/lib/cache/cache.config.service.ts:11).
   - Impact: TTL semantics differ across stores; `cacheManager.set(..., ttl * 1000)` may be incorrect for the configured store. `clearNamespace` uses `store.keys` which may not be portable for large datasets.

5. Tests/CI enforcement gap:
   - Location: package scripts: [`apps/api/package.json`](apps/api/package.json:8).
   - Impact: Risk of regressions without CI coverage & test gating.

## 5. Recommendations

Short-term fixes (quick wins — days)

- Use DI for `JwksClient` and `JwtValidator`:
  - Update [`apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts`](apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts:38) constructor to accept injected providers; remove `new` calls.

- Make DB SSL handling consistent:
  - Update to use boolean from ConfigService: change `ssl: this.configService.get<string>('DATABASE_SSL') === 'true'` to `ssl: this.configService.get<boolean>('DATABASE_SSL')` in [`apps/api/src/config/database.config.ts`](apps/api/src/config/database.config.ts:34).

- Harden CSP in bootstrap:
  - Apply relaxed CSP only when Swagger is enabled (non-production). Update [`apps/api/src/main.ts`](apps/api/src/main.ts:151).

- Fix cache TTL usage:
  - Use store-agnostic TTL (e.g., pass TTL in seconds using the cache-manager API appropriate for your store) and confirm units. See [`apps/api/src/lib/cache/cache.service.ts`](apps/api/src/lib/cache/cache.service.ts:54).

- Make rate-limiting atomic:
  - Replace read‑modify‑write with Redis INCR + EXPIRE or a Lua sliding-window script. Update logic in [`apps/api/src/common/guards/rate-limit.guard.ts`](apps/api/src/common/guards/rate-limit.guard.ts:136).

- Add CI workflow:
  - Ensure `pnpm lint` and `pnpm test:cov` run in CI with coverage gating. Configure GitHub Actions to fail the build on insufficient coverage.

Long-term improvements (strategic)

- Integrate `nest-winston` and structured JSON logging:
  - Wire Winston early in bootstrap and route logs to ELK/Datadog. Winston is already present in deps: [`apps/api/package.json`](apps/api/package.json:78).

- Implement robust distributed rate limiting & abuse detection:
  - Use Redis atomic algorithms, add Prometheus metrics for rate-limit events, and create alerting rules.

- Align cache invalidation with SCAN / key tagging:
  - Use SCAN with batching for large keyspaces or maintain sets of keys for namespaces; use pipelines for bulk deletes.

- Expand tests for critical integration paths:
  - Add integration tests for JWKS/JWT flows (mocked), rate-limiter concurrency, DB migrations, and cache invalidation. Enforce through CI.

- Create ADRs and runbooks:
  - Document decisions about auth patterns, CSP choices, rate-limiting algorithm, and secret rotation. Add incident runbooks for auth and cache outages.

## 6. Implementation examples (snippets)

- DI fix for JwtAuthService (high-level):

  - Before (problem): see [`apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts`](apps/api/src/lib/auth/authentication/jwt/jwt-auth.service.ts:38)

  - After (sketch): inject `JwksClient` and `JwtValidator` via constructor and remove `new` creation.

- Atomic rate-limiter idea (Redis INCR + EXPIRE):

  - Replace the read/modify/set pattern in [`apps/api/src/common/guards/rate-limit.guard.ts`](apps/api/src/common/guards/rate-limit.guard.ts:136) with an atomic Redis operation (INCR + EXPIRE) or a Lua script to implement a sliding window or token bucket.

## 7. Action plan (priority)

1. Fix DI in JwtAuthService, fix DB SSL flag handling, and harden CSP — (1–3 days)  
2. Replace rate-limiter algorithm with atomic Redis ops and add metrics — (3–5 days)  
3. Fix cache TTL semantics and invalidation approach — (2–4 days)  
4. Add CI workflows with coverage gating and integrate Winston — (2–4 days)  
5. Expand integration tests and create ADRs/runbooks — (ongoing)

---

### Evidence index (quick links)

- App bootstrap & Helmet / Swagger / Telemetry: [`apps/api/src/main.ts`](apps/api/src/main.ts:83), [`apps/api/src/main.ts`](apps/api/src/main.ts:151), [`apps/api/src/main.ts`](apps/api/src/main.ts:203)  
- Root module & modular wiring: [`apps/api/src/app.module.ts`](apps/api/src/app.module.ts:56)  
- Env validation (Zod): [`apps/api/src/config/env.validation.ts`](apps/api/src/config/env.validation.ts:8)  
- TypeORM configuration and pool settings: [`apps/api/src/config/database.config.ts`](apps/api/src/config/database.config.ts:26)  
- Swagger configuration: [`apps/api/src/config/swagger.config.ts`](apps/api/src/config/swagger.config.ts:7)  
- Global exception filter: [`apps/api/src/common/filters/global-exception.filter.ts`](apps/api/src/common/filters/global-exception.filter.ts:88)  
- Validation pipe: [`apps/api/src/common/pipes/validation.pipe.ts`](apps/api/src/common/pipes/validation.pipe.ts:22)  
- Logging & correlation ID middleware: [`apps/api/src/common/interceptors/logging.interceptor.ts`](apps/api/src/common/interceptors/logging.interceptor.ts:34), [`apps/api/src/common/middleware/correlation-id.middleware.ts`](apps/api/src/common/middleware/correlation-id.middleware.ts:31)  
- Auth wiring and providers: [`apps/api/src/lib/auth/auth.module.ts`](apps/api/src/lib/auth/auth.module.ts:39)  
- JWKS client & caching: [`apps/api/src/lib/auth/authentication/jwt/jwks-client.ts`](apps/api/src/lib/auth/authentication/jwt/jwks-client.ts:50)  
- JWT validator: [`apps/api/src/lib/auth/authentication/jwt/jwt-validator.ts`](apps/api/src/lib/auth/authentication/jwt/jwt-validator.ts:45)  
- Guards (JWT / API key / Dual): [`apps/api/src/lib/auth/authentication/guards/jwt-auth.guard.ts`](apps/api/src/lib/auth/authentication/guards/jwt-auth.guard.ts:48), [`apps/api/src/lib/auth/authentication/guards/api-key.guard.ts`](apps/api/src/lib/auth/authentication/guards/api-key.guard.ts:56), [`apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts`](apps/api/src/lib/auth/authentication/guards/dual-auth.guard.ts:31)  
- Rate-limit decorator & guard & interceptor: [`apps/api/src/common/decorators/rate-limit.decorator.ts`](apps/api/src/common/decorators/rate-limit.decorator.ts:60), [`apps/api/src/common/guards/rate-limit.guard.ts`](apps/api/src/common/guards/rate-limit.guard.ts:136), [`apps/api/src/common/interceptors/rate-limit.interceptor.ts`](apps/api/src/common/interceptors/rate-limit.interceptor.ts:137)  
- Cache service & interceptor: [`apps/api/src/lib/cache/cache.service.ts`](apps/api/src/lib/cache/cache.service.ts:98), [`apps/api/src/lib/cache/cache.interceptor.ts`](apps/api/src/lib/cache/cache.interceptor.ts:76)  
- Base repository (DAL): [`apps/api/src/lib/database/base.repository.ts`](apps/api/src/lib/database/base.repository.ts:55)  
- Test runner and package scripts: [`apps/api/package.json`](apps/api/package.json:17)

-- End of report --
