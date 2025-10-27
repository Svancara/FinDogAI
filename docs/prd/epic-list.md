# Epic List

### **Epic 1: Foundation & Authentication Infrastructure**
**Goal:** Establish project setup, Firebase infrastructure, multi-tenant authentication, and basic data model with a deployable "hello world" that proves the tech stack works end-to-end.

### **Epic 2: Core Data Management & Business Setup**
**Goal:** Implement Jobs, Resources (Team Members, Vehicles, Machines), and Business/Person Profiles with full CRUD operations, enabling users to set up their business context before voice flows.

### **Epic 3: Voice Pipeline & Active Job Context**
**Goal:** Build the voice infrastructure (STT → LLM → TTS pipeline) and implement the "Set Active Job" voice flow with confirmation loop, establishing the foundational voice-first interaction pattern.

### **Epic 4: Journey Tracking & Cost Management**
**Goal:** Implement "Start Journey" voice flow and manual cost entry screens for all five categories (Transport, Material, Labor, Machine, Other), delivering complete cost capture functionality.

### **Epic 5: Audit Logging & Team Roles**
**Goal:** Implement Cloud Functions audit triggers, basic role-based access control for team members, and compliance features (GDPR data deletion, audit log TTL cleanup).

### **Epic 6: Reporting, Export & MVP Polish**
**Goal:** Add job financial summaries, offline sync status visibility, and final UX polish to deliver production-ready MVP; PDF generation is deferred to Phase 2.
