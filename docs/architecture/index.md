# FinDogAI Architecture Documentation Index

This folder contains the complete technical architecture documentation for FinDogAI, organized into focused sections for better maintainability and navigation.

## 📚 Documentation Structure

### Core Architecture
- [High-Level Overview](./high-level-overview.md) - System architecture, patterns, and platform choices
- [Tech Stack](./tech-stack.md) - Complete technology selections and versions
- [Data Models](./data-models.md) - Entity definitions, schemas, and relationships
- [API Specification](./api-specification.md) - Firestore patterns, Cloud Functions, and triggers

### System Design
- [Security](./security.md) - Authentication, authorization, multi-tenant isolation, and security best practices
- [Voice Architecture](./voice-architecture.md) - Complete voice pipeline, online/offline modes, STT/LLM/TTS integration
- [Voice Pipeline Implementation](./voice-pipeline-implementation.md) - **NEW:** Detailed sequence diagrams, error handling (FR21), latency optimization (NFR1/NFR2), and quality monitoring (NFR5/NFR6)
- [Performance & Quality Monitoring](./performance-monitoring-architecture.md) - **NEW:** NFR compliance monitoring, rate limiting, infrastructure sizing, and alerting strategy
- [Third-Party Integrations](./third-party-integrations.md) - Voice APIs (STT/LLM/TTS), Stripe payments, OAuth providers, rate limiting, and quota management
- [Monitoring](./monitoring.md) - Observability stack, metrics, logging, alerting, and dashboards

### Implementation Details
- [Components](./components.md) - Frontend, backend, and shared component architecture
- [Frontend Integration Services](./frontend-integration-services.md) - Angular services for voice, payments, and auth with NgRx state management
- [Project Structure](./project-structure.md) - Monorepo organization and file layout
- [Development Workflow](./development-workflow.md) - Setup instructions and development commands
- [Deployment](./deployment.md) - CI/CD pipeline and deployment strategy

### Standards & Guidelines
- [Coding Standards](./coding-standards.md) - Critical rules, naming conventions, and best practices
- [Testing Strategy](./testing-strategy.md) - Comprehensive testing approach covering unit, integration, E2E, performance, security, accessibility, and visual regression testing

### Feature-Specific Guides
- [Business Profile Migration](./business-profile-migration.md) - Migration guide for resource visibility feature
- [Business Profile Implementation Checklist](./business-profile-implementation-checklist.md) - Developer implementation roadmap

### Architecture Validation
- [Architecture Checklist](../ARCHITECTURE_CHECKLIST.md) - Comprehensive validation report with NFR compliance assessment
- [Architecture Gaps Resolved](./ARCHITECTURE_GAPS_RESOLVED.md) - **NEW:** Summary of addressed gaps from section 1.2 (NFR alignment)

## 🎯 Quick Navigation

### For AI Assistants
Start with [Data Models](./data-models.md) and [API Specification](./api-specification.md) to understand the system structure.

### For Frontend Developers
Focus on [Components](./components.md), [Tech Stack](./tech-stack.md), [Coding Standards](./coding-standards.md), and [Testing Strategy](./testing-strategy.md).

### For Backend Developers
Prioritize [API Specification](./api-specification.md), [Data Models](./data-models.md), [Testing Strategy](./testing-strategy.md), and [Deployment](./deployment.md).

### For DevOps Engineers
See [Deployment](./deployment.md), [Testing Strategy](./testing-strategy.md), [Project Structure](./project-structure.md), and [Development Workflow](./development-workflow.md).

### For QA Engineers
Start with [Testing Strategy](./testing-strategy.md), then review [Tech Stack](./tech-stack.md) and [Deployment](./deployment.md) for CI/CD integration.

## 📋 Related Documentation

- [Parent Overview](../architecture.md) - High-level summary
- [PRD Documentation](../prd/) - Product requirements
- [Backend Architecture](../backend-architecture/) - Detailed backend specifications
- [UI Architecture](../ui-architecture/) - Detailed frontend specifications
- [Implementation Checklist](../implementation-checklist.md) - Development roadmap

## 🏗️ Architecture Principles

1. **Offline-First** - Full functionality without connectivity
2. **Voice-Driven** - Natural language interaction as primary interface
3. **Type-Safe** - Shared TypeScript definitions across the stack
4. **Serverless** - Auto-scaling with pay-per-use model
5. **Multi-Tenant** - Secure data isolation at database level

## 📝 Document Maintenance

- **Version:** 1.0
- **Last Updated:** 2025-10-29
- **Maintained By:** Architecture Team
- **Review Cycle:** Quarterly or on major changes

## 🔄 Change Process

1. Create a branch for architecture changes
2. Update relevant documents in this folder
3. Update this index if adding new sections
4. Submit PR with clear description of changes
5. Require review from tech lead

---

*This is the authoritative source for FinDogAI technical architecture. All development should align with these specifications.*