---
name: java-soap-to-microservices-migration-orchestration
description: Master orchestration prompt that chains legacy-modernizer, java-architect, microservices-architect, and playwright-expert skills to execute a complete Java SOAP/WSDL to microservices migration with Oracle database on OpenShift OCP. Use when migrating complex enterprise SOAP systems and need coordinated execution across multiple domain specialists.
---

# Java SOAP to Microservices Migration: Multi-Skill Orchestration

This is the master prompt for executing a complete migration of legacy Java SOAP/WSDL applications to modern Spring Boot 3.x microservices deployable on OpenShift using Oracle databases.

## Prerequisites

- **Legacy WSDL files** or SOAP service documentation available
- **Oracle database** (19c+) with current schema and sample data
- **OpenShift OCP cluster** (4.10+) with admin access
- **Team**: Enterprise architect, Java developers, DevOps engineer, QA automation specialist

## End-to-End Execution Flow

### Stage 1: Assessment & Analysis (Run legacy-modernizer)

**Goal**: Understand the legacy SOAP system, identify service boundaries, and plan the migration roadmap.

**Invoke**: legacy-modernizer

**Specific Tasks**:
1. Analyze WSDL files to extract:
   - All SOAP operations/endpoints
   - Input/output message structures
   - Fault definitions and error handling patterns
   - Service dependencies (external systems, databases)

2. Map Oracle database:
   - Analyze schema (tables, relationships, constraints)
   - Identify data ownership boundaries
   - Document stored procedures and triggers
   - Assess current data volumes and growth rates

3. Identify service boundaries using Domain-Driven Design:
   - Define bounded contexts (e.g., User Management, Order Processing, Inventory)
   - Map SOAP operations to bounded contexts
   - Identify data dependencies between services
   - Produce dependency graph

4. Assess migration risks:
   - Identify backward compatibility requirements
   - Document external integrations (APIs, message queues, file systems)
   - List regulatory or compliance constraints
   - Quantify schema complexity and data migration effort

5. Output deliverables:
   - **legacy-analysis.md**: Comprehensive SOAP/WSDL documentation
   - **services-boundary-map.md**: DDD bounded contexts and service assignments
   - **dependency-graph.png**: Visual representation of service and data dependencies
   - **migration-roadmap.md**: Phased approach with timeline, risks, and rollback triggers
   - **oracle-schema-decomposition.md**: Plan for breaking monolithic schema into service-owned databases

**Validation checkpoint**: All legacy analyses documented; architecture approved by stakeholders before proceeding to Phase 2.

---

### Stage 2: Architecture & Design (Run microservices-architect)

**Goal**: Design the target microservice architecture, communication patterns, and deployment topology on OpenShift.

**Invoke**: microservices-architect

**Specific Tasks**:
1. Design microservice architecture:
   - Define service responsibilities and boundaries
   - Specify inter-service communication (REST/gRPC/async)
   - Design API gateway layer for legacy client compatibility
   - Plan data synchronization and eventual consistency strategies
   - Design circuit breaker patterns for resilience

2. Create OpenShift deployment topology:
   - Define Kubernetes namespaces (dev, staging, prod)
   - Specify service mesh (Istio) for advanced routing and observability
   - Plan resource requests/limits per service
   - Design ingress strategy (OpenShift Route vs Ingress)
   - Plan secret management (Sealed Secrets, HashiCorp Vault)

3. Design strangler fig pattern:
   - API Gateway routes SOAP calls to new REST services
   - Feature flags for traffic shifting
   - Request/response translation layer
   - Fallback to legacy SOAP if needed

4. Define observability stack:
   - Distributed tracing (OpenTelemetry + Jaeger)
   - Centralized logging (ELK/Datadog)
   - Metrics collection (Prometheus + Grafana)
   - Health checks (liveness, readiness probes)

5. Output deliverables:
   - **services-architecture.md**: Service interactions, contracts, and patterns
   - **openshift-topology.yaml**: Kubernetes resource templates (namespaces, quotas, RBAC)
   - **api-gateway-design.md**: Gateway routing rules, traffic policies, fallback mechanisms
   - **data-synchronization-strategy.md**: Event sourcing, CDC, saga patterns
   - **observability-stack.md**: Logging, tracing, metrics configuration

