# VybePM — Build Instructions for Claude Code

You are building VybePM, the task orchestration hub for 1000Problems. Read SPEC.md first — it is the source of truth for architecture, data model, API, and UI.

## Step 0: Repository Setup

This repo (`1000Problems/Vybe`) has an old SwiftUI scaffold that needs to be replaced.

1. Delete everything in the repo except `.git/`, `SPEC.md`, and `CLAUDE.md`
2. Initialize a new Next.js 15 project with TypeScript and App Router
3. Create `.gitignore` (Node + Next.js defaults, include `.env*.local`)
4. Commit: "Replace SwiftUI scaffold with Next.js project for VybePM"
5. Push to main

## Step 1: Database & Auth (Phase 1 from SPEC.md)

1. Install `@neondatabase/serverless`
2. Create `src/lib/db.ts` — Neon connection helper with parameterized query wrapper
3. Create `src/lib/auth.ts` — password cookie validation + API key validation middleware. Password check sets HTTP-only cookie for 30 days. API key check via `X-API-Key` header. Either one grants access.
4. Create `src/lib/types.ts` — TypeScript interfaces for Project, Task, Attachment
5. Create `migrations/001_init.sql` with the full schema from SPEC.md. **Important:** prefix all tables with `vybepm_` to avoid conflicts on the shared Neon instance (e.g., `vybepm_projects`, `vybepm_tasks`, `vybepm_attachments`)
6. Create login page at `/login` — single password input, POST to `/api/auth/login`, redirect to `/` on success
7. Add middleware that checks for valid session cookie or API key on all routes except `/login` and `/api/auth/login`

## Step 2: Projects & Tasks API (Phase 1 continued)

1. Build all project API routes from SPEC.md:
   - GET /api/projects (list with task counts)
   - GET /api/projects/[slug] (single project)
   - POST /api/projects (create)
   - PATCH /api/projects/[slug] (update)
2. Build all task API routes:
   - GET /api/projects/[slug]/tasks (list with filters: status, assignee)
   - POST /api/projects/[slug]/tasks (create)
   - PATCH /api/tasks/[id] (update — enforce state machine transitions)
   - DELETE /api/tasks/[id] (hard delete for pending only, reject otherwise)
   - PATCH /api/tasks/[id]/reorder (update sort_order)
3. Seed data: create entries for all 6 projects with correct details:
   - ytcombinator: "YouTube keyword research dashboard", ["Next.js", "TypeScript", "Neon"], "1000Problems/ytcombinator", color "#58a6ff"
   - KitchenInventory: "AI-powered kitchen inventory with voice input", ["Swift", "SwiftUI", "SwiftData"], "1000Problems/KitchenInventory", color "#3fb950"
   - voiceq-api: "Voice-activated task queue API", ["Next.js", "TypeScript", "Neon"], "1000Problems/voiceq-api", color "#d29922"
   - GitMCP: "Local MCP server for native git access", ["Node.js", "TypeScript", "MCP"], "1000Problems/GitMCP", color "#f0883e"
   - VybePM: "Task orchestration hub", ["Next.js", "TypeScript", "Neon"], "1000Problems/Vybe", color "#a371f7"
   - RubberJoints-iOS: "RubberJoints iOS client", ["Swift", "iOS"], "1000Problems/RubberJoints-iOS", color "#f85149"
4. Test every endpoint

## Step 3: UI (Phase 1 continued)

1. Install Tailwind CSS 4
2. Build the home page (`/`) — project grid with cards per SPEC.md
3. Build the project view (`/projects/[slug]`) — task sheet grid
4. Task grid must support:
   - Inline editing (click cell to edit title, dropdowns for type/status/assignee)
   - New task row at bottom (always visible, start typing to create)
   - Status colors as defined in SPEC.md
   - Completed tasks section at bottom (collapsed by default)
5. Task detail expansion (click row to expand inline or slide-in panel)
6. Mobile: task grid becomes card layout on screens < 768px

## Step 4: Google Drive Integration (Phase 2 from SPEC.md)

1. Install `googleapis` package
2. Create `src/lib/drive.ts`:
   - Initialize Google Drive client with service account credentials from env vars
   - `createUploadSession(fileName, mimeType, parentFolderId)` → returns resumable upload URL + file ID
   - `getFileMetadata(fileId)` → returns URL, thumbnail URL, name, size
   - `deleteFile(fileId)` → removes file from Drive
3. Build upload API routes:
   - POST /api/upload/request — validates file name + MIME type, creates Drive upload session, returns signed URL
   - POST /api/upload/complete — takes file ID, fetches metadata from Drive, returns full file info
4. Build attachment API routes:
   - POST /api/tasks/[id]/attachments — links a Drive file ID to a task
   - DELETE /api/attachments/[id] — removes attachment record, optionally deletes from Drive
