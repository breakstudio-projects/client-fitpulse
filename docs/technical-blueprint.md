# Technical Blueprint — FitPulse

---

## Part 1: System Architecture



# System Architecture Document — FitPulse

**Prepared by:** Break Studio — Architecture
**Date:** June 2025
**Status:** Draft — Pre-Discovery Architecture Scaffold
**Architect:** Senior Solutions Architect, Break Studio

---

## ⚠️ PREFACE: BUILDING ON INCOMPLETE FOUNDATIONS

I need to be transparent. The strategy brief and MVP scoping documents that should feed this architecture document are both empty scaffolds. No defined users, no confirmed features, no validated problem statement, no load expectations.

**However** — the project name "FitPulse" and the specified tech stack (Next.js + Supabase + Vercel) give us enough to build a rigorous, opinionated architecture scaffold for what is almost certainly a **fitness/health tracking application**. This document assumes that scope and builds a production-grade architecture around it.

**The moment discovery inputs land, we adjust. Not before.**

Every assumption is flagged with `[ASSUMPTION]` so nothing slips through as validated when it isn't.

---

## 1. ARCHITECTURE OVERVIEW

### 1.1 High-Level Architecture Diagram Description

```
┌─────────────────────────────────────────────────────────────────────┐
│                          CLIENT LAYER                               │
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │              Next.js Application (Vercel)                    │   │
│  │                                                              │   │
│  │  ┌──────────┐  ┌──────────────┐  ┌─────────────────────┐   │   │
│  │  │  React   │  │  Server      │  │  Static/ISR Pages   │   │   │
│  │  │  SPA     │  │  Components  │  │  (Marketing, Help)  │   │   │
│  │  │  (App)   │  │  (RSC)       │  │                     │   │   │
│  │  └────┬─────┘  └──────┬───────┘  └─────────────────────┘   │   │
│  │       │               │                                      │   │
│  │       └───────┬───────┘                                      │   │
│  │               │                                               │   │
│  │    ┌──────────▼──────────┐                                   │   │
│  │    │  Supabase Client    │                                   │   │
│  │    │  (@supabase/ssr)    │                                   │   │
│  │    └──────────┬──────────┘                                   │   │
│  └───────────────┼──────────────────────────────────────────────┘   │
│                  │                                                   │
└──────────────────┼───────────────────────────────────────────────────┘
                   │ HTTPS (REST + Realtime WebSocket)
                   │
┌──────────────────▼───────────────────────────────────────────────────┐
│                       SUPABASE PLATFORM LAYER                        │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────────────┐  │
│  │  Supabase    │  │  PostgREST   │  │  Supabase Realtime        │  │
│  │  Auth        │  │  (Auto API)  │  │  (WebSocket Subscriptions)│  │
│  │  (GoTrue)    │  │              │  │                           │  │
│  └──────┬───────┘  └──────┬───────┘  └─────────────┬─────────────┘  │
│         │                 │                         │                │
│  ┌──────▼─────────────────▼─────────────────────────▼─────────────┐  │
│  │                    PostgreSQL Database                          │  │
│  │                                                                │  │
│  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌────────────┐ │  │
│  │  │  Users &   │ │  Workouts  │ │  Health    │ │  Social /  │ │  │
│  │  │  Profiles  │ │  & Plans   │ │  Metrics   │ │  Community │ │  │
│  │  └────────────┘ └────────────┘ └────────────┘ └────────────┘ │  │
│  │                                                                │  │
│  │  ┌─────────────────────────────────────────────────────────┐  │  │
│  │  │  Row Level Security (RLS) Policies — ALL TABLES         │  │  │
│  │  └─────────────────────────────────────────────────────────┘  │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────┐  ┌─────────────────────────────────┐    │
│  │  Supabase Edge         │  │  Supabase Storage               │    │
│  │  Functions (Deno)      │  │  (Profile pics, workout media)  │    │
│  │                        │  │                                  │    │
│  │  • Webhook handlers    │  │  • Bucket-level policies        │    │
│  │  • 3rd-party API proxy │  │  • Image transformations        │    │
│  │  • Complex business    │  │                                  │    │
│  │    logic               │  │                                  │    │
│  └────────────────────────┘  └─────────────────────────────────┘    │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
                   │
                   │ (Edge Functions → External)
                   ▼
┌──────────────────────────────────────────────────────────────────────┐
│                    THIRD-PARTY SERVICES                              │
│                                                                      │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────────┐  │
│  │  Wearable    │  │  Payment     │  │  Notification Services   │  │
│  │  APIs        │  │  Provider    │  │  (Email / Push)          │  │
│  │  [ASSUMPTION]│  │  [ASSUMPTION]│  │                          │  │
│  │  Apple Health│  │  Stripe      │  │  Resend (email)          │  │
│  │  Google Fit  │  │              │  │  FCM / APNs (push)       │  │
│  └──────────────┘  └──────────────┘  └──────────────────────────┘  │
│                                                                      │
└──────────────────────────────────────────────────────────────────────┘
```

### 1.2 Architecture Pattern: **Serverless-First with Managed Backend (BaaS)**

**Pattern:** Backend-as-a-Service (Supabase) + Serverless Frontend (Vercel) + Edge Functions for custom logic.

**This is NOT microservices. This is NOT a monolith. This is a deliberately chosen third path.**

#### Justification

| Factor | Decision Rationale |
|---|---|
| **Team size** | `[ASSUMPTION]` Small team (2–5 devs). Microservices would be organizational suicide. A monolith would be fine, but Supabase gives us the monolith's simplicity with managed infrastructure. |
| **Time to market** | Supabase eliminates 60–70% of backend boilerplate (auth, CRUD APIs, realtime, storage). For a product with no validated market fit yet, speed of iteration is the only metric that matters. |
| **Operational overhead** | Zero server management. No Kubernetes. No Docker in production. No DevOps hire needed at this stage. Vercel and Supabase handle scaling, deployments, SSL, CDN, and database management. |
| **Escape hatch** | The database is standard PostgreSQL. If we outgrow Supabase, we migrate the database to any managed Postgres (RDS, Cloud SQL, Neon) and swap PostgREST for a custom API layer. No vendor lock-in on the data layer. |
| **Cost profile** | Both Vercel and Supabase have generous free tiers and predictable scaling costs. For a pre-PMF product, this keeps burn rate near zero until we have users. |

**When this pattern STOPS working:**
- When we need >10 distinct background job types running concurrently (move to dedicated workers)
- When Edge Function cold starts become unacceptable for latency-critical paths (move to always-on compute)
- When we exceed Supabase's connection limits under sustained high concurrency (introduce PgBouncer or move to dedicated Postgres)

We are not there. We plan for it. We don't build for it today.

---

## 2. TECH STACK

### 2.1 Frontend: Next.js 14+ (App Router) + React 18+

**Deployed on: Vercel**

| Aspect | Detail |
|---|---|
| **Framework** | Next.js 14+ with App Router |
| **UI Library** | React 18+ with Server Components |
| **Styling** | Tailwind CSS + shadcn/ui `[ASSUMPTION]` |
| **State Management** | React Server Components for server state; Zustand or React Context for lightweight client state `[ASSUMPTION]` |
| **Data Fetching** | `@supabase/ssr` for server-side Supabase calls; TanStack Query for client-side caching/revalidation `[ASSUMPTION]` |
| **Forms** | React Hook Form + Zod for validation |

#### Justification

**Why Next.js over plain React SPA, Remix, or Astro:**

1. **Server Components (RSC):** Fitness dashboards are data-heavy. RSC lets us fetch workout history, metrics, and plans on the server, ship zero JavaScript for read-heavy views, and dramatically reduce client bundle size. A gym user on a 4G connection in a basement gym will thank us.

2. **Hybrid rendering model:** Marketing pages → Static (ISR). Dashboard → Server-rendered with streaming. Real-time workout tracking → Client-side SPA with WebSocket. One framework handles all three without architectural gymnastics.

3. **Vercel deployment:** Next.js on Vercel is a zero-config deployment with edge middleware, automatic image optimization, incremental static regeneration, and preview deployments for every PR. The DX velocity multiplier is real.

4. **Ecosystem maturity:** Auth helpers (`@supabase/ssr`), middleware patterns, and the React ecosystem all have first-class Next.js support. We're not fighting the tooling.

**What we're trading off:** Vendor affinity with Vercel. If deployment costs become problematic, Next.js can be self-hosted, but we lose some Vercel-specific optimizations. Acceptable trade-off for current stage.

### 2.2 Backend: Supabase

| Component | Technology | Purpose |
|---|---|---|
| **Database** | PostgreSQL 15 | Primary data store. Full SQL, JSON support, extensions ecosystem. |
| **Auth** | Supabase Auth (GoTrue) | User registration, login, OAuth, JWT management, MFA |
| **API** | PostgREST (auto-generated) | Instant RESTful API from database schema. No backend code for CRUD. |
| **Realtime** | Supabase Realtime | WebSocket subscriptions for live workout tracking, social feeds `[ASSUMPTION]` |
| **Edge Functions** | Deno runtime | Custom server

---

## Part 2: Database Schema

```sql
-- ============================================================================
-- Break Studio - Complete PostgreSQL Schema for Supabase
-- ============================================================================
-- Since the project is "undefined", I'm building a comprehensive creative
-- studio/project management platform schema that covers common MVP features:
-- Users, Teams, Projects, Tasks, Assets, Comments, Notifications, and Audit Log.
-- ============================================================================

-- ============================================================================
-- ENUMS
-- ============================================================================

-- Project status lifecycle
CREATE TYPE project_status AS ENUM (
    'draft',
    'active',
    'on_hold',
    'completed',
    'archived',
    'cancelled'
);

-- Task status lifecycle
CREATE TYPE task_status AS ENUM (
    'backlog',
    'todo',
    'in_progress',
    'in_review',
    'done',
    'cancelled'
);

-- Task priority levels
CREATE TYPE task_priority AS ENUM (
    'critical',
    'high',
    'medium',
    'low',
    'none'
);

-- Team member roles within a team
CREATE TYPE team_role AS ENUM (
    'owner',
    'admin',
    'member',
    'viewer',
    'guest'
);

-- Asset types for uploaded files
CREATE TYPE asset_type AS ENUM (
    'image',
    'video',
    'audio',
    'document',
    'design_file',
    'archive',
    'other'
);

-- Notification delivery status
CREATE TYPE notification_status AS ENUM (
    'unread',
    'read',
    'dismissed'
);

-- Invitation status
CREATE TYPE invitation_status AS ENUM (
    'pending',
    'accepted',
    'declined',
    'expired',
    'revoked'
);

-- Audit log action categories
CREATE TYPE audit_action AS ENUM (
    'create',
    'update',
    'delete',
    'restore',
    'login',
    'logout',
    'invite',
    'join',
    'leave',
    'role_change',
    'status_change',
    'export',
    'import',
    'permission_change'
);

-- ============================================================================
-- HELPER FUNCTION: auto-update updated_at timestamp
-- ============================================================================

CREATE OR REPLACE FUNCTION update_updated_at_column()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- ============================================================================
-- TABLE: user_profiles
-- Extends Supabase auth.users with application-specific profile data.
-- One-to-one relationship with auth.users via id.
-- ============================================================================

CREATE TABLE user_profiles (
    id              UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
    email           TEXT NOT NULL,
    full_name       TEXT,
    display_name    TEXT,
    avatar_url      TEXT,
    bio             TEXT,
    phone           TEXT,
    timezone        TEXT DEFAULT 'UTC',
    locale          TEXT DEFAULT 'en',
    metadata        JSONB DEFAULT '{}'::jsonb,
    onboarding_completed BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    last_seen_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE user_profiles IS 'Extended user profiles linked 1:1 with Supabase auth.users';

CREATE INDEX idx_user_profiles_email ON user_profiles(email);
CREATE INDEX idx_user_profiles_display_name ON user_profiles(display_name);
CREATE INDEX idx_user_profiles_is_active ON user_profiles(is_active);
CREATE INDEX idx_user_profiles_last_seen_at ON user_profiles(last_seen_at DESC);

CREATE TRIGGER trg_user_profiles_updated_at
    BEFORE UPDATE ON user_profiles
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- TABLE: teams
-- Organizational unit that groups users and projects together.
-- ============================================================================

CREATE TABLE teams (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    description     TEXT,
    logo_url        TEXT,
    owner_id        UUID NOT NULL REFERENCES auth.users(id) ON DELETE RESTRICT,
    metadata        JSONB DEFAULT '{}'::jsonb,
    is_personal     BOOLEAN DEFAULT FALSE,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE teams IS 'Teams/organizations that group users and projects';

CREATE INDEX idx_teams_slug ON teams(slug);
CREATE INDEX idx_teams_owner_id ON teams(owner_id);
CREATE INDEX idx_teams_is_active ON teams(is_active);

CREATE TRIGGER trg_teams_updated_at
    BEFORE UPDATE ON teams
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- TABLE: team_members
-- Junction table for many-to-many relationship between users and teams.
-- Tracks role and membership status.
-- ============================================================================

CREATE TABLE team_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    role            team_role NOT NULL DEFAULT 'member',
    joined_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    invited_by      UUID REFERENCES auth.users(id) ON DELETE SET NULL,
    is_active       BOOLEAN DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(team_id, user_id)
);

COMMENT ON TABLE team_members IS 'Membership join table linking users to teams with roles';

CREATE INDEX idx_team_members_team_id ON team_members(team_id);
CREATE INDEX idx_team_members_user_id ON team_members(user_id);
CREATE INDEX idx_team_members_role ON team_members(role);
CREATE INDEX idx_team_members_is_active ON team_members(team_id, is_active);

CREATE TRIGGER trg_team_members_updated_at
    BEFORE UPDATE ON team_members
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- TABLE: team_invitations
-- Pending invitations to join a team, tracked separately for audit.
-- ============================================================================

CREATE TABLE team_invitations (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    email           TEXT NOT NULL,
    role            team_role NOT NULL DEFAULT 'member',
    invited_by      UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    status          invitation_status NOT NULL DEFAULT 'pending',
    token           TEXT NOT NULL UNIQUE DEFAULT gen_random_uuid()::text,
    accepted_by     UUID REFERENCES auth.users(id) ON DELETE SET NULL,
    expires_at      TIMESTAMPTZ NOT NULL DEFAULT (NOW() + INTERVAL '7 days'),
    responded_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

COMMENT ON TABLE team_invitations IS 'Invitations for users to join teams, with expiry and status tracking';

CREATE INDEX idx_team_invitations_team_id ON team_invitations(team_id);
CREATE INDEX idx_team_invitations_email ON team_invitations(email);
CREATE INDEX idx_team_invitations_token ON team_invitations(token);
CREATE INDEX idx_team_invitations_status ON team_invitations(status);
CREATE INDEX idx_team_invitations_expires_at ON team_invitations(expires_at);

CREATE TRIGGER trg_team_invitations_updated_at
    BEFORE UPDATE ON team_invitations
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- TABLE: projects
-- Core entity representing a body of work within a team.
-- ============================================================================

CREATE TABLE projects (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL,
    description     TEXT,
    status          project_status NOT NULL DEFAULT 'draft',
    cover_image_url TEXT,
    color           TEXT DEFAULT '#6366f1',
    owner_id        UUID NOT NULL REFERENCES auth.users(id) ON DELETE RESTRICT,
    start_date      DATE,
    target_end_date DATE,
    actual_end_date DATE,
    metadata        JSONB DEFAULT '{}'::jsonb,
    is_public       BOOLEAN DEFAULT FALSE,
    is_archived     BOOLEAN DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(team_id, slug)
);

COMMENT ON TABLE projects IS 'Projects belonging to a team, the primary container for tasks and assets';

CREATE INDEX idx_projects_team_id ON projects(team_id);
CREATE INDEX idx_projects_owner_id ON projects(owner_id);
CREATE INDEX idx_projects_status ON projects(status);
CREATE INDEX idx_projects_team_slug ON projects(team_id, slug);
CREATE INDEX idx_projects_is_archived ON projects(is_archived);
CREATE INDEX idx_projects_created_at ON projects(created_at DESC);

CREATE TRIGGER trg_projects_updated_at
    BEFORE UPDATE ON projects
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- TABLE: project_members
-- Tracks which users have access to specific projects (beyond team membership).
-- ============================================================================

CREATE TABLE project_members (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    user_id         UUID NOT NULL REFERENCES auth.users(id) ON DELETE CASCADE,
    role            team_role NOT NULL DEFAULT 'member',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(project_id, user_id)
);

COMMENT ON TABLE project_members IS 'Per-project membership for fine-grained access control beyond team level';

CREATE INDEX idx_project_members_project_id ON project_members(project_id);
CREATE INDEX idx_project_members_user_id ON project_members(user_id);

CREATE TRIGGER trg_project_members_updated_at
    BEFORE UPDATE ON project_members
    FOR EACH ROW EXECUTE FUNCTION update_updated_at_column();

-- ============================================================================
-- TABLE: tags
-- Reusable tags scoped to a team for categorizing tasks and assets.
-- ============================================================================

CREATE TABLE tags (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    team_id         UUID NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    name            TEXT NOT NULL,
    color           TEXT DEFAULT '#9ca3af',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    UNIQUE(team_id, name)
);

COMMENT ON TABLE tags IS 'Team-scoped tags for categorizing tasks, projects, and assets';

CREATE INDEX idx_tags_team_id ON tags(team_id);

-- ============================================================================
-- TABLE: tasks
-- Individual work items within a project. Supports hierarchy via parent_id.
-- ============================================================================

CREATE TABLE tasks (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id      UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    parent_id       UUID REFERENCES tasks(id) ON DELETE SET NULL,
    title           TEXT NOT NULL,
    description     TEXT,
    status          task_status NOT NULL DEFAULT 'backlog',
    priority        task_priority NOT NULL DEFAULT 'none',
    assignee_id     UUID REFERENCES auth.users(id) ON DELETE SET NULL,
    reporter_id     
```

---

## Part 3: API Specification



# API Specification

## Overview

This document provides the complete API specification for the application built on Supabase. Since the project context is undefined, this specification establishes a **comprehensive, production-ready API foundation** covering authentication, user management, core CRUD resources, file storage, notifications, webhooks, and administrative operations.

---

## Table of Contents

1. [Architecture & Conventions](#architecture--conventions)
2. [API Versioning Approach](#api-versioning-approach)
3. [Rate Limiting Strategy](#rate-limiting-strategy)
4. [Authentication Flows](#authentication-flows)
5. [Feature Area: User Management](#feature-area-user-management)
6. [Feature Area: Core Resources](#feature-area-core-resources)
7. [Feature Area: File Storage](#feature-area-file-storage)
8. [Feature Area: Notifications](#feature-area-notifications)
9. [Feature Area: Admin & Analytics](#feature-area-admin--analytics)
10. [Feature Area: Webhooks](#feature-area-webhooks)
11. [Third-Party API Integrations](#third-party-api-integrations)
12. [Error Code Reference](#error-code-reference)

---

## Architecture & Conventions

### Base URLs

| Layer | Base URL |
|---|---|
| Supabase REST (PostgREST) | `https://<project-ref>.supabase.co/rest/v1` |
| Supabase Auth | `https://<project-ref>.supabase.co/auth/v1` |
| Supabase Storage | `https://<project-ref>.supabase.co/storage/v1` |
| Supabase Realtime | `wss://<project-ref>.supabase.co/realtime/v1` |
| Edge Functions | `https://<project-ref>.supabase.co/functions/v1` |

### Common Headers

```
apikey: <SUPABASE_ANON_KEY>          # Always required
Authorization: Bearer <JWT>           # Required for authenticated endpoints
Content-Type: application/json
Prefer: return=representation         # For POST/PATCH to return created/updated row
```

### Naming Conventions

- REST endpoints use **snake_case** table names (auto-generated by PostgREST)
- Edge Functions use **kebab-case** paths
- All timestamps are **ISO 8601 / UTC**
- UUIDs are used for all primary keys
- Pagination via `Range` header or `?offset=0&limit=20`

### Row Level Security (RLS)

All tables have RLS enabled. The JWT `sub` claim is mapped to `auth.uid()` and the `role` claim (via custom claims or app_metadata) controls access.

### Roles

| Role | Description |
|---|---|
| `anon` | Unauthenticated / public |
| `authenticated` | Logged-in user |
| `moderator` | Elevated permissions for content moderation |
| `admin` | Full access to all resources |
| `service_role` | Server-side only (never exposed to client) |

---

## API Versioning Approach

### Strategy: Header-Based + URL Prefix for Edge Functions

| Component | Versioning Method |
|---|---|
| Supabase REST (PostgREST) | Schema-based — use `Accept-Profile` header to target versioned schemas (e.g., `public`, `v2`) |
| Edge Functions | URL prefix: `/functions/v1/my-function`. New major versions get new function deployments (e.g., `my-function-v2`) |
| Third-party Webhooks | Payload includes `"api_version": "2024-01-01"` field |

### Migration Policy

- **Non-breaking changes** (adding nullable columns, new endpoints): deployed immediately
- **Breaking changes**: new schema version or new edge function version, with 90-day deprecation window on the old version
- Deprecation communicated via `Sunset` and `Deprecation` response headers

---

## Rate Limiting Strategy

### Tiers

| Tier | Identifier | Limit | Window |
|---|---|---|---|
| Anonymous | IP address | 60 requests | 1 minute |
| Authenticated (free) | User ID | 120 requests | 1 minute |
| Authenticated (pro) | User ID | 600 requests | 1 minute |
| Admin / Service Role | API Key | 3,000 requests | 1 minute |
| Auth endpoints (sign-up, sign-in) | IP address | 10 requests | 1 minute |
| Password reset | IP + email | 3 requests | 15 minutes |
| File uploads | User ID | 30 requests | 1 minute |
| Webhooks outbound | Per destination | 100 events | 1 minute |

### Implementation

- **Edge Functions**: Rate limiting via an in-memory store (Deno `Map`) with sliding window, backed by a `rate_limits` table for persistence across cold starts.
- **PostgREST**: Enforced at the Supabase project level (built-in) + additional enforcement via a Postgres function `check_rate_limit(user_id, action)` called in RLS policies for sensitive operations.
- **Response Headers**:
  ```
  X-RateLimit-Limit: 120
  X-RateLimit-Remaining: 87
  X-RateLimit-Reset: 1700000000
  ```
- **Exceeded**: Returns `429 Too Many Requests` with `Retry-After` header.

---

## Authentication Flows

All authentication is handled via **Supabase Auth (GoTrue)**.

---

### 1. Sign Up (Email + Password)

**`POST /auth/v1/signup`**

| Field | Value |
|---|---|
| **Description** | Register a new user with email and password |
| **Auth Required** | No (anon key only) |

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "secureP@ssw0rd!",
  "data": {
    "display_name": "Jane Doe",
    "avatar_url": null
  }
}
```

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "aud": "authenticated",
  "role": "authenticated",
  "email": "user@example.com",
  "phone": "",
  "app_metadata": {
    "provider": "email",
    "providers": ["email"]
  },
  "user_metadata": {
    "display_name": "Jane Doe",
    "avatar_url": null
  },
  "identities": [...],
  "created_at": "2024-01-15T10:00:00Z",
  "updated_at": "2024-01-15T10:00:00Z",
  "confirmation_sent_at": "2024-01-15T10:00:00Z"
}
```

**Error Codes:**
| Code | Meaning |
|---|---|
| `400` | Invalid email format or weak password |
| `422` | User already registered |
| `429` | Rate limit exceeded |

---

### 2. Sign In (Email + Password)

**`POST /auth/v1/token?grant_type=password`**

| Field | Value |
|---|---|
| **Description** | Authenticate and receive JWT tokens |
| **Auth Required** | No (anon key only) |

**Request Body:**
```json
{
  "email": "user@example.com",
  "password": "secureP@ssw0rd!"
}
```

**Response `200 OK`:**
```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "expires_in": 3600,
  "expires_at": 1700003600,
  "refresh_token": "v1.MjA...",
  "user": {
    "id": "uuid",
    "email": "user@example.com",
    "app_metadata": { "role": "authenticated" },
    "user_metadata": { "display_name": "Jane Doe" }
  }
}
```

**Error Codes:**
| Code | Meaning |
|---|---|
| `400` | Invalid login credentials |
| `422` | Email not confirmed |
| `429` | Rate limit exceeded |

---

### 3. Sign In (OAuth — Google, GitHub, Apple, etc.)

**`GET /auth/v1/authorize?provider=google&redirect_to=https://app.example.com/callback`**

| Field | Value |
|---|---|
| **Description** | Initiate OAuth flow; redirects to provider |
| **Auth Required** | No |

**Flow:**
1. Client redirects user to the URL above
2. User authenticates with provider
3. Supabase redirects back to `redirect_to` with `#access_token=...&refresh_token=...` in the URL fragment
4. Client extracts tokens from fragment

**Callback Response (URL fragment):**
```
https://app.example.com/callback#access_token=eyJ...&expires_in=3600&refresh_token=v1.Mj...&token_type=bearer&type=signup
```

**Error Codes:**
| Code | Meaning |
|---|---|
| `400` | Unsupported provider |
| `403` | OAuth provider denied access |
| `500` | Provider configuration error |

---

### 4. Token Refresh

**`POST /auth/v1/token?grant_type=refresh_token`**

| Field | Value |
|---|---|
| **Description** | Exchange refresh token for new access token |
| **Auth Required** | No (anon key only) |

**Request Body:**
```json
{
  "refresh_token": "v1.MjA..."
}
```

**Response `200 OK`:**
```json
{
  "access_token": "eyJhbGciOi...",
  "token_type": "bearer",
  "expires_in": 3600,
  "expires_at": 1700007200,
  "refresh_token": "v1.NzQ..."
}
```

**Error Codes:**
| Code | Meaning |
|---|---|
| `400` | Invalid or expired refresh token |
| `401` | Refresh token revoked |

---

### 5. Password Reset Request

**`POST /auth/v1/recover`**

| Field | Value |
|---|---|
| **Description** | Send password reset email |
| **Auth Required** | No |

**Request Body:**
```json
{
  "email": "user@example.com"
}
```

**Response `200 OK`:**
```json
{}
```
> Always returns 200 regardless of whether the email exists (to prevent enumeration).

**Error Codes:**
| Code | Meaning |
|---|---|
| `429` | Rate limit exceeded |

---

### 6. Password Update (after reset)

**`PUT /auth/v1/user`**

| Field | Value |
|---|---|
| **Description** | Update password using the recovery token (sent as access token) |
| **Auth Required** | Yes — Bearer token from recovery link |

**Request Body:**
```json
{
  "password": "newSecureP@ssw0rd!"
}
```

**Response `200 OK`:**
```json
{
  "id": "uuid",
  "email": "user@example.com",
  "updated_at": "2024-01-15T12:00:00Z"
}
```

**Error Codes:**
| Code | Meaning |
|---|---|
| `401` | Invalid or expired recovery token |
| `422` | Password doesn't meet requirements |

---

### 7. Sign Out

**`POST /auth/v1/logout`**

| Field | Value |
|---|---|
| **Description** | Revoke the current session |
| **Auth Required** | Yes — any authenticated user |

**Request Body:** None

**Response `204 No Content`**

---

### 8. Magic Link (Passwordless)

**`POST /auth/v1/magiclink`**

| Field | Value |
|---|---|
| **Description**