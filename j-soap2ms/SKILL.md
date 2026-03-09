---
name: java-soap-to-microservices-migration
description: "Migrates legacy Java SOAP/WSDL applications with Oracle databases to modern Spring Boot 3.x microservices deployable on OpenShift OCP. Use when modernizing enterprise SOAP services, decomposing monoliths into cloud-native microservices, converting WSDL contracts to REST/gRPC APIs, or upgrading to Java 21 LTS. Produces service boundary maps, API contracts, database migration scripts, Helm charts for OCP deployment, and E2E test suites with Playwright."
metadata:
  version: "1.0.0"
  domain: specialized
  author: arumthi
  triggers: Java SOAP to microservices, WSDL migration, Oracle to modern, OpenShift deployment, Java legacy modernization, SOAP REST conversion, microservices migration
  role: architect
  scope: full-stack migration
  output-format: code+analysis+deployment
  related-skills: legacy-modernizer, java-architect, microservices-architect, playwright-expert
---

# Java SOAP to Microservices Migration

End-to-end skill for migrating monolithic Java SOAP/WSDL applications with Oracle databases to containerized, cloud-native microservices on OpenShift using Java 21 LTS and Spring Boot 3.x.

## Core Migration Workflow

### Phase 1: Assessment & Planning
1. **Analyze legacy system** — Map SOAP/WSDL endpoints, database schema, business domains, and data dependencies
   - Extract service boundaries using Domain-Driven Design (DDD)
   - Document external integrations and data contracts
   - Produce a dependency graph and risk register
   - *Validation checkpoint:* All domains and dependencies documented before proceeding

2. **Design target architecture** — Sketch microservice boundaries, communication patterns, and deployment topology for OCP
   - Define service responsibilities using bounded contexts
   - Map data ownership per service
   - Plan API gateway layer
   - Select messaging strategy (REST, gRPC, or async events)
   - *Validation checkpoint:* Architecture diagram and service contract stubs approved before coding

3. **Plan database strategy** — Decompose monolithic Oracle schema into service-owned databases
   - Create database migration scripts (Flyway/Liquibase)
   - Plan data synchronization for dependent services
   - Assess distributed transaction requirements
   - *Validation checkpoint:* Migration scripts tested on staging Oracle instance

### Phase 2: Build Safety Net & Foundation
4. **Create characterization tests** — Capture legacy SOAP behavior as golden-master tests before refactoring
   - Test all WSDL operations with real Oracle data
   - Document response contracts and error cases
   - Achieve 80%+ coverage of critical paths
   - *Validation checkpoint:* Full test suite passes green on unmodified legacy code

5. **Set up modern CI/CD pipeline** — Prepare for OpenShift deployment with container registry, build pipelines, and observability
   - Create Dockerfile with multi-stage builds
   - Set up Maven/Gradle with Kubernetes plugins
   - Initialize Helm charts for OCP deployment
   - Configure distributed tracing, logging, and metrics
   - *Validation checkpoint:* Helm chart deploys successfully to OCP staging environment

### Phase 3: Incremental Migration
6. **Convert SOAP to REST/gRPC** — Build new service endpoints using Spring Boot 3.x Web/WebFlux
   - Implement domain objects as Spring Data entities
   - Create REST controllers with OpenAPI/Swagger documentation
   - Wire JPA/Hibernate for Oracle database access
   - Apply Spring Security with OAuth2/JWT for API protection
   - *Validation checkpoint:* All endpoints tested locally with embedded Oracle (TestContainers)

7. **Deploy with strangler fig pattern** — Route legacy SOAP traffic through an API gateway facade to new services
   - Create gateway service with routing rules
   - Implement feature flags for gradual traffic shifting
   - Monitor error rates, latency, and business metrics
   - *Validation checkpoint:* Gateway routes 5% traffic to new service, error rates within baseline

8. **Shift traffic incrementally** — Gradually increase traffic to new services (5% → 25% → 50% → 100%)
   - Monitor dashboards at each increment
   - Execute rollback procedures if metrics degrade
   - Verify database synchronization is working
   - *Validation checkpoint:* New service stable at 100% traffic for at least one release cycle before retiring legacy

### Phase 4: Testing & Validation
9. **Build comprehensive E2E test suite** — Use Playwright to automate user journey tests across all microservices
   - Test API integrations via REST/gRPC
   - Validate database state changes end-to-end
   - Cover happy paths, error cases, and edge cases
   - Create performance and load tests
   - *Validation checkpoint:* E2E suite runs green in staging; performance baseline established

10. **Validate business continuity** — Run production cutover checklist
    - Confirm all legacy functionality replicated in new services
    - Verify backward compatibility for client integrations
    - Validate monitoring and alerting pipelines
    - Execute game-day exercise simulating failure scenarios
    - *Validation checkpoint:* All stakeholders sign off; rollback plan tested and ready

---

## Reference Guides

Load detailed guidance based on migration phase:

| Topic | Reference | Load When |
|-------|-----------|-----------|
| WSDL Analysis | `references/wsdl-analysis.md` | Understanding legacy SOAP endpoints and schema |
| Service Boundaries | `references/ddd-service-design.md` | Designing microservice boundaries with DDD |
| Spring Boot 3.x Setup | `references/spring-boot-3-setup.md` | Project structure, dependencies, Java 21 features |
| REST API Design | `references/rest-api-design.md` | Converting SOAP operations to REST endpoints |
| Oracle Database | `references/oracle-database-migration.md` | Schema decomposition, migration scripts, TestContainers |
| OpenShift Deployment | `references/openshift-deployment.md` | Dockerfile, Helm charts, resource limits, health probes |
| Spring Security | `references/spring-security-oauth2.md` | API protection with JWT, role-based access control |
| Playwright Testing | `references/playwright-e2e-testing.md` | E2E test framework, page objects, data-driven tests |
| API Gateway | `references/api-gateway-pattern.md` | Gateway implementation, routing rules, traffic shifting |
| Strangler Fig | `references/strangler-fig-migration.md` | Facade pattern, feature flags, gradual migration |

---

## Code Architecture Overview

### Target Project Structure
```
my-app-platform/
├── api-gateway/                    # Spring Cloud Gateway (routes to services)
│   ├── src/main/java/.../
│   ├── Dockerfile
│   ├── helm-charts/gateway/values.yaml
│   └── pom.xml
│
├── user-service/                   # Example microservice
│   ├── src/main/java/.../
│   │   ├── controller/UserController.java
│   │   ├── service/UserService.java
│   │   ├── repository/UserRepository.java
│   │   ├── entity/User.java
│   │   ├── dto/UserDto.java
│   │   └── config/DatabaseConfig.java
│   ├── src/test/java/.../
│   │   └── e2e/UserServiceE2ETest.java
│   ├── src/main/resources/
│   │   ├── db/migration/V1__InitSchema.sql
│   │   └── application.yml
│   ├── Dockerfile
│   ├── helm-charts/user-service/values.yaml
│   └── pom.xml
│
├── order-service/                  # Another microservice example
│   ├── [similar structure to user-service]
│
├── shared-libs/                    # DTOs, common utilities
│   ├── src/main/java/.../
│   │   ├── dto/
│   │   ├── exception/
│   │   └── config/
│   └── pom.xml
│
├── e2e-tests/                      # Playwright test suite
│   ├── tests/
│   │   ├── user-flows.spec.ts
│   │   ├── order-flows.spec.ts
│   │   └── integration-flows.spec.ts
│   ├── pages/
│   │   ├── api-gateway.page.ts
│   │   └── user-service.page.ts
│   ├── fixtures/
│   │   ├── auth.fixture.ts
│   │   └── database.fixture.ts
│   ├── playwright.config.ts
│   ├── package.json
│   └── tsconfig.json
│
├── helm-charts/                    # Kubernetes/OpenShift deployment
│   ├── Chart.yaml
│   ├── values.yaml
│   ├── templates/
│   │   ├── deployment.yaml
│   │   ├── service.yaml
│   │   ├── configmap.yaml
│   │   ├── secret.yaml
│   │   └── ingress.yaml
│
├── infrastructure/                 # Infrastructure-as-Code
│   ├── docker-compose.yml          # Local development
│   ├── k8s/
│   │   ├── namespaces.yaml
│   │   ├── network-policies.yaml
│   │   └── rbac.yaml
│
└── README.md                       # Architecture and runbook
```

---

## Quick Start: Execution Steps

### 1. Use with Multi-Skill Agent Orchestration
This skill is designed to work with a **coordination prompt** that invokes multiple skills:
- `legacy-modernizer`: Phase 1 assessment and planning
- `java-architect`: Phase 3 implementation (services, security, data access)
- `microservices-architect`: Phase 1 & 3 (service boundaries, deployment)
- `playwright-expert`: Phase 4 (E2E testing)

**Start with:** Review `create-skill.prompt.md` for the orchestration prompt template.

### 2. Run Assessment Phase
```bash
# Create legacy SOAP system assessment
invoke: legacy-modernizer
- Analyze WSDL files and mapped SOAP endpoints
- Complete: ddd-service-design.md with bounded contexts
```

### 3. Design Microservices
```bash
invoke: microservices-architect
- Design service boundaries and communication
- Complete: OpenShift deployment topology
```

### 4. Build Services
```bash
invoke: java-architect
- Implement Spring Boot 3.x services
- Complete: REST endpoints, JPA models, Spring Security
- Database: Oracle with Flyway migrations
```

### 5. Test End-to-End
```bash
invoke: playwright-expert
- Build E2E test suite with page objects
- Complete: API integration tests, data validation
```

---

## Key Constraints & Requirements

### MUST DO
- **Java 21 LTS** — Use latest LTS features (records, sealed classes, pattern matching)
- **Spring Boot 3.x** — Use current stable version (3.2.x or later)
- **Oracle Database** — Support Oracle 19c+ with connection pooling (HikariCP)
- **Kubernetes-ready** — All services must run in containers with health probes and graceful shutdown
- **Observability** — Implement distributed tracing (OpenTelemetry), structured logging (SLF4J/Logback), and metrics (Micrometer)
- **Security** — Apply Spring Security with JWT/OAuth2 for API protection
- **Idempotency** — Ensure database migrations are idempotent (Flyway/Liquibase)
- **Testing** — Minimum 80% code coverage for business logic; E2E tests with Playwright

### MUST NOT DO
- Store secrets in code or ConfigMaps (use OpenShift Sealed Secrets or external vaults)
- Use synchronous blocking calls in reactive handlers
- Skip input validation on API endpoints
- Deploy without resource requests/limits in Helm charts
- Test only happy paths — include error scenarios, timeouts, data inconsistencies
- Leave legacy SOAP endpoints running after migration completion without documented sunset date

---

## Migration Workbook

Use these templates to guide your migration effort:

| Artifact | File | Purpose |
|----------|------|---------|
| Legacy Analysis | `templates/legacy-analysis.md` | Document SOAP operations, DB schema, data flows |
| WSDL to REST Mapping | `templates/wsdl-to-rest-mapping.md` | Correlate legacy SOAP calls to new REST endpoints |
| Service Contract | `templates/service-contract.md` | OpenAPI spec stubs for new microservices |
| Database Migration Plan | `templates/db-migration-plan.md` | Decomposition strategy and Flyway/Liquibase scripts |
| Feature Flags Config | `templates/feature-flags.md` | Define flags for strangler fig pattern |
| Helm Values Overlay | `templates/helm-values-ocp.yaml` | OpenShift-specific Kubernetes resources |
| E2E Test Plan | `templates/e2e-test-plan.md` | Playwright test scenarios and coverage matrix |
| Rollback Runbook | `templates/rollback-runbook.md` | Step-by-step procedures to revert traffic to legacy |

---

## Success Criteria

By the end of the migration, you should have:

✅ **Architecture**: Service boundary map, dependency diagrams, OpenAPI specs for all services  
✅ **Code**: Spring Boot 3.x microservices with Java 21, REST endpoints, JPA/Hibernate for Oracle  
✅ **Data**: Flyway migrations for schema decomposition, data synchronization scripts  
✅ **Deployment**: Helm charts for OpenShift, docker-compose for local dev, CI/CD pipelines  
✅ **Testing**: Unit tests (85%+ coverage), integration tests (TestContainers), E2E tests (Playwright)  
✅ **Operations**: Observability stack (logging, tracing, metrics), health probes, graceful shutdown  
✅ **Migration**: Strangler fig facade in production, feature flags, 100% traffic on new services, legacy sunset plan  

---

## Support & Examples

For hands-on examples and reference implementations:
- Review `templates/` directory for boilerplate code and configurations
- Consult `references/` for detailed guides on specific topics
- Check `create-skill.prompt.md` for multi-skill orchestration instructions