5. Build `FileUpload.tsx` component:
   - Drag-and-drop zone
   - Clipboard paste handler (Cmd+V)
   - Calls /api/upload/request, uploads to Drive URL, calls /api/upload/complete, then links to task
   - Shows upload progress bar
6. Add attachment thumbnails to task rows
7. Add lightbox for full-size image viewing

## Step 5: Studio & Quick Add (Phase 3 from SPEC.md)

1. Build video studio page (`/projects/[slug]/studio`):
   - Request form: prompt textarea, style dropdown, duration selector, reference image upload
   - Submit creates a task of type `animation` with assignee `cowork`
   - Gallery of completed animation tasks below the form
2. Build quick add page (`/quick`):
   - Minimal form: project dropdown (pre-filled if `?project=` param), title, type, file upload
   - Designed for mobile home screen bookmark
3. Task reordering via drag and drop
4. Completed tasks collapsible section

## Step 6: Executor API (Phase 4 from SPEC.md)

1. GET /api/executor/next — returns next pending task assigned to calling executor
2. PATCH /api/executor/tasks/[id]/pickup — atomic pickup with SELECT FOR UPDATE
3. PATCH /api/executor/tasks/[id]/complete — set to review, include notes + attachment Drive file IDs
4. GET /api/digest?since=YYYY-MM-DD — task changes since date, grouped by project

## Non-Negotiable Rules

- **No ORMs.** Use `sql` tagged templates with @neondatabase/serverless only.
- **No `any` types.** TypeScript strict mode. Every function, parameter, and return value typed.
- **No `exec()` or `eval()`.** Never, for any reason.
- **Validate all inputs server-side.** Every POST/PATCH checks field types, lengths, enum values. Return 400 with clear error message.
- **Enforce state machine.** Status transitions per SPEC.md. Reject invalid transitions with 400.
- **Sanitize file names.** Strip special chars, reject executable MIME types (.exe, .sh, .bat, etc).
- **Parameterized queries only.** Never concatenate user input into SQL strings.
- **No secrets in code.** DATABASE_URL, VYBEPM_API_KEY, VYBEPM_PASSWORD, GOOGLE_SERVICE_ACCOUNT_EMAIL, GOOGLE_SERVICE_ACCOUNT_PRIVATE_KEY, GOOGLE_DRIVE_FOLDER_ID all from env vars.
- **Prefix all tables with `vybepm_`** — shared Neon instance.
- **Mobile responsive.** Task grid → card layout at < 768px. Quick add page must work well on phone.

## What NOT To Do

- Do NOT create user accounts or a users table — password gate only
- Do NOT add WebSocket/real-time features — simple fetch/refetch
- Do NOT add a sidebar navigation — project grid → project view → back
- Do NOT use `dangerouslySetInnerHTML`
- Do NOT add analytics, tracking, or telemetry
- Do NOT store files in Vercel Blob or locally — all files go to Google Drive
- Do NOT create separate upload paths for different file sizes — one pipeline for everything

## Design Direction

- Dark theme by default, no toggle needed
- Color palette: dark backgrounds (#0e1117, #161b22), muted borders (#30363d), light text (#e6edf3)
- Each project gets a unique accent color (stored in DB, used for card borders and headers)
- Typography: system font stack. No custom fonts.
- Component libraries (shadcn etc) are fine if they speed up the build
- Dense layout — productivity tool, not marketing site. Maximize information density.

## Commit Strategy

Commit after each major step:
1. "Replace SwiftUI scaffold with Next.js project"
2. "Add database schema, migrations, seed data, and auth"
3. "Add projects API and tasks API with full CRUD"
4. "Add home page with project grid"
5. "Add project view with task grid and inline editing"
6. "Add Google Drive integration and file upload pipeline"
7. "Add video studio page and quick add"
8. "Add executor API and digest endpoint"

Push to main after each commit. Vercel auto-deploys.

## Environment Setup

Before starting, confirm these env vars are available (ask the user if not):
- `DATABASE_URL` — Neon PostgreSQL connection string (shared instance)
- `VYBEPM_API_KEY` — any secure random string
- `VYBEPM_PASSWORD` — shared password for web UI access
- `GOOGLE_SERVICE_ACCOUNT_EMAIL` — from Google Cloud Console
- `GOOGLE_SERVICE_ACCOUNT_PRIVATE_KEY` — from downloaded JSON key
- `GOOGLE_DRIVE_FOLDER_ID` — ID of the shared VybePM folder in Drive

Note: Google Drive setup can be deferred to Step 4. Steps 0-3 only need DATABASE_URL, VYBEPM_API_KEY, and VYBEPM_PASSWORD.

## Testing

- Test every API endpoint
- Verify state machine rejects invalid transitions
- Verify file upload creates Drive file and returns correct metadata
- Verify SQL injection is prevented (try `'; DROP TABLE vybepm_tasks; --` in inputs)
- Verify password gate blocks unauthenticated access
- Verify mobile layout at 375px width
- Verify large file upload works (test with a 50MB+ file if possible)
