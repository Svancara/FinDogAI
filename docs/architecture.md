# FinDogAI Fullstack Architecture Document

## Overview

**FinDogAI** is a voice-first field service management system designed for tradespeople working in challenging environments where hands-free operation is essential. This document serves as the entry point to the complete technical architecture documentation.

### Core Technical Differentiators

- **Offline-First Architecture**: Full functionality without connectivity
- **Voice Pipeline Integration**: STT → LLM → TTS for natural interaction
- **Real-Time Sync**: Automatic multi-device synchronization when online
- **Multi-Tenant Isolation**: Secure data separation at the database level

## Architecture Documentation

The complete architecture documentation has been organized into focused sections for better maintainability and navigation. All detailed specifications are available in the [architecture folder](./architecture/).

### Core Architecture Documents

1. **[High-Level Overview](./architecture/high-level-overview.md)**
   - Technical summary and platform choices
   - Infrastructure and deployment regions
   - Repository structure and monorepo organization
   - Architectural patterns and diagrams
   - Key architectural decisions and rationales

2. **[Technology Stack](./architecture/tech-stack.md)**
   - Definitive technology selections and versions
   - Frontend and backend frameworks
   - Database, authentication, and storage solutions
   - Testing and development tools
   - Version compatibility matrix

3. **[Data Models](./architecture/data-models.md)**
   - Complete entity definitions with TypeScript interfaces
   - Tenant, Member, Job, Cost, and Resource models
   - Relationships and data management patterns
   - Soft delete and resource availability patterns
   - Design rationales and trade-offs

4. **[API Specification](./architecture/api-specification.md)**
   - Firestore direct access patterns
   - Cloud Functions callable API
   - Firestore triggers for automatic processing
   - Security rules integration
   - Error handling standards

### Implementation Details

5. **[Components](./architecture/components.md)**
   - Frontend components (Voice Pipeline, Offline Sync, Job Management)
   - Backend components (Sequence Allocator, Audit Logger, PDF Generator)
   - Shared components (Type Definitions, Validation Library)
   - Component interaction diagrams
   - Deployment strategies

6. **[Project Structure](./architecture/project-structure.md)**
   - Unified monorepo structure
   - Directory organization and file layout
   - Dependency graph and build order
   - File naming conventions
   - Module organization patterns

7. **[Development Workflow](./architecture/development-workflow.md)**
   - Local development setup and prerequisites
   - Environment configuration
   - Development commands and best practices
   - Firebase emulator usage
   - Debugging and code quality tools

8. **[Deployment](./architecture/deployment.md)**
   - Deployment strategy for frontend and backend
   - CI/CD pipeline with GitHub Actions
   - Environment configurations (dev, staging, production)
   - Monitoring, alerting, and security best practices
   - Disaster recovery procedures

### Standards & Guidelines

9. **[Coding Standards](./architecture/coding-standards.md)**
   - Critical fullstack rules
   - Naming conventions
   - TypeScript, Angular, and Cloud Functions standards
   - Error handling and testing standards
   - Security and performance guidelines

## Quick Start

### For AI Assistants
Start with [Data Models](./architecture/data-models.md) and [API Specification](./architecture/api-specification.md) to understand the system structure, then review [Coding Standards](./architecture/coding-standards.md) for implementation guidelines.

### For Frontend Developers
Focus on [Components](./architecture/components.md), [Tech Stack](./architecture/tech-stack.md), and [Coding Standards](./architecture/coding-standards.md).

### For Backend Developers
Prioritize [API Specification](./architecture/api-specification.md), [Data Models](./architecture/data-models.md), and [Deployment](./architecture/deployment.md).

### For DevOps Engineers
See [Deployment](./architecture/deployment.md), [Project Structure](./architecture/project-structure.md), and [Development Workflow](./architecture/development-workflow.md).

## System Architecture Summary

### Platform
Firebase (Google Cloud Platform) with services:
- **Authentication**: Identity management with offline token caching
- **Firestore**: NoSQL database with offline sync and conflict resolution
- **Cloud Functions 2nd Gen**: Serverless compute for server-only operations
- **Cloud Storage**: File/PDF storage with signed URL access
- **Hosting**: PWA delivery via global CDN

### Tech Stack Highlights
- **Frontend**: Angular 20+ / Ionic 8+ / Capacitor 5+
- **Backend**: TypeScript Cloud Functions (Node.js 20)
- **Database**: Cloud Firestore with offline persistence
- **State Management**: NgRx 18+
- **Build Tool**: Nx 19+ monorepo
- **Testing**: Jest, Playwright
- **CI/CD**: GitHub Actions

### Architectural Patterns
- Offline-First with automatic sync
- Direct Firestore access with security rules
- Event-driven backend with Firestore triggers
- Component-based UI with Ionic
- Repository pattern for data access
- Optimistic UI updates with eventual consistency

## Architecture Principles

1. **Offline-First**: Full functionality without connectivity
2. **Voice-Driven**: Natural language interaction as primary interface
3. **Type-Safe**: Shared TypeScript definitions across the stack
4. **Serverless**: Auto-scaling with pay-per-use model
5. **Multi-Tenant**: Secure data isolation at database level

## Related Documentation

- **[Sharded Architecture](./architecture/)** - Detailed technical specifications (you are here)
- **[PRD Documentation](./prd/)** - Product requirements
- **[Backend Architecture](./backend-architecture/)** - Detailed backend specifications
- **[UI Architecture](./ui-architecture/)** - Detailed frontend specifications
- **[Implementation Checklist](./implementation-checklist.md)** - Development roadmap

## Document Maintenance

- **Version:** 1.0
- **Last Updated:** 2025-10-29
- **Maintained By:** Architecture Team
- **Review Cycle:** Quarterly or on major changes

## Change Log

| Date | Version | Description | Author |
|------|---------|-------------|--------|
| 2025-10-29 | v1.0 | Initial fullstack architecture from sharded docs consolidation | Winston (Architect) |

---

*For detailed technical specifications, navigate to the [architecture folder](./architecture/) and select the relevant section.*

*This is the authoritative source for FinDogAI technical architecture. All development should align with these specifications.*
