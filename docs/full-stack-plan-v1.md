# Full Stack Implementation Plan (v1)

This document captures the approved 2‑page workflow (Dashboard + Gym Detail) and the end‑to‑end engineering delivery plan, incorporating the naming decision to use "Needs Files".

---
## 1. Core Goals
Deliver a 2‑page app:
1. Dashboard (Month + Gyms overview).
2. Gym Detail (assignments/posts workflow with uploads, validation, review, posting states).

Primary success criteria:
- Coaches can complete uploads with zero written instruction.
- Reviewers can enforce guard rails (spec compliance, safety).
- Status integrity (no illegal transitions).
- Fast page transitions (<250ms perceived after data cached).
- Secure & auditable (every status/file change traceable).

Non‑functional targets:
- >99.5% availability.
- P95 API <300ms (non-upload).
- Horizontal scalability with stateless API nodes.
- CI/CD: green tests + lint mandatory.

---
## 2. Stack Selection (Recommended)
Frontend:
- React + Next.js (or Vite + React as fallback) with TypeScript.
- React Query for server cache.
- Tailwind CSS + component lib (shadcn/ui or Chakra).
- Uploads: direct S3 multipart (pre‑signed) or tus.io wrapper.
- Testing: Vitest + React Testing Library + Playwright (E2E).

Backend:
- Node.js (18+ LTS) with Fastify (or NestJS if preferred).
- TypeScript everywhere.
- PostgreSQL primary, Redis cache & queues.
- S3 for asset storage.
- BullMQ workers (Redis) for metadata extraction & safety check stub.
- Auth0 / Cognito / custom JWT (role claims).
- Prisma ORM; Zod for validation (shared package).
- Observability: OpenTelemetry + (Grafana / Honeycomb) + pino logs.

Optional Phase 2:
- WebSockets (Socket.IO) or SSE for real‑time updates (start with polling 15s).

---
## 3. High‑Level Architecture
Browser → API (Fastify) → PostgreSQL | Redis | S3 | BullMQ Workers → (Future AI Service)
Telemetry sidecar: OTEL Collector → Metrics/Tracing stack.

---
## 4. Domain Model (Tables)
plans (id, month, created_at)
gyms (id, name, slug, instagram_url, facebook_url, business_suite_url, created_at)
plan_gyms (id, plan_id, gym_id) if needed
assignments (id, gym_id, plan_id, title, slug, content_type, due_date, required_photo_count, required_video_count, status, channel_status_ig, channel_status_fb, caption_ig, caption_fb, safety_blocked, safety_reason, created_at, updated_at)
assets (id, assignment_id, type, file_url, filename, width, height, duration_seconds, codec, ratio, check_aspect, check_duration, check_filename, notes, created_at)
assignment_events (id, assignment_id, actor_user_id, event_type, from_value, to_value, metadata jsonb, created_at)
comments (id, assignment_id, user_id, body, created_at)
users (id, name, email, role)

Indices: assignments(plan_id,gym_id); assets(assignment_id); comments(assignment_id, created_at); assignment_events(assignment_id, created_at)

---
## 5. Status Machine
Not Started → In Progress → Submitted → (Changes Requested ↺ In Progress) → Approved → Posted

Guards:
- Submit: all required assets uploaded; all checks true; safety_blocked=false.
- Approve: revalidate same set server-side.
- Posted: only if status Approved AND a channel switch set to Posted.

Implement validateTransition(current, next, context) returning success|error code.

---
## 6. Filename Pattern Validation
Regex:
^([A-Z0-9]+)_(\d{6})_([a-z0-9-]+)_(Photo|Clip)(\d{2})\.(jpg|jpeg|png|mp4|mov)$

Rules:
- yyyymm must match assignment plan month.
- Provide error codes: FILENAME_INVALID, SPEC_RATIO, SPEC_DURATION, SPEC_DIMENSIONS, STATUS_GUARD_FAIL.

---
## 7. API Surface (Initial)
Auth:
- GET /me
Plans/Dashboard:
- GET /plans?month=2025-08 → plan summary + gym stats
- GET /plans/:planId/gyms (optional if not embedded)
Assignments:
- GET /gyms/:gymId/assignments?planId=
- GET /assignments/:id
- PATCH /assignments/:id/status { nextStatus }
- PATCH /assignments/:id/channel-status { ig?, fb? }
- PATCH /assignments/:id/captions { caption_ig, caption_fb }
Assets:
- POST /assignments/:id/assets/init-upload { filename, type }
- PUT (direct to S3)
- POST /assignments/:id/assets/complete { uploadId }
- PATCH /assets/:id { filename, notes }
- DELETE /assets/:id (pre-approval only)
Comments:
- GET /assignments/:id/comments
- POST /assignments/:id/comments { body }
Events:
- GET /assignments/:id/events
Utilities:
- GET /assignments/:id/validation

Unified error JSON.

