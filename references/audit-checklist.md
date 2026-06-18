# Audit Checklist (expanded)

## Architecture (12 checks)
See section 4.1 in SKILL.md — full grep commands included.

## Production-readiness (10 checks)

| # | Check | Command | Severity |
|---|-------|---------|----------|
| 1 | Actuator dependency | `grep -r "spring-boot-starter-actuator" pom.xml` | MEDIUM |
| 2 | OpenAPI dependency | `grep -r "springdoc\|openapi" pom.xml` | MEDIUM |
| 3 | Health endpoint configured | `grep -r "management.endpoints" src/main/resources/` | MEDIUM |
| 4 | Connection pool configured | `grep -r "hikari\|tomcat-pool" src/main/resources/` | LOW |
| 5 | Flyway migrations present | `ls src/main/resources/db/migration/ 2>/dev/null` | LOW |
| 6 | MDC correlation ID filter | `grep -rn "CorrelationIdFilter\|MDC" src/main/java/` | MEDIUM |
| 7 | Dockerfile exists | `ls Dockerfile 2>/dev/null` | LOW |
| 8 | .dockerignore exists | `ls .dockerignore 2>/dev/null` | LOW |
| 9 | No hardcoded secrets | `grep -rn "password.*=.*[a-zA-Z]\|api.key\|secret" src/main/resources/ --include="*.yml"` | CRITICAL |
| 10 | Error codes in API response | `grep -rn "public static.*error" src/main/java/*/shared/` | LOW |

## Performance (5 checks)

| # | Check | Command | Severity |
|---|-------|---------|----------|
| 1 | N+1 query risk | `grep -rn "@EntityGraph\|JOIN FETCH\|@Query" src/main/java/*/adapter/*/persistence/` | MEDIUM |
| 2 | Missing indexes | `grep -rn "@Table" -A5 src/main/java/*/adapter/*/entity/ \| grep "@Index"` | LOW |
| 3 | Wide transactions | `grep -rn "@Transactional" -B2 src/main/java/*/application/` (check for read-write mixing) | MEDIUM |
| 4 | Synchronous blocking | `grep -rn ".get()\|.join()" src/main/java/*/adapter/*/client/` | LOW |
| 5 | Missing cache | `grep -rn "@Cacheable\|@CacheEvict" src/main/java/` | LOW |

## Security (4 checks)

| # | Check | Command | Severity |
|---|-------|---------|----------|
| 1 | @PreAuthorize on port | `grep -rn "@PreAuthorize" src/main/java/*/application/port/inbound/` | MEDIUM |
| 2 | No @PreAuthorize on controller | `grep -rn "@PreAuthorize" src/main/java/*/adapter/inbound/rest/` | MEDIUM |
| 3 | SecurityFilterChain present | `grep -rn "SecurityFilterChain" src/main/java/` | HIGH |
| 4 | CSRF disabled (for stateless API) | `grep -rn "csrf.*disable" src/main/java/` | LOW |

## Test coverage (3 checks)

| # | Check | Command | Severity |
|---|-------|---------|----------|
| 1 | Use case tests exist | `ls src/test/java/*/application/service/*Test.java 2>/dev/null` | HIGH |
| 2 | Controller tests exist | `ls src/test/java/*/adapter/inbound/rest/*Test.java 2>/dev/null` | MEDIUM |
| 3 | Testcontainers for integration | `grep -rn "Testcontainers\|@Container" src/test/java/` | LOW |