**Validation checkpoint**: Architecture diagram approved; OpenShift cluster provisioned with namespaces and quotas; observability stack deployed.

---

### Stage 3: Implementation (Run java-architect x 1 per service)

**Goal**: Implement Spring Boot 3.x microservices with Java 21, REST endpoints, and Oracle database integration.

**Invoke**: java-architect (once per microservice)

**High-level sequence**:

#### 3.1 Create Shared Library
- DTOs for inter-service communication
- Common exception hierarchies
- Spring Security configurations and JWT validators
- Database connection pooling (HikariCP)

#### 3.2 For each microservice (e.g., user-service, order-service):

**Specific Tasks**:
1. Initialize Spring Boot 3.x project:
   - Create Maven project with Kotlin/Java 21 configuration
   - Add starters (Web, Data JPA, Security, Actuator)
   - Configure application.yml with Oracle connection (HikariCP)
   - Set up Flyway for schema migrations

2. Design domain models:
   - Create JPA entities using Java records or class-based models
   - Define relationships (OneToMany, ManyToMany, etc.)
   - Add constraint validation (Jakarta Bean Validation)
   - Ensure fetch strategies prevent N+1 queries

3. Implement data access layer:
   - Create Spring Data repositories with custom queries
   - Implement query optimization (projections, batch fetching)
   - Add transaction boundaries (@Transactional)
   - Use TestContainers for integration tests

4. Implement business logic:
   - Create service classes with clear responsibilities
   - Handle cross-cutting concerns (observability, security)
   - Implement idempotency patterns for critical operations
   - Add domain events for eventual consistency

5. Create REST API endpoints:
   - Convert legacy SOAP operations to REST endpoints (GET/POST/PUT/DELETE)
   - Document with OpenAPI/Swagger annotations
   - Implement input validation and error handling
   - Add pagination, filtering, sorting for list endpoints

6. Configure Spring Security:
   - JWT token validation via Spring Security filters
   - Role-based access control (@PreAuthorize)
   - CORS configuration for cross-service requests
   - API key validation if needed

7. Implement observability:
   - Add OpenTelemetry instrumentation
   - Structured logging with SLF4J/Logback (JSON format)
   - Micrometer metrics for custom business metrics
   - Health check endpoints (liveness, readiness)

8. Create Dockerfile:
   - Multi-stage build (compile stage, runtime stage)
   - Use eclipse-temurin:21-jre as base image
   - Set resource limits and JVM options
   - Expose health check endpoint

9. Output deliverables (per service):
   - **pom.xml**: Maven configuration with all dependencies
   - **src/main/java/**: Domain entities, repositories, services, controllers
   - **src/test/java/**: Unit and integration tests (80%+ coverage)
   - **src/main/resources/db/migration/V*.sql**: Flyway schema migrations
   - **src/main/resources/application.yml**: Service configuration
   - **Dockerfile**: Container image definition
   - **helm-chart/**: Kubernetes deployment manifests

**Validation checkpoint**: All services deploy to Docker locally; integration tests pass with TestContainers Oracle; endpoints respond correctly.

---

### Stage 4: Deploy API Gateway & Strangler Facade

**Goal**: Create the API Gateway that routes legacy SOAP traffic to new microservices with gradual traffic shifting.

**Invoke**: java-architect (once for API Gateway service)

**Specific Tasks**:
1. Create Spring Cloud Gateway service:
   - Add spring-cloud-gateway dependency
   - Configure routes from legacy SOAP paths to new REST endpoints
   - Implement request/response transformation
   - Add circuit breaker support

2. Implement strangler fig pattern:
   - Feature flags for traffic percentage shifting
   - Fallback routing to legacy SOAP if new service fails
   - Request/response translation (SOAP ↔ REST)
   - Logging and tracing of all requests

3. Security at gateway:
   - JWT token validation before routing to services
   - Rate limiting and DDoS protection
   - CORS configuration

4. Deployment:
   - Containerize gateway service
   - Deploy to OpenShift via Helm chart
   - Monitor gateway performance and errors

5. Output deliverables:
   - **ApiGatewayController.java**: Route definitions and translation logic
   - **FeatureFlagService.java**: Traffic shifting configuration
   - **GatewayConfig.java**: Spring Cloud Gateway configuration
   - **gateway-helm-chart/**: Kubernetes deployment

**Validation checkpoint**: Gateway routes 5% traffic to new services successfully; error rates within baseline.

---

### Stage 5: Orchestrate Incremental Migration

**Goal**: Shift traffic from legacy SOAP to new microservices in phases (5% → 25% → 50% → 100%).

**Tasks**:
1. Prepare rollback procedures:
   - Document health check dashboards
   - Create runbooks for instant fallback to legacy
   - Test rollback procedure in staging

2. Execute Phase 1 (5% traffic):
   - Enable feature flag for 5% of requests to new services
   - Monitor dashboards for errors, latency, database queries
   - Review logs for unexpected behavior
   - **Validation**: Error rate < baseline, latency within SLA

3. Execute Phase 2 (25% traffic):
   - Increase feature flag percentage
   - Monitor for 24-48 hours
   - Validate data consistency
   - **Validation**: All metrics green, no data discrepancies

4. Execute Phase 3 (50% traffic):
   - Continue traffic increase
   - Conduct canary release if possible
   - Monitor production dashboards continuously
   - **Validation**: New service handles 50% load without degradation

5. Execute Phase 4 (100% traffic):
   - Route all traffic to new services
   - Keep legacy SOAP running for 1-2 weeks as safety net
   - Monitor comprehensive metrics
   - **Validation**: 100% traffic, all SLAs met, zero critical errors

6. Output deliverables:
   - **traffic-shifting-runbook.md**: Phase-by-phase procedures
   - **monitoring-dashboard.json**: Grafana dashboard for migration metrics
   - **rollback-procedures.md**: Emergency procedures

**Validation checkpoint**: All phases successful; legacy can be retired with documented sunset date.

---

### Stage 6: End-to-End Testing (Run playwright-expert)

**Goal**: Create comprehensive E2E test suite with Playwright to validate all user journeys and API integrations.

**Invoke**: playwright-expert

**Specific Tasks**:
1. Set up Playwright test infrastructure:
   - Initialize TypeScript-based Playwright project
   - Configure browsers (Chromium, Firefox, WebKit)
   - Set up test fixtures for authentication and database state
   - Create custom reporters for HTML/JUnit output

2. Create Page Object Models:
   - **ApiGatewayPage**: Methods for gateway interactions
   - **UserServicePage**: User CRUD operations
   - **OrderServicePage**: Order management operations
   - **AuthenticationPage**: Login/token generation

3. Build database fixtures:
   - Insert test data directly into Oracle
   - Clean up after each test run
   - Support for data-driven test scenarios

4. Write E2E test scenarios:
   - **User creation and updates**: Create user → Edit profile → Verify database → Delete
   - **Order workflow**: Create order → Add items → Submit → Verify inventory → Confirm shipment
   - **Cross-service integration**: User places order → Triggers payment → Inventory updated
   - **Error scenarios**: Invalid input → Proper error response → No database corruption
   - **Performance**: API response times < 500ms under normal load
   - **Backward compatibility**: Legacy SOAP clients still work during migration

5. Create performance tests:
   - Load test: 1000 concurrent users
   - Stress test: Ramp up until failure
   - Endurance test: Run for 24 hours
   - Spike test: Sudden traffic increase

6. Implement CI/CD integration:
   - GitHub Actions or GitLab CI pipeline
   - Run tests on every merge to main
   - Generate HTML reports in artifacts
   - Slack notifications on failures

7. Output deliverables:
   - **e2e-tests/tests/**: All Playwright test files
   - **e2e-tests/pages/**: Page Object Models
   - **e2e-tests/fixtures/**: Auth and database fixtures
   - **e2e-tests/playwright.config.ts**: Test configuration
   - **e2e-tests/README.md**: Test execution instructions
   - **playwright-report/**: HTML test report
   - **.github/workflows/e2e-tests.yml**: CI/CD pipeline

**Validation checkpoint**: All E2E tests pass; performance baselines established; CI/CD pipeline green on every commit.

---

## Complete Migration Checklist

### Pre-Migration
- [ ] WSDL analysis complete (legacy-modernizer output)
- [ ] Service boundaries defined (microservices-architect output)
- [ ] OpenShift cluster provisioned and namespaces created
- [ ] Oracle database backups taken
- [ ] Team trained on Spring Boot 3.x and microservices patterns

### Development
- [ ] All microservices implemented with 80%+ test coverage (java-architect output)
- [ ] Shared libraries and DTOs created
- [ ] API Gateway implemented with strangler fig pattern
- [ ] Database migrations tested on dev Oracle instance
- [ ] Docker images built and pushed to registry

### Testing
- [ ] Unit tests pass (85%+ coverage)
- [ ] Integration tests pass with TestContainers
- [ ] E2E tests pass with Playwright (5+ critical user journeys)
- [ ] Performance tests meet latency SLAs
- [ ] Load tests handle expected peak traffic

### Staging
- [ ] Helm charts deploy successfully to staging OCP cluster
- [ ] All services communicate correctly (service-to-service, with gateway)
- [ ] Database migrations run successfully (idempotent)
- [ ] Observability stack (logging, tracing, metrics) operational
- [ ] Health probes working correctly

### Production Cutover
- [ ] Production OCP cluster ready with capacity planning
- [ ] Database replicated to production Oracle
- [ ] Secrets (JWT keys, database passwords) secured in Sealed Secrets
- [ ] Monitoring dashboards created
- [ ] Rollback runbook tested in staging

### Migration Phases
- [ ] Phase 1 (5% traffic) stable for 24 hours
- [ ] Phase 2 (25% traffic) stable for 24 hours
- [ ] Phase 3 (50% traffic) stable for 24 hours
- [ ] Phase 4 (100% traffic) stable for 48 hours
- [ ] Data consistency verified end-to-end
- [ ] Legacy SOAP decommissioned after 2-week safety period

---

## Rollback Triggers

If any of the following occur, execute immediate rollback to legacy SOAP:
- **Error rate** exceeds 5% (vs. < 1% baseline)
- **Latency (p99)** exceeds 2 seconds (vs. < 500ms baseline)
- **Database corruption** detected (referential integrity violations, missing records)
- **Data loss** in migration process
- **Security incident** (unauthorized access, token validation failures)
- **Critical business logic failure** (transactions not completed, orders not processed)

Rollback procedure:
1. Update feature flag to 0% new traffic
2. Monitor legacy service error rates for recovery
3. Investigate root cause in staging environment
4. Fix and re-test before re-attempting migration
5. Notify stakeholders of rollback and recovery plan

---

## Success Metrics (Target)

| Metric | Target | Baseline |
|--------|--------|----------|
| Error rate (4xx+5xx) | < 0.5% | < 1% |
| Latency (p50) | < 200ms | < 300ms |
| Latency (p99) | < 500ms | < 800ms |
| Database query latency | < 50ms | < 100ms |
| Service startup time | < 30s | N/A |
| Pod memory usage | < 512MB | N/A |
| Pod CPU usage | < 2 cores | N/A |
| E2E test coverage | > 80% critical paths | N/A |
| Code coverage | > 85% | N/A |
| Deployments per week | > 5 | N/A (monolith doesn't deploy easily) |
| Mean time to recovery (MTTR) | < 10 minutes | N/A |

---

## Next Steps

1. **Review this prompt** with your team and confirm alignment
2. **Invoke legacy-modernizer** to analyze your legacy SOAP system
3. **Invoke microservices-architect** to design target architecture
4. **Invoke java-architect** for each microservice implementation
5. **Invoke playwright-expert** to create end-to-end test suite
6. **Execute migration phases** with continuous monitoring
7. **Retire legacy SOAP** after 2-week safety period

---

## Support & Documentation

- Review the **java-soap-to-microservices** SKILL.md for comprehensive guidance
- Use **templates/** directory for boilerplate code and configurations
- Consult **references/** for detailed guides on specific migration topics
- Schedule weekly architecture reviews during migration phases
- Maintain a **migration war room** for incident response and escalation