---
## 8. Frontend Component Map
DashboardPage: MonthPicker, GymGrid(GymCard*)
GymCard: DonutMeter, StatsPills (Approved, Submitted, Needs Files)
GymDetailPage: TopSummaryBar, PostCardRail(PostCard), UploadTray(FileDropzone, AssetList/AssetTile), AssignmentDrawer (Tabs: Overview, Captions, Instructions, Comments)
Shared: Tooltip, ModalConfirm, Toast, Donut, GuardedActionButton.

React Query Keys: ['plans', month]; ['gymAssignments', gymId, planId]; ['assignment', id]; ['assets', assignmentId]; ['comments', assignmentId].

---
## 9. File Upload Pipeline
1. Local pre-check (extension, ratio, duration) via FileReader & video element.
2. init-upload → pre-signed URL + placeholder asset (PROCESSING).
3. PUT to S3.
4. complete → enqueue job.
5. Worker extracts metadata, runs checks, updates flags.
6. Client polls or receives push event to refresh asset list/checklist.

Checks: aspect (1:1 ±5% photos; 9:16 ±5% videos), duration (10–60s videos), filename regex.

---
## 10. Safety Check Stub
analyzeAsset(assetId) → { safe: true }. Future: CV model sets safety_blocked + reason.

---
## 11. Caching Strategy
- Gym assignments list cached 60s in Redis (key month+gym).
- Donut percentages precomputed or on-demand cached.
- Invalidate on mutation via pub/sub.

---
## 12. Analytics / Events
Client: ui.drawer.open, ui.upload.drop, ui.status.change, ui.caption.copy, ui.comment.add.
Server: assignment_events table (authoritative audit).

---
## 13. Auth & RBAC
Roles: COACH, REVIEWER, ADMIN.
COACH: upload, captions (pre-approval), submit, comment, mark channel posted (after approval).
REVIEWER: + approve / request changes.
ADMIN: + manage plans, gyms, assignments.

---
## 14. Error Handling Contract
HTTP codes: 400,401,403,404,409,422,500.
Format: { error: { code, message, field? } }

---
## 15. Concurrency & Locking
- optimistic version (updated_at or version column) on assignments.
- Approve re-check assets; deny if asset updated after approval check timestamp.

---
## 16. Testing Strategy
Unit: status transitions, filename parser, spec checks.
Integration: API endpoints.
E2E: Playwright (upload→submit→approve→post).
Performance: k6 (burst uploads & status changes).
Security: role-based access tests.

---
## 17. Accessibility
- Keyboard navigable drawer & tabs.
- Donut has aria-label with percentage.
- Contrast for status pills; tooltips accessible.

---
## 18. Deployment & Environments
Envs: dev, staging, prod.
Infra: (early) Render/Fly.io; (mature) AWS (ECS Fargate, RDS, Elasticache, S3, CloudFront).
CI: lint → test → build → migrate → deploy → smoke tests.

---
## 19. Incremental Delivery Roadmap
M1 Foundations (Week1-2)
M2 Dashboard (Week3)
M3 Assignment Drawer (Week4)
M4 Upload Pipeline (Week5-6)
M5 Review Flow (Week7)
M6 Captions & Channel Switches (Week8)
M7 Comments & Notes (Week9)
M8 Safety Stub & Hardening (Week10)
M9 QA & Perf (Week11)
M10 Launch Prep (Week12)

---
## 20. Repository Structure (Monorepo)
/apps/api
/apps/web
/packages/shared-types
/packages/ui
/packages/validation
/prisma
/infrastructure
/tests (e2e)
/docs

---
## 21. Shared Types & OpenAPI
Use Zod schemas in /packages/validation and generate OpenAPI + TS client to /packages/shared-types.

---
## 22. Logging & Observability
pino logs (requestId). Metrics: assignments_status_count, asset_checks_failed_total, upload_latency_ms. Traces on key flows (/assignments/:id, worker jobs).

---
## 23. Security Considerations
- Pre-signed URL expiry 10m; content-length-range.
- Enforce MIME & disable sniffing.
- Rate limiting (uploads, status changes).
- CORS whitelist.
- Helmet CSP.
- UUIDs only.

---
## 24. Future Enhancements
AI caption generation, realtime updates, bulk approve, TikTok channel, asset re-ordering, plan templates, SLA dashboards.

---
## 25. Definition of Done (Per Feature)
API + tests, UI wired (optimistic + error), accessibility pass, telemetry added, docs updated, QA checklist updated.

---
## 26. Risk Matrix Highlights
Upload performance → multipart + progress UI.
Spec drift → shared Zod schemas.
Status races → optimistic locking.
Scope creep (AI) → feature flags.

---
## 27. Seed Script Outline
- Create plan (current month)
- Create gyms
- Create assignments (sample counts)
- Output role-based test users

---
## 28. Metrics Post-Launch
Cycle time (In Progress→Submitted, Submitted→Approved), % first-pass validation success, reopen rate, failure reason distribution, caption copy events.

---
## 29. Naming Decision
UI & docs use "Needs Files" (replaces "Missing"). Consider aligning API fields (missingCount → needsFilesCount) before first release.

---
## 30. Open Questions
- Align API field naming to "needsFiles"? (Decide before implementing dashboard endpoint.)
- Use Next.js vs Vite? (Decision M1.)

End of v1.
