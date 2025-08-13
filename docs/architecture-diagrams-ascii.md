# ASCII Architecture Diagrams (v1)

This document provides text-drawn (monospace) diagrams of the system, per user request.

---
## 1. High-Level System Architecture

Legend:
[ ] component  |  /----/ dashed boundary = logical tier  |  --> sync  |  ~> async

```
                         +--------------------------+
                         |      Auth Provider       |
                         |   (Auth0 / Cognito / ?)  |
                         +------------+-------------+
                                      |
                                      v  (JWT / OIDC)
+---------------------------------------------------------------+
|                       FRONTEND TIER                           |
|  +--------------------+                                       |
|  |  User Browser      |                                       |
|  +----------+---------+                                       |
|             |  HTTPS (HTML/JS/CSS, API JSON)                  |
|             v                                                 |
|        +---------+        React Query Cache (in-memory)       |
|        | Web App |-----(cache)------+                         |
|        +----+----+                  |                         |
+-------------|-----------------------+-------------------------+
              |
              v
/----------------------------------------------------------------\
|                        API / APP TIER                          |
|  +------------------+      +------------------+                |
|  | Fastify / NodeJS |----->|   Validation &   |                |
|  |  REST Endpoints  |      |  Status Guards   |                |
|  +-----+------+-----+      +------------------+                |
|        |      |                                               |
|        |      | Enqueue job (upload complete)                 |
|        |      +--------------------+                          |
|        |                           v                          |
|        |                      +---------+                     |
|        |                      | Queue   | (BullMQ / Redis)    |
|        |                      +----+----+                     |
|        |                           |  ~> async                |
\----------------------------------------------------------------/
         |                           |
         v                           v
/--------------------\     /------------------------------\
|  DATA & STORAGE    |     |       WORKER TIER            |
|                    |     |  +------------------------+  |
| +--------+  +----+ |     |  |  Worker Processes      |  |
| |Postgres|  |S3  | |<--->|  | (asset metadata,       |  |
| +---+----+  +--+-+ |     |  |  safety checks)        |  |
|     ^         ^ |  |     |  +----+-------------------+  |
|     | SQL      | |  |     |       |                      |
|     |          | |  |     |       | get object (S3)      |
|     |          | |  |     |       v                      |
|     |          | |  |     |  +-----------+               |
|     |          | |  |     |  |  S3       | (shared)      |
|     |          | |  |     |  +-----------+               |
|  +--+---+    +-+-+  |     \------------------------------/
|  |Redis |<-->|API|  |  (cache, pub/sub invalidation)
|  +------+    +----+ |
\----------------------/
         |
         v (logs/traces/metrics export)
+------------------------------------------------+
|      Observability Stack (OTel Collector,      |
|      Metrics DB, Logs Store, Tracing UI)       |
+------------------------------------------------+
```

---
## 2. Monorepo / Package Dependency Graph

```
+-----------------+        +-------------------+
|  apps/web       |------->| packages/ui       |
|                 |------->| packages/shared   |
|                 |------->| packages/validation|
+-----------------+        +-------------------+
        |                         ^
        |                         |
        v                         |
+-----------------+        +-------------------+
|  apps/api       |------->| packages/shared   |
|                 |------->| packages/validation
|                 |------->| prisma (schema)   |
+-----------------+                         |
        |                                    |
        v                                    |
+-----------------+                          |
| tests (e2e)     |--------------------------+
+-----------------+
        |
        v
+-----------------+
| infrastructure  |
+-----------------+
```

---
## 3. Domain Entity Relationship (ERD Simplified)

```
(Plan) (1)─* (Assignment) *─(1) (Gym)
   |                        ^
   |                        |
   | (optional link table)  |
   +----* (Plan_Gym) *------+

[Plan] ──┐
         └─(1)─* [Assignment] *─(1)─[Gym]

[Assignment] (1)─* [Asset]
[Assignment] (1)─* [Comment]
[Assignment] (1)─* [Assignment_Event]

[User] (1)─* [Comment]
[User] (1)─* [Assignment_Event]
```

Detailed Box View:
```
+--------------------+        +-----------------+
|       Plan         |        |       Gym       |
| id (uuid)          |        | id (uuid)       |
| month (date)       |        | name            |
+---------+----------+        +--------+--------+
          | (1)                       | (1)
          |                           |
          | (1)                       |
        (1)*                        (1)*
        +---------------+---------------+
                        |
                   +----v--------------------+
                   |       Assignment        |
                   | id, plan_id, gym_id     |
                   | title, status           |
                   | required_* counts       |
                   +----+---------+----------+
                        |         | (1)* events
          (1)* assets   |         v
                        |  +------------+
                        |  |Assignment_ |
                        |  |  Event     |
                        |  +------------+
                        |
          +-------------+--------------+
          |                            |
     (1)* v                       (1)* v
   +----------+                 +-----------+
   |  Asset   |                 |  Comment  |
   +----------+                 +-----------+
```

---
## 4. Assignment Status State Machine

