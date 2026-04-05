# VybePM — Task Orchestration Hub for 1000Problems

## Overview

VybePM is the central command center for the 1000Problems portfolio. It replaces voiceq-api's task queue and the dead VybePM iOS scaffold with a web app purpose-built for writing, assigning, tracking, and executing tasks across all 1000Problems projects.

The core insight: tasks aren't just dev work. They include animations, design, content, deployments, reports, and anything else the CEO needs done. VybePM routes each task to the right executor (Angel, Cowork, Claude Code) and tracks it through completion.

## Core Principles

1. **Spreadsheet-speed input.** Adding a task should feel like typing in a cell, not filling out a form. No modals, no friction.
2. **Project-scoped views.** No flat list with a project dropdown. Home screen shows projects, you tap into one and see only that project's tasks.
3. **Images and files are first-class.** Every task can have files attached — screenshots, mockups, videos, reference material, generated output. All files go to Google Drive via signed URL upload. Same path for a 200KB screenshot and a 500MB video.
4. **Two users max.** Angel + one collaborator. No multi-tenant complexity. Simple password gate.

## Architecture

### Tech Stack
- **Framework:** Next.js 15 (App Router)
- **Language:** TypeScript (strict mode)
- **Database:** Neon PostgreSQL (shared free tier instance with other 1000Problems projects)
- **ORM:** None. Raw SQL with parameterized queries via @neondatabase/serverless
- **Hosting:** Vercel (auto-deploy on push to main)
- **File Storage:** Google Drive via service account + signed resumable upload URLs
- **Styling:** Tailwind CSS 4. Component libraries are fine if they speed up the build.

### Why These Choices
- Neon: Already proven in the portfolio (ytcombinator, voiceq-api). Shared free tier to avoid managing multiple instances.
- Google Drive: 15GB free, shareable between 2 people, works for images and videos alike. Service account handles auth headlessly.
- No ORM: Raw SQL is faster to write, easier to debug, and avoids abstraction overhead for a small schema.

### File Storage Architecture

All files (images, videos, PDFs, anything) use a single upload pipeline through Google Drive:

1. **Client requests upload** → calls `POST /api/upload/request` with file name, MIME type, size
2. **VybePM backend authenticates** with Google Drive API using a service account
3. **Backend creates a resumable upload session** in a shared Drive folder (`1000Problems/VybePM/[project-slug]/`)
4. **Backend returns the signed resumable upload URL** to the client
5. **Client uploads directly to Google Drive** using the signed URL — file bytes never touch VybePM's server
6. **Client notifies VybePM** → calls `POST /api/tasks/[id]/attachments` with the Drive file ID
7. **VybePM stores the attachment record** (Drive file ID, link, name, type, size)

This means:
- No file size limits from Vercel (files bypass the serverless function entirely)
- Same flow for a 50KB screenshot and a 2GB video
- VybePM's server only handles metadata (a few hundred bytes per request)
- Both users see all files in the shared Google Drive folder
- Files are served via Google Drive's CDN

### Google Drive Service Account Setup
1. Create a service account in Google Cloud Console (free)
2. Enable the Google Drive API
3. Download the JSON key file
4. Create a shared folder `1000Problems/VybePM` in Angel's Google Drive
5. Share that folder with the service account's email address (gives it write access)
6. Share the same folder with the other collaborator
7. Add the service account credentials to VybePM's environment variables

## Data Model

### projects
```sql
CREATE TABLE projects (
    id              SERIAL PRIMARY KEY,
    name            VARCHAR(100) NOT NULL UNIQUE,     -- slug: "ytcombinator"
    display_name    VARCHAR(200) NOT NULL,             -- "YTCombinator"
    description     TEXT,                              -- short project description
    tech_stack      TEXT[],                            -- ["Next.js", "TypeScript", "Neon"]
    github_repo     VARCHAR(200),                      -- "1000Problems/ytcombinator"
    deploy_url      VARCHAR(500),                      -- "https://ytcombinator.1000problems.com"
    color           VARCHAR(7),                        -- hex color for UI: "#58a6ff"
    sort_order      INTEGER DEFAULT 0,
    is_active       BOOLEAN DEFAULT true,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

### tasks
```sql
CREATE TABLE tasks (
    id              SERIAL PRIMARY KEY,
    project_id      INTEGER NOT NULL REFERENCES projects(id),
    title           VARCHAR(500) NOT NULL,
    description     TEXT,
    task_type       VARCHAR(30) NOT NULL DEFAULT 'dev'
                    CHECK (task_type IN ('dev', 'design', 'animation', 'content', 'deploy', 'report', 'other')),
    priority        INTEGER NOT NULL DEFAULT 2
                    CHECK (priority BETWEEN 1 AND 4),  -- 1=critical, 2=high, 3=medium, 4=low
    status          VARCHAR(30) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'in_progress', 'review', 'checked_in', 'deployed', 'done')),
    assignee        VARCHAR(30) DEFAULT 'angel'
                    CHECK (assignee IN ('angel', 'cowork', 'claude-code')),
    sort_order      INTEGER DEFAULT 0,
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW(),
    started_at      TIMESTAMPTZ,
    completed_at    TIMESTAMPTZ
);

