# User Personas & Roles

## Overview

The AI Software Factory serves multiple user types across the software production lifecycle. Each role has specific permissions, workflows, and interface needs.

## Role Hierarchy

```
Platform Admin
    └── Project Owner
        └── Team Lead
            ├── Developer
            ├── Designer
            └── QA Engineer
        └── Product Manager
    └── Viewer (read-only)
```

## Detailed Personas

### 1. Platform Admin

| Field | Value |
|-------|-------|
| Role ID | `platform_admin` |
| Access Level | Full system access |
| Primary Goal | Platform health, user management, configuration |

**Capabilities:**
- Manage all users and roles
- Configure LLM providers and API keys
- Monitor system health and costs
- Access all projects
- Manage infrastructure settings
- View audit logs

**Key Pages:** Settings, User Management, System Health, Cost Dashboard

---

### 2. Project Owner (CTO / VP Engineering)

| Field | Value |
|-------|-------|
| Role ID | `project_owner` |
| Access Level | Full project access |
| Primary Goal | Strategic decisions, final approvals, resource allocation |

**Capabilities:**
- Create and configure projects
- Submit high-level goals and PRDs
- Final approval authority (override team lead)
- View all execution history and costs
- Configure project-level settings
- Assign team leads

**Key Pages:** Dashboard (overview), Approvals, Cost Analytics, Project Settings

---

### 3. Team Lead

| Field | Value |
|-------|-------|
| Role ID | `team_lead` |
| Access Level | Full feature access within assigned modules |
| Primary Goal | Review plans, approve execution, merge outputs, quality gate |

**Capabilities:**
- Review and approve/reject plans
- Merge approved outputs into prompt repository
- Assign tasks to developers
- Monitor execution progress
- Trigger re-execution on failure
- Manage prompt repository (commit, version bump)
- Configure agent parameters for their modules

**Key Pages:** Approvals, Prompt Repository, Executions, Team Dashboard

**Critical Role:** After approval at each stage, the team lead merges output into the repository and bumps version.

---

### 4. Developer

| Field | Value |
|-------|-------|
| Role ID | `developer` |
| Access Level | Read/write on assigned features |
| Primary Goal | Submit tasks, review generated code, provide feedback |

**Capabilities:**
- Submit feature requests and bug reports
- View generated plans (read-only)
- Review agent-generated code
- Provide feedback (accept/reject/edit)
- Browse prompt repository (read-only)
- View execution logs

**Key Pages:** Tasks, Executions (own), Feedback, Code Review

---

### 5. Product Manager

| Field | Value |
|-------|-------|
| Role ID | `product_manager` |
| Access Level | Read on technical, write on product artifacts |
| Primary Goal | Define requirements, prioritize backlog, track progress |

**Capabilities:**
- Submit PRDs and feature requests
- Prioritize task backlog
- Review plans (scope/timeline, not technical)
- View progress dashboards
- Manage user stories and acceptance criteria
- Cannot approve technical execution

**Key Pages:** Tasks (submit), Plans (review scope), Dashboard (progress), Backlog

---

### 6. Designer

| Field | Value |
|-------|-------|
| Role ID | `designer` |
| Access Level | Read/write on UI/design files |
| Primary Goal | Define UI specs, review generated interfaces |

**Capabilities:**
- Submit UI requirements and wireframes
- Review generated frontend code (visual)
- Provide design feedback
- Update design system files in repository
- Cannot approve backend changes

**Key Pages:** Design System, UI Preview, Feedback (design)

---

### 7. QA Engineer

| Field | Value |
|-------|-------|
| Role ID | `qa_engineer` |
| Access Level | Read on code, write on test artifacts |
| Primary Goal | Validate generated tests, report issues |

**Capabilities:**
- Review generated test suites
- Submit bug reports
- Validate execution results
- Provide quality feedback
- View test coverage reports

**Key Pages:** Test Results, Bug Reports, Coverage, Executions

---

### 8. Viewer

| Field | Value |
|-------|-------|
| Role ID | `viewer` |
| Access Level | Read-only on all non-sensitive data |
| Primary Goal | Observe progress, learn from system |

**Capabilities:**
- View dashboards and progress
- Browse prompt repository (read-only)
- View execution history (no logs)
- Cannot submit, approve, or modify anything

**Key Pages:** Dashboard (read-only), Repository Browser

---

## Permissions Matrix

| Action | Admin | Owner | Lead | Dev | PM | Designer | QA | Viewer |
|--------|-------|-------|------|-----|-----|----------|-----|--------|
| Create project | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Submit task | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| View plans | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Approve plans | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Merge to repo | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Edit repo files | ✅ | ✅ | ✅ | ❌ | ❌ | 🟡* | ❌ | ❌ |
| View executions | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| Provide feedback | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Manage users | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| View costs | ✅ | ✅ | ✅ | ❌ | ✅ | ❌ | ❌ | ❌ |
| Configure LLM | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ | ❌ |
| Version bump | ✅ | ✅ | ✅ | ❌ | ❌ | ❌ | ❌ | ❌ |

*🟡 Designer can edit only `ui/` and `design.md` files.

## Authentication Flow

1. User navigates to platform
2. Redirected to OAuth provider (Keycloak/Auth0)
3. After auth, JWT issued with role claims
4. Frontend stores token, includes in all API calls
5. Backend validates JWT, extracts role, enforces permissions
6. Session expires after 8 hours (configurable)

## Team Structure (Example)

```
Project: "E-Commerce Platform"
├── Owner: CTO
├── Team Lead (Backend): Senior Dev A
├── Team Lead (Frontend): Senior Dev B
├── Developers: Dev C, Dev D, Dev E
├── Product Manager: PM F
├── Designer: Designer G
└── QA: QA Engineer H
```