```
          +--------------+
          | Not Started  |
          +------+-------+
                 |
                 | start / first upload
                 v
          +--------------+
          | In Progress  |
          +------+-------+
                 | submit (all assets & checks ok)
                 v
          +--------------+
          |  Submitted   |
          +--+-------+---+
             |       |
   approve() |       | requestChanges()
             |       v
             |  +-----------+
             |  |  Changes  |
             |  | Requested |
             |  +-----+-----+
             |        |
             | continueWork()
             |        |
             +--------+
                 |
                 v
          +--------------+
          |   Approved   |
          +------+-------+
                 |
                 | markChannelPosted() (per IG/FB)
                 v
          +--------------+
          |    Posted    |
          +--------------+
```

Guards:
- submit: counts OK & all checks pass & !safety_blocked
- approve: revalidate assets still OK
- posted: status Approved & channel_status_X moves to Posted

---
## 5. File Upload & Validation Flow

```
User
  |
  | drag/drop
  v
+----------------+        +------------------+
| Frontend (FE)  |--pre-->| local pre-checks |
+----------------+        +------------------+
          |
          | POST init-upload
          v
     +---------+                +-----------+
     |  API    |--insert row--> | Postgres  |
     +----+----+                +-----------+
          |
          | presigned URL
          v
+----------------+
| Frontend (FE)  |
+--------+-------+
         |
         | PUT (binary)
         v
       +----+
       | S3 |
       +----+
         |
         | POST complete
         v
     +---------+
     |  API    |
     +----+----+
          |
          | enqueue
          v
       +------+
       |Queue |
       +--+---+
          |  ~~~>
          v
     +---------+
     | Worker  |
     +----+----+
          | read file head
          v
         S3
          |
          | metadata -> checks
          v
       Postgres (update asset flags)
          |
          | (optional event insert)
          v
     +---------+
     |  API    |
     +----+----+
          |
          | [POLL] GET /validation or /assignment
          v
     Frontend refresh UI (status / progress)
```

---
## 6. Caching & Invalidation

```
Client -> API -> Redis (MISS) -> Postgres -> Redis SET (TTL 60s) -> API -> Client

Mutation -> API -> Postgres -> Redis PUB invalidation -> Redis DEL key
```

Detailed:
```
          +--------+
          | Client |
          +---+----+
              |
              v
          +--------+         (MISS)      +----------+
          |  API   |-------------------->|  Redis   |
          +---+----+<--------------------+----------+
              |            data (if hit)
              | (fallback)
              v
          +----------+
          | Postgres |
          +-----+----+
                |
                +--> Redis SET key (TTL)
                |
                v
             Return data

Mutation path:
Client -> API -> Postgres
                   |
                   +--> Redis PUB invalidation
                   |
                   Redis DEL key
```

---
## 7. Frontend Component Hierarchy

```
DashboardPage
  ├─ MonthPicker
  └─ GymGrid
       └─ GymCard
           ├─ DonutMeter
           └─ StatusPills (Approved / Submitted / Needs Files)

GymDetailPage
  ├─ TopSummaryBar (Title, Due, Progress Bar, Donut, Social)
  ├─ Tabset
  │    ├─ OverviewPane
  │    ├─ CaptionPane
  │    ├─ InstructionsPane
  │    └─ CommentsPane
  └─ UploadTray
       ├─ FileDropzone
       ├─ AssetList
       │    └─ AssetTile (×N)
       └─ ValidationSummary (computed)

CaptionPane:
  └─ CaptionFields (Instagram, Facebook)

CommentsPane:
  ├─ CommentList
  └─ CommentForm
```

---
## 8. RBAC Matrix

```
Roles: COACH (C), REVIEWER (R), ADMIN (A)

Action                  C   R   A
---------------------------------
Upload Asset            ✓   ✓   ✓
Delete Asset (pre-appr) ✓   ✓   ✓
Submit Assignment       ✓   ✓   ✓
Request Changes         -   ✓   ✓
Approve                 -   ✓   ✓
Edit Captions (pre)     ✓   ✓   ✓
Mark Posted             ✓   ✓   ✓
Create Plan/Gym         -   -   ✓
```

---
## 9. Event / Audit Flow

```
+-------+         +-----+          +-----------+
| User  | ----->  | API | -------> | EventsTbl |
+-------+         +--+--+          +-----------+
                      ^
                      | (worker emits)
                +-----+------+
                |  Worker    |
                +------------+
```

---
## 10. All-in-One Overview

```
+--------------------------------------------------------------------------------------+
|  Browser UI (React)                                                                  |
|   - Dashboard (Plans, Gyms)                                                          |
|   - Gym Detail (Tabs, UploadTray, Captions, Comments)                                |
|        |                                                                             |
|        | JSON over HTTPS                                                             |
|        v                                                                             |
|  API (Fastify, Node) -- Prisma --> Postgres <---- Worker Jobs (metadata, checks)     |
|        |                    ^                                                        |
|        | Redis (cache, pub/sub)  <---- Worker (also uses Redis)                      |
|        |                    |                                                        |
|        |-- pre-signed URLs --> S3 (assets) <--- Worker fetches objects               |
|        |                                                                             |
|        +--> BullMQ Queue --> Worker Processes                                        |
|                                                                                        |
|  Observability: API & Worker emit logs/traces/metrics → OTel Collector → Monitoring  |
|  Auth Provider: issues JWT → used by Browser + API guards                            |
+--------------------------------------------------------------------------------------+
```

---
## 11. Changelog
v1: Initial ASCII diagrams added.

---
End of file.