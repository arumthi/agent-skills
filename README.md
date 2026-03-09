# Java SOAP to Microservices Migration Skill: Complete Guide

## Welcome! 👋

This skill provides a complete, step-by-step framework for migrating legacy Java SOAP/WSDL applications to modern Spring Boot 3.x microservices deployable on OpenShift OCP.

**What you get**:
- ✅ 10-phase migration workflow with validation checkpoints
- ✅ Reference guides for WSDL analysis, DDD service design, Spring Boot setup, Oracle migration, Kubernetes deployment
- ✅ Multi-skill orchestration prompt to coordinate legacy-modernizer, java-architect, microservices-architect, and playwright-expert
- ✅ Playwright E2E test framework with page objects and data fixtures
- ✅ Boilerplate templates for legacy analysis, service contracts, migration plans
- ✅ Production-ready Helm charts for OpenShift deployment

---

## Quick Start (15 minutes)

### 1. Read the Master Orchestration Prompt

Start here: **`create-skill.prompt.md`**

This is your execution roadmap. It breaks down the entire migration into **6 sequential stages**, each invoking a specific skill:

- **Stage 1**: legacy-modernizer (assess legacy SOAP system)
- **Stage 2**: microservices-architect (design target architecture)
- **Stage 3**: java-architect (implement Spring Boot services, 1 per microservice)
- **Stage 4**: java-architect (build API gateway with strangler pattern)
- **Stage 5**: (manual execution of traffic shifting phases)
- **Stage 6**: playwright-expert (create E2E test suite)

### 2. Review the Main Skill Document

Next: **`SKILL.md`**

This document contains:
- The **10-phase migration workflow** with validation checkpoints
- Reference guide catalog (what each guide is for and when to use it)
- Target project structure (how your code will be organized)
- Key constraints and requirements
- Success criteria for the entire migration

### 3. Use Templates to Get Started

Use these templates as starting points for your specific migration:

| Template | Purpose | When to Use |
|----------|---------|------------|
| `templates/legacy-analysis.md` | Document your legacy SOAP system (endpoints, schema, load) | First task after assessment phase |
| `templates/ddd-service-boundaries.md` | Define service boundaries and bounded contexts | After identifying business domains |
| `templates/e2e-test-plan.md` | Plan E2E test scenarios and coverage | Before implementing Playwright tests |
| `templates/db-migration-plan.md` | Plan schema decomposition and Flyway migrations | During implementation phase |

### 4. Load Reference Guides As Needed

**Reference guides** are detailed, technical deep-dives on specific topics. Load them when you're ready to execute that phase:

| Guide | Content | Load When |
|-------|---------|-----------|
| `references/wsdl-analysis.md` | How to extract WSDL contracts | Analyzing legacy SOAP system |
| `references/ddd-service-design.md` | DDD patterns and service decomposition | Designing service boundaries |
| `references/spring-boot-3-setup.md` | Spring Boot 3.x project setup with Java 21 | Creating new microservice |
| `references/oracle-database-migration.md` | Schema decomposition and Flyway migrations | Migrating database |
| `references/openshift-deployment.md` | Helm charts and Kubernetes manifests | Setting up OCP deployment |
| `references/playwright-e2e-testing.md` | Playwright test framework, fixtures, page objects | Building E2E tests |

---

## Detailed Workflow

### Phase 1: Assessment & Analysis
**Owner**: Legacy Modernizer Skill  
**Artifact**: `templates/legacy-analysis.md`

1. Fill out the legacy-analysis template with your SOAP system details
2. Extract WSDL operations using SoapUI or wsimport
3. Map Oracle schema tables and relationships
4. Define external integrations (payment gateway, shipping API, etc.)
5. Document current performance baselines

**Deliverable**: Completed legacy-analysis.md with all WSDL operations, database schema, and identified risks

---

### Phase 2: Service Boundary Identification
**Owner**: Microservices Architect Skill  
**Artifact**: `templates/ddd-service-boundaries.md`

1. Load `references/ddd-service-design.md` for detailed guidance
2. Group WSDL operations by business domain
3. Define bounded contexts for each service
4. Document domain events and inter-service communication
5. Create context map showing service relationships