CREATE INDEX idx_tasks_project ON tasks(project_id);
CREATE INDEX idx_tasks_status ON tasks(status);
CREATE INDEX idx_tasks_assignee ON tasks(assignee);
```

### attachments
```sql
CREATE TABLE attachments (
    id              SERIAL PRIMARY KEY,
    task_id         INTEGER NOT NULL REFERENCES tasks(id) ON DELETE CASCADE,
    drive_file_id   VARCHAR(200) NOT NULL,             -- Google Drive file ID
    url             VARCHAR(2000) NOT NULL,            -- Google Drive view/download link
    file_name       VARCHAR(500),
    file_type       VARCHAR(50),                       -- "image/png", "video/mp4", "application/pdf"
    file_size       BIGINT,                            -- bytes (BIGINT for large videos)
    thumbnail_url   VARCHAR(2000),                     -- Drive thumbnail link (auto-generated by Drive)
    caption         TEXT,
    created_by      VARCHAR(30) DEFAULT 'angel',
    created_at      TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_attachments_task ON attachments(task_id);
```

## Status State Machine

```
pending → in_progress       (executor picks up task)
in_progress → review        (work done, ready for Angel to check)
review → checked_in         (approved, pushed to git)
checked_in → deployed       (live on Vercel/production)
deployed → done             (verified, archived)

-- Backward transitions (rejection/rework):
review → in_progress        (needs changes)
deployed → in_progress      (production issue found)

-- Skip transitions (for non-dev tasks):
in_progress → done          (no deploy needed — e.g., animation, report)
review → done               (approved, no deploy step)
```

## Pages & UI

### 1. Home — Project Grid (`/`)

Grid of project cards. Each card shows:
- Project color bar (left edge)
- Project name + short description
- Tech stack badges
- Task counts: pending | in progress | total
- Last activity (from git or task updates)

Click a card → navigate to `/projects/[slug]`

**Layout:** Responsive grid, 2-3 columns on desktop, 1 on mobile. No sidebar.

### 2. Project View — Task Sheet (`/projects/[slug]`)

Top bar: project name, tech badges, GitHub link, deploy link, "back to projects" breadcrumb.

Main area: **task grid** that behaves like a spreadsheet.
- Rows = tasks, sorted by priority then sort_order
- Columns: Priority (color dot), Title (editable inline), Type (dropdown), Status (dropdown with color), Assignee (dropdown), Attachments (thumbnail count), Updated
- Click a row to expand it into a detail panel (slides in from right or expands inline)
- New task: empty row at the bottom, always visible. Click and start typing. Tab through columns.
- Drag to reorder rows (updates sort_order)

**Attachment interaction:**
- Drag-and-drop a file anywhere on a task row → triggers signed URL upload to Drive
- Paste from clipboard (Cmd+V while a task is focused) → same flow
- Click attachment thumbnails → lightbox view (images) or Drive link (videos/other)

**Status colors:**
- pending: gray
- in_progress: blue
- review: yellow
- checked_in: green
- deployed: purple
- done: muted/dim (visually de-emphasized)

**Completed tasks:** Tasks with status `done` collapse into a "Completed" section at the bottom of the list. Expandable, default collapsed. Shows completion date.

### 3. Task Detail (inline expand or `/projects/[slug]/tasks/[id]`)

Full task view with:
- Title (large, editable)
- Description (markdown, editable)
- All metadata (type, priority, status, assignee)
- Attachment gallery (full-size images, video embeds or Drive links)
- Activity log (status changes with timestamps)
- Notes field (for executor to write what was done)

### 4. Video Studio (`/projects/[slug]/studio`)

Dedicated page for requesting video/animation generation per project.

**Layout:**
- Request form at top: prompt text area, style dropdown (cinematic, explainer, social, custom), duration selector, reference image upload (uses same Drive signed URL flow)
- Submit creates a task of type `animation` with status `pending` and assignee `cowork`
- Below the form: gallery of completed video tasks for this project, showing thumbnail, prompt, status, and Drive link

**Flow:**
1. Angel fills out the form, optionally attaches reference images
2. VybePM creates an `animation` task in the database
3. Executor (Cowork) picks up the task, generates the video
4. Executor uploads video to Drive via VybePM's upload API
5. Executor marks task complete with the Drive link
6. Angel sees the result in the studio gallery

### 5. Quick Add (`/quick?project=[slug]`)

Minimal page optimized for mobile bookmarks. Just:
- Project name (pre-filled from URL param, or dropdown if not specified)
- Title input
- Type selector
- File drop zone
- Submit button

Bookmark this on your phone home screen for fast task entry on the go.

## API Endpoints

### Projects
```
GET    /api/projects                    — List all active projects with task counts
GET    /api/projects/[slug]             — Single project with full details
POST   /api/projects                    — Create project
PATCH  /api/projects/[slug]             — Update project
```

### Tasks
```
GET    /api/projects/[slug]/tasks       — List tasks for project (filter by status, assignee)
POST   /api/projects/[slug]/tasks       — Create task
PATCH  /api/tasks/[id]                  — Update task (title, description, status, priority, assignee)
DELETE /api/tasks/[id]                  — Delete task (soft delete or hard delete for pending only)
PATCH  /api/tasks/[id]/reorder          — Update sort_order
```

### File Upload (Google Drive signed URL flow)
```
POST   /api/upload/request              — Request a signed upload URL
  Body: { "file_name": "screenshot.png", "mime_type": "image/png", "file_size": 204800 }
  Returns: { "upload_url": "https://www.googleapis.com/upload/drive/v3/files?uploadType=resumable&...", "file_id": "abc123" }

POST   /api/upload/complete             — Confirm upload completed, get file metadata
  Body: { "file_id": "abc123" }
  Returns: { "file_id": "abc123", "url": "https://drive.google.com/file/d/abc123/view", "thumbnail_url": "...", "file_name": "screenshot.png", "file_size": 204800 }
```

### Attachments
```
POST   /api/tasks/[id]/attachments      — Link a Drive file to a task
  Body: { "drive_file_id": "abc123", "caption": "Bug screenshot" }
DELETE /api/attachments/[id]            — Remove attachment record (optionally delete from Drive too)
```

### Executor API (for Cowork/MCP integration)
```
GET    /api/executor/next               — Get next pending task assigned to caller
PATCH  /api/executor/tasks/[id]/pickup  — Atomically set to in_progress (SELECT FOR UPDATE)
PATCH  /api/executor/tasks/[id]/complete — Set to review with notes and optional attachment Drive file IDs
```

### Digest API (for daily reports)
```
GET    /api/digest?since=2026-04-04     — Summary of all task changes since date, grouped by project
```

## Auth

Simple password gate for 2 users:
- **Web UI:** Single shared password stored as `VYBEPM_PASSWORD` env var. On first visit, user sees a password input. Correct password sets an HTTP-only cookie that lasts 30 days. No accounts, no signup, no user table.
- **API:** `X-API-Key` header matching `VYBEPM_API_KEY` env var. Required for all API calls.
- **Executor API:** Same API key, identifies caller via `X-Executor` header (`cowork` or `claude-code`).
- **Password bypass for API calls:** API key auth and password cookie auth are both valid — having either one grants access. The cookie is for the browser UI, the API key is for programmatic access.

## MCP Integration

VybePM exposes an API that Cowork (via a future MCP tool or direct HTTP) can call to:
1. Poll for tasks assigned to `cowork`
2. Pick up a task (atomic status transition to `in_progress`)
3. Upload files to Drive via the signed URL flow
4. Report completion with notes and attachment Drive file IDs
5. Query digest for daily reports

This replaces the current git-log-scraping approach for daily reports with structured task data.

## Environment Variables
```
DATABASE_URL=postgresql://...@neon.tech/neondb?sslmode=require     # shared Neon instance
VYBEPM_API_KEY=[generated-api-key]
VYBEPM_PASSWORD=[shared-password-for-web-ui]
GOOGLE_SERVICE_ACCOUNT_EMAIL=[service-account@project.iam.gserviceaccount.com]
GOOGLE_SERVICE_ACCOUNT_PRIVATE_KEY=[-----BEGIN PRIVATE KEY-----\n...\n-----END PRIVATE KEY-----]
GOOGLE_DRIVE_FOLDER_ID=[id-of-shared-1000Problems-VybePM-folder]
```

## Project Structure
```
src/
  app/
    page.tsx                          -- Home: project grid
    login/
      page.tsx                        -- Password gate
    projects/
      [slug]/
        page.tsx                      -- Project view: task sheet
        studio/
          page.tsx                    -- Video studio: request + gallery
    quick/
      page.tsx                        -- Quick add (mobile bookmark)
    api/
      projects/
        route.ts                      -- GET, POST projects
        [slug]/
          route.ts                    -- GET, PATCH project
          tasks/
            route.ts                  -- GET, POST tasks
      tasks/
        [id]/
          route.ts                    -- PATCH, DELETE task
          reorder/
            route.ts                  -- PATCH reorder
          attachments/
            route.ts                  -- POST link, GET list
      attachments/
        [id]/
          route.ts                    -- DELETE attachment
      upload/
        request/
          route.ts                    -- POST: get signed Drive upload URL
        complete/
          route.ts                    -- POST: confirm upload, get metadata
      executor/
        next/
          route.ts                    -- GET next task for executor
        tasks/
          [id]/
            pickup/
              route.ts                -- PATCH atomic pickup
            complete/
              route.ts                -- PATCH complete with notes
      digest/
        route.ts                      -- GET task changes since date
  lib/
    db.ts                             -- Neon connection + query helpers
    auth.ts                           -- Password cookie + API key validation
    types.ts                          -- Shared TypeScript types
    drive.ts                          -- Google Drive service account client + signed URL generation
  components/
    ProjectCard.tsx
    TaskGrid.tsx
    TaskRow.tsx
    TaskDetail.tsx
    AttachmentGallery.tsx
    FileUpload.tsx                    -- Unified upload component (signed URL flow)
    StatusBadge.tsx
    InlineEdit.tsx
    VideoStudio.tsx
    QuickAdd.tsx
    PasswordGate.tsx
```

## Non-Negotiable Rules

1. **No ORMs.** Raw SQL with parameterized queries only.
2. **No `any` types.** TypeScript strict mode. Every function typed.
3. **No `exec()` or `eval()`.** Ever.
4. **Validate all inputs.** Every POST/PATCH validates all fields server-side.
5. **State machine enforcement.** Status transitions must be validated — no jumping from `pending` to `done`.
6. **File upload validation.** Check MIME type and sanitize file names before creating Drive upload session. Reject executable file types.
7. **SQL injection prevention.** Always use parameterized queries. Never interpolate user input into SQL strings.
8. **No secrets in code.** All credentials via environment variables. Google service account key NEVER committed.
9. **Responsive.** Must work on mobile. Task grid collapses to card view on screens < 768px.
10. **One upload path.** All files go through the same signed URL → Google Drive pipeline. No special cases for different file sizes.

## Build Priority

### Phase 1: Core (MVP)
1. Database schema + migrations (use existing shared Neon — create VybePM tables with `vybepm_` prefix to avoid conflicts)
2. Password gate (login page + cookie middleware)
3. Projects API + seed data (all 6 1000Problems projects)
4. Tasks API (CRUD + status transitions)
5. Home page with project grid
6. Project view with task grid (inline editing, status changes)
7. New task creation (inline at bottom of grid)

### Phase 2: File Upload
8. Google Drive service account integration (`lib/drive.ts`)
9. Signed URL upload flow (`/api/upload/request` + `/api/upload/complete`)
10. Attachments API (link Drive files to tasks)
11. Drag-and-drop + paste upload on task rows
12. Attachment thumbnails and lightbox

### Phase 3: Studio & Mobile
13. Video studio page (`/projects/[slug]/studio`)
14. Quick add page (`/quick`)
15. Mobile responsive layout
16. Task reordering (drag and drop)
17. Completed tasks section (collapsible)

### Phase 4: Executor Integration
18. Executor API endpoints
19. Task pickup with atomic locking
20. Completion reporting with notes + attachments
21. Digest API for daily reports

## Deployment

- GitHub repo: `1000Problems/Vybe` (repurpose existing repo — delete SwiftUI scaffold, fresh start)
- Vercel project: auto-deploy on push to main
- Domain: TBD (vybepm.1000problems.com or similar)
- Neon database: shared free tier instance (prefix tables with `vybepm_`)
- Google Drive: shared folder `1000Problems/VybePM` with service account + collaborator access
