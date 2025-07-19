
# SuperClaude Entry Point
@COMMANDS.md
@FLAGS.md
@PRINCIPLES.md
@RULES.md
@MCP.md
@PERSONAS.md
@ORCHESTRATOR.md
@MODES.md

# Care Management System – CLAUDE.md

## Project Overview
This repository contains the **Care Management System**, which is built to handle:
- Care package management.
- Carer coordination and scheduling.
- Competency tracking and assessments.
- Rota management and shift distribution.
- PDF reporting and compliance tools.

The system follows a structured **8-phase roadmap**, with **security, scalability, and maintainability** at its core.

## Tech Stack
- **Backend:** NestJS with Prisma ORM + PostgreSQL  
- **Frontend:** Next.js (Admin & Carer Web Apps)  
- **Mobile:** React Native (Carer Mobile App with offline sync)  
- **Infrastructure:** Render (preferred) or AWS for deployments, using Dockerized services  
- **Notifications:** Firebase Cloud Messaging (push notifications) and SendGrid (emails)  
- **Storage:** S3-compatible storage for assessment reports and PDFs

# Core Principles
### 1. Always Reference the Master Plan
- **PLAN.md:** Contains the **full project roadmap and architecture**.
- **plans/Phase-X.md:** Each phase (1–8) has a **detailed implementation guide** with milestones.

### 2. Security First
- All features must include **Role-Based Access Control (RBAC)**, **JWT authentication**, encryption for sensitive data, and input validation.

### 3. Clean Architecture & Domain-Driven Design (DDD)
- Keep business logic independent of frameworks.
- Code organized by **business domains** (e.g., `/modules/assessments`, `/modules/rota`, `/modules/care-packages`).

### 4. Testing & Quality
- Minimum **80% test coverage** is required before merging.
- Use **unit tests, integration tests, and e2e tests** as outlined per phase.

### 5. Documentation-Driven
- Keep `PLAN.md` and **phase-specific documents in `plans/`** updated with changes.
- Maintain a **CHANGELOG.md**.

### 6. Performance & Scalability
- Optimize database queries with Prisma and indexing.
- Use caching strategies where needed (e.g., Redis for sessions/queues).

# Development Workflow
## Step-by-Step Implementation Process
1. **Read the Phase Plan**  
   - Start by checking `plans/Phase-X.md` for the current phase requirements.
2. **Create a Feature Branch**  
   - Use the format:  
     `feature/phase-X-feature-name`
3. **Implement According to Architecture**  
   - Follow Clean Architecture and DDD principles. Keep components modular.
4. **Write Tests First**  
   - Apply **Test-Driven Development (TDD)** when possible.
5. **Run Quality Checks**  
   - Run all linting, tests, and builds locally:
     ```bash
     npm run test && npm run lint && npm run build
     ```
6. **Push to Git**  
   - Follow the commit message format below.
7. **Deploy to Staging**  
   - Deploy to Render **at the end of each phase** for QA.

# Git Workflow
## Branch Naming Conventions
- `feature/phase-X-feature-name` – for new features.
- `bugfix/phase-X-description` – for bug fixes.
- `hotfix/critical-fix-description` – for urgent patches.

## Commit Message Format
```
feat(phase-X): add feature description
fix(phase-X): resolve bug description
docs(phase-X): update documentation
test(phase-X): add tests for [module]
```

# Project Structure
```
Care-system/
├── PLAN.md                # Master roadmap and phases
├── CLAUDE.md              # This development workflow ruleset
├── plans/                 # Phase-specific blueprints
│   ├── Phase-1.md
│   ├── Phase-2.md
│   └── ...
├── apps/                  # Core application code
│   ├── backend/           # NestJS backend
│   ├── admin-web/         # Admin web app (Next.js)
│   ├── carer-web/         # Carer web app (Next.js)
│   └── mobile/            # Carer mobile app (React Native)
├── packages/              # Shared modules/libraries
│   ├── ui/                # UI components
│   ├── types/             # Shared TypeScript types
│   └── utils/             # Utility functions
└── docs/                  # API docs, ERD, and diagrams
```

# Quality Gates
Before merging or moving to the next phase:
- [ ] All **phase milestones are completed**.
- [ ] Code passes **ESLint + Prettier**.
- [ ] **Test coverage is at least 80%**.
- [ ] **Security checks** (Snyk or OWASP top 10) pass.
- [ ] Documentation (`PLAN.md`, `plans/`) is updated.
- [ ] **Staging deployment tested** and validated.

# Security Checklist
- Use **short-lived JWT access tokens** with refresh token rotation.
- **Role-Based Access Control** on all backend routes.
- Data encryption at rest (PostgreSQL, S3) and **TLS in transit**.
- **Audit Logging** of all CRUD operations (carers, tasks, assessments, rota).
- Secure **PDF reports** (signed URLs, limited access).

# Deployment Rules
- **Staging Deployment:**  
  Deployed **at the end of every phase** (e.g., M1.4, M2.4).
- **Production Deployment:**  
  Only after **Phase 8 QA is passed** (M8.2).
- **Mobile App Publishing:**  
  Submit builds to **App Store & Google Play** during **Phase 8** (M8.3).

# Testing Commands
```bash
# Run all tests
npm run test

# Run unit tests
npm run test:unit

# Run integration tests
npm run test:integration

# Check coverage
npm run test:coverage
```

# Best Practices
- **Never commit secrets** – always use `.env` files or Render secrets.
- Write **modular, reusable code** – shared components go in `/packages`.
- Optimize database queries (Prisma with proper indexing).
- Use structured logging (Winston or Pino).
- Generate PDFs server-side and distribute them via **signed URLs**.

# Key Documents
- **PLAN.md:** Master roadmap (8 phases).
- **plans/:** Detailed implementation guides per phase.
- **docs/:** Architecture diagrams, ERDs, and API contracts.
- **CHANGELOG.md:** History of features and changes.

# Next Steps for Developers
1. **Start with the active phase plan** (`plans/`).
2. **Create a feature branch** using the naming convention.
3. **Implement features** following Clean Architecture and DDD.
4. **Push to Git** with proper commit messages.
5. **Deploy to staging** and request **code review**.
6. **Merge only after all quality gates pass**.