**Deliverable**: Completed ddd-service-boundaries.md with all bounded contexts mapped

---

### Phase 3: Design Target Architecture
**Owner**: Microservices Architect Skill

1. Design microservice topology (services, databases, communication patterns)
2. Plan API gateway layer for backward compatibility
3. Design event-driven messaging (Kafka/RabbitMQ topics)
4. Plan observability stack (logging, tracing, metrics)
5. Create Kubernetes/OpenShift deployment topology

**Deliverable**: Services architecture doc, OpenShift namespace design, event model

---

### Phase 4: Implement Microservices
**Owner**: Java Architect Skill (1 invocation per service)  
**Artifact**: Spring Boot 3.x service code

For `order-service`, `inventory-service`, `payment-service`, `notification-service`:

1. Load `references/spring-boot-3-setup.md`
2. Initialize Spring Boot 3.x project with Java 21
3. Create JPA entities mapping your Oracle tables
4. Implement REST controllers with OpenAPI/Swagger
5. Configure Spring Security with JWT
6. Add Flyway migration scripts
7. Create unit and integration tests (80%+ coverage)
8. Write Dockerfile for containerization

**Deliverable**: Each microservice is a complete Maven project with:
- Spring Boot 3.x configuration
- REST endpoints with OpenAPI docs
- JPA/Hibernate domain models
- Unit and integration tests
- Flyway database migrations
- Dockerfile with multi-stage build

---

### Phase 5: Build API Gateway with Strangler Facade
**Owner**: Java Architect Skill

1. Create Spring Cloud Gateway service
2. Implement request routing rules:
   - SOAP `/legacy/*` requests → translate to REST → call new services
   - REST `/api/v1/*` requests → route to new services directly
3. Implement feature flags for traffic shifting (5% → 25% → 50% → 100%)
4. Add circuit breaker for new services (fallback to legacy)
5. Implement request/response transformation
6. Deploy to OpenShift

**Deliverable**: API Gateway service with strangler fig pattern and traffic shifting

---

### Phase 6: Database Migration
**Owner**: Java Architect + Manual SQL

1. Load `references/oracle-database-migration.md`
2. Decompose monolithic schema into service-owned databases
3. Create Flyway migration scripts (V1__*, V2__*, etc.)
4. Test migrations on staging Oracle instance
5. Execute data export/import from monolithic to new schemas
6. Verify data consistency and referential integrity

**Deliverable**: Idempotent Flyway migrations for each service database

---

### Phase 7: Create E2E Test Suite
**Owner**: Playwright Expert Skill  
**Artifact**: TypeScript + Playwright test suite

1. Load `references/playwright-e2e-testing.md`
2. Set up Playwright project with TypeScript
3. Create page objects (API client abstractions)
4. Create test fixtures for authentication and database setup
5. Write test scenarios for critical user journeys
6. Implement integration tests across microservices
7. Set up CI/CD pipeline (GitHub Actions / GitLab CI)

**Deliverable**: Complete Playwright test suite with:
- Page object models for each service
- 80%+ coverage of critical workflows
- Data-driven tests
- HTML test reports
- CI/CD integration

---

### Phase 8: Incremental Traffic Migration
**Owner**: DevOps + Manual Execution

1. Deploy API Gateway with feature flag set to 0% new traffic
2. Run production health checks
3. **Phase 1 (5% traffic)**:
   - Enable feature flag for 5% requests
   - Monitor error rates, latency, database performance
   - Validation checkpoint: All metrics green for 24 hours
4. **Phase 2 (25% traffic)**: Increase to 25%, monitor 24 hours
5. **Phase 3 (50% traffic)**: Increase to 50%, monitor 24 hours
6. **Phase 4 (100% traffic)**: Route all traffic to new services
7. Keep legacy SOAP running for 2-week safety period
8. Monitor new services for 48 hours at full load
9. After sign-off, decommission legacy SOAP

**Deliverable**: Production cutover with zero downtime

---

### Phase 9: Production Validation
**Owner**: QA + Stakeholders

1. Run full E2E test suite in production
2. Execute game-day exercise (simulate failures)
3. Verify monitoring dashboards operational
4. Confirm all stakeholder sign-offs
5. Document migration completion and lessons learned

**Deliverable**: Production sign-off and post-mortem documentation

---

### Phase 10: Legacy System Decommission
**Owner**: Infrastructure Team

1. Stop legacy SOAP application
2. Archive legacy codebase
3. Remove legacy database backups (per retention policy)
4. Document sunset timeline for dependent systems
5. Notify external API consumers of deprecation

**Deliverable**: Legacy system retired, migration complete! 🎉

---

## How to Invoke Skills

This skill is designed to work with **multi-skill orchestration**. Use the orchestration prompt to coordinate all skills:

### Option 1: Run All Phases (End-to-End)

Copy the entire `create-skill.prompt.md` and paste into chat:

```
Follow instructions in create-skill.prompt.md:
[paste the full orchestration prompt]
```

The agent will execute all 6 stages sequentially, invoking the appropriate skills at each step.

### Option 2: Run Individual Phases

If you only need a specific phase, copy just that section:

```
I need help with Phase 1: Assessment & Analysis

[copy Phase 1 section from create-skill.prompt.md]
```

The agent will invoke legacy-modernizer for your specific phase.

### Option 3: Use Standalone Skill References

If you just need technical guidance without skill coordination:

```
I'm migrating a SOAP application to microservices. Help me:
1. Analyze my WSDL files
2. Design Spring Boot REST endpoints

[load references/wsdl-analysis.md and references/spring-boot-3-setup.md]
```

---

## Key Files & Their Purpose

### Core Skill Files
- **`SKILL.md`** ← Start here! Overview of entire skill and 10-phase workflow
- **`create-skill.prompt.md`** ← Orchestration prompt for multi-skill execution
- **`README.md`** ← This file

### Reference Guides (Technical Deep-Dives)
- **`references/wsdl-analysis.md`** — Extract WSDL contracts, schema, load characteristics
- **`references/ddd-service-design.md`** — Design service boundaries with Domain-Driven Design
- **`references/spring-boot-3-setup.md`** — Set up Spring Boot 3.x project with Java 21 LTS
- **`references/oracle-database-migration.md`** — Decompose monolithic schema into service-owned databases
- **`references/openshift-deployment.md`** — Create Helm charts and Kubernetes manifests for OCP
- **`references/playwright-e2e-testing.md`** — Build comprehensive E2E test suite

### Templates (Artifacts for Your Specific Migration)
- **`templates/legacy-analysis.md`** — Analyze your legacy SOAP system
- **`templates/ddd-service-boundaries.md`** — Define your service boundaries
- **`templates/e2e-test-plan.md`** — Plan your E2E tests
- **`templates/db-migration-plan.md`** — Plan your database decomposition

---

## Common Use Cases

### "I need to migrate a SOAP application to microservices"

1. Read `SKILL.md` (10 min)
2. Copy and run `create-skill.prompt.md` in chat
3. Agent will guide you through all 6 stages

### "I'm already using legacy-modernizer. What's next?"

Once legacy-modernizer completes Phase 1 (assessment), move to:
1. Load `references/ddd-service-design.md` (understand bounded contexts)
2. Create your `ddd-service-boundaries.md` (define services)
3. Invoke microservices-architect for Phase 2 (design target architecture)

### "I have the architecture. How do I build the services?"

1. Load `references/spring-boot-3-setup.md` (learn Spring Boot 3.x structure)
2. For each microservice:
   - Invoke java-architect to implement service
   - Use reference as guide for best practices
3. Then load `references/openshift-deployment.md` for Kubernetes setup

### "I need E2E tests. How do I use Playwright?"

1. Load `references/playwright-e2e-testing.md`
2. Copy test examples and adapt to your APIs
3. Invoke playwright-expert when ready for professional test architecture

### "How do I deploy to OpenShift?"

1. Load `references/openshift-deployment.md` (understand Helm charts)
2. Copy Helm chart templates from reference
3. Customize `values.yaml` for your environment
4. Deploy: `helm install` with your custom values

---

## Success Criteria

When migration is complete, you will have:

### ✅ Architecture
- Service boundary diagram showing all microservices
- Dependency graph showing inter-service communication
- Event model showing domain events and event flows
- OpenAPI specs for all REST endpoints

### ✅ Code
- Spring Boot 3.x microservices (1 per bounded context)
- REST API Gateway with strangler pattern and feature flags
- JPA/Hibernate entities for Oracle database access
- Unit tests (85%+ coverage) + integration tests (TestContainers)
- Playwright E2E tests for critical user journeys

### ✅ Data
- Flyway migration scripts (V1_*, V2_*, etc.) for schema decomposition
- Idempotent migrations that can be re-run safely
- Data successfully migrated from monolithic to service-owned databases
- Data consistency verified end-to-end

### ✅ Deployment
- Helm charts for each microservice
- Docker images in container registry
- CI/CD pipelines for automated builds and tests
- OpenShift routes and network policies
- Monitoring dashboards for logging, tracing, metrics

### ✅ Testing
- 80%+ coverage of critical business workflows
- E2E tests passing in staging and production
- Performance baselines established
- Load tests passing (target peak load)

### ✅ Operations
- Runbooks for deployment, scaling, and incident response
- Observability: Prometheus metrics, Jaeger tracing, ELK logging
- Health checks (liveness, readiness probes)
- Graceful shutdown and pod termination handling

---

## Frequently Asked Questions

### Q: "Can I run this skill without using the orchestration prompt?"

**A**: Yes! Each reference guide is standalone, and templates are self-contained. You can:
- Read SKILL.md and tackle each phase manually
- Load individual references as you need them
- Create templates using provided examples
- Invoke individual skills (legacy-modernizer, java-architect, etc.) for specific phases

The orchestration prompt is just a **convenient way** to coordinate all phases sequentially; it's not required.

### Q: "Do I have to follow all 10 phases?"

**A**: The 10 phases represent a **complete migration**. However, you can:
- Skip Phase 10 (decommission) if keeping legacy running
- Compress phases if you have high team capacity
- Run phases in parallel if you have separate teams
- Stop at Phase 8 if you don't need E2E testing

The phases are **ordered dependencies** (can't do Phase 3 before Phase 2), but they're flexible.

### Q: "What if my legacy system is different?"

**A**: The phases are **generic patterns** that apply to any SOAP monolith. Adapt the templates:
- Use `templates/legacy-analysis.md` for YOUR system's WSDL and schema
- Adjust `ddd-service-boundaries.md` to reflect YOUR business domains
- Modify `spring-boot-3-setup.md` examples for YOUR specific services

The **concepts** (DDD, strangler pattern, event-driven saga, Playwright E2E tests) apply universally.

### Q: "How do I handle backward compatibility during migration?"

**A**: The **API Gateway with strangler pattern** (Phase 4) handles this:
1. Gateway intercepts all legacy SOAP requests
2. Translates SOAP → REST and routes to new services
3. Translates response back REST → SOAP
4. If new service fails, routes to legacy SOAP as fallback
5. Feature flag controls percentage of traffic to new services

This allows gradual migration (5% → 25% → 50% → 100%) without breaking existing clients.

### Q: "What's the test coverage requirement?"

**A**: Targets differ by layer:
- **Unit tests**: 85%+ of business logic
- **Integration tests**: 80%+ of critical data flows
- **E2E tests**: 80%+ of user-facing workflows

E2E tests are most important; they validate the entire system end-to-end.

---

## Next Steps

1. **Read `SKILL.md`** (10 minutes)
2. **Copy `create-skill.prompt.md`** and paste into chat to invoke multi-skill orchestration
3. **Follow the 6 stages** as the agent guides you
4. **Reference the guides** as needed during your implementation
5. **Use templates** to document your specific migration artifacts

---

## Support & Troubleshooting

If you get stuck:

1. **Check the relevant reference guide** (most questions answered there)
2. **Review the template examples** (see how others document similar migrations)
3. **Re-invoke the orchestration prompt** with specific phase (agent can retry)
4. **Invoke individual skills** for specific technical help

---

## Document Versioning

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2024 | Initial release with 10-phase workflow, 6 reference guides, 4 templates |

---

**Happy migrating! 🚀**

For questions or refinements, reach out to your architecture team.

