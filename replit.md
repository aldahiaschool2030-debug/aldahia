# Workspace

## Overview

pnpm workspace monorepo running **نظام الضاحية الذكي** (Aldahia Smart System) — an Arabic / RTL school attendance management platform. Phase 1 (this artifact) is the admin web app. Phase 2 (planned) is a teacher mobile app.

## Stack

- **Monorepo tool**: pnpm workspaces
- **Node.js version**: 24
- **Package manager**: pnpm
- **TypeScript version**: 5.9
- **API framework**: Express 5
- **Database**: PostgreSQL + Drizzle ORM
- **Validation**: Zod (`zod/v4`), `drizzle-zod`
- **API codegen**: Orval (from OpenAPI spec) → React Query hooks
- **Build**: esbuild (CJS bundle) for the API server, Vite for the web app
- **Web app**: React 18 + Vite + wouter + TanStack Query + shadcn/ui + Tailwind + Recharts + jsbarcode
- **Locale**: Arabic (RTL) with Cairo font from Google Fonts

## Artifacts

- `artifacts/api-server` — Express API exposing `/api/dashboard/*`, `/api/students` (incl. `POST /api/students/import` bulk import), `/api/classes`, `/api/teachers`, `/api/attendance`, `/api/reports/*`. Health route at `/api/health`.
- `artifacts/aldahia-admin` — RTL admin web. Pages: Login (logo + school name), Dashboard, Students (incl. Excel import + template download), Student Detail (barcode + popup print + monthly report), Classes, Class Detail (bulk attendance), Teachers, Attendance log, Reports (Excel + PDF export, WhatsApp send per row, school-branded header), Settings (school name, subtitle, address, phone). Auth via simple localStorage gate (default `admin` / `aldahia2026`, defined in `src/lib/auth.ts`).
- `artifacts/mockup-sandbox` — design preview server (canvas only).

## Branding

- School: **مدارس الضاحية الأهلية للبنين** — إحدى مدارس المالكي التعليمية.
- Logo PNG lives in `attached_assets/Logo_AldahiaSchools_boy_-6_*.png`, imported via the `@assets` Vite alias and re-exported from `src/lib/school.ts` (`SCHOOL_LOGO_URL`).
- School name + subtitle are user-editable in Settings (persisted in `localStorage` under `school-settings`); `loadSchool()` / `saveSchool()` wrap that.
- WhatsApp helpers (`toWhatsAppNumber`, `whatsAppLink`) normalize Saudi phone numbers (`05XXXXXXXX` → `9665XXXXXXXX`) and build `https://wa.me/` deep links.

## Reporting / Export

- Excel export: `src/lib/export.ts` `exportRowsToXlsx(rows, filename, sheetName?, colWidths?)` — RTL sheet view enabled.
- PDF export: same file, `exportElementToPdf(element, filename)` — uses dynamic-imported `html2canvas` + `jspdf` to rasterize the rendered Arabic HTML and paginate to A4. This avoids jsPDF's poor native Arabic shaping.
- Barcode print: `student-detail.tsx` opens a popup window with the serialized SVG and auto-prints (no extra deps).

## Database (lib/db/src/schema)

- `classes` — id, name, stage (primary|intermediate|secondary), grade
- `students` — id, studentNumber (unique), fullName, stage, classId → classes, parentPhone, notes, barcode (`ALD-{studentNumber}`)
- `teachers` — id, fullName, email (unique), role (teacher|admin), stage
- `attendance_records` — id, studentId, teacherId, status (present|late|absent), date, recordedAt; UNIQUE(studentId, date) for upsert via `onConflictDoUpdate`

## Authentication

- Two login modes (single page, tabs): admin and teacher.
- Admin: hard-coded local credentials `admin / aldahia2026` (kept in `lib/auth.ts`, no server call). Stored as `AdminSession` in localStorage.
- Teacher: real auth against `POST /api/auth/teacher-login` (route in `artifacts/api-server/src/routes/auth.ts`). Verifies `email` + `password` against `teachers.password_hash` using `verifyPassword` (scrypt). Returns the `Teacher` payload, stored as `TeacherSession`.
- `useAuth()` exposes `session`, `isAuthenticated`, `isAdmin`, `isTeacher`. Layout sidebar uses `teacherNavItems` (just Scan + Attendance log) when teacher-only; admin gets full nav including the new Scan link.
- Routing: in `App.tsx`, teacher-only sessions are restricted to `/scan` and `/attendance`; admins see everything. Teachers landing on `/` are redirected to `/scan`.
- Demo teacher seeded: `teacher@aldahia.edu / teacher2026`, stage `primary`, role `teacher`.

## Barcode scan page (`src/pages/scan.tsx`)

- Auto-focus barcode input, Enter submits. Calls `useScanStudent` (`POST /api/students/scan` with `{ barcode, teacherId }`); 403 = stage mismatch, 404 = unknown barcode.
- On success, optionally auto-commits attendance via `useCreateAttendance` with today's date and the chosen status (present/late/absent). Status buttons mirror the present/late/absent color palette.
- Recent operations log (last 30) shown in a side panel; success/failure both logged. Invalidates `getListAttendanceQueryKey()` after commit.

## Key Commands

- `pnpm run typecheck` — full typecheck across all packages
- `pnpm run build` — typecheck + build all packages
- `pnpm --filter @workspace/api-spec run codegen` — regenerate API hooks and Zod schemas from OpenAPI spec
- `pnpm --filter @workspace/db run push` — push DB schema changes (dev only)
- `pnpm --filter @workspace/api-server run dev` — run API server locally
- `pnpm --filter @workspace/aldahia-admin run dev` — run admin web locally
- `cd artifacts/api-server && pnpm dlx tsx src/seed.ts` — seed initial demo data (no-op if students already exist)

## Conventions

- Generated React Query hooks return data directly (no `.data` envelope). When passing `enabled`, also pass `queryKey` via the corresponding `getXxxQueryKey(...)` helper.
- Mutations: `create.mutate({ data })`, `update.mutate({ id, data })`, `remove.mutate({ id })`, bulk attendance: `bulk.mutate({ data: { date, records: [{studentId,status}] } })`. Always `queryClient.invalidateQueries({ queryKey })` after success.
- Status colors: present → emerald, late → amber, absent → rose. Stage labels: primary → ابتدائي, intermediate → متوسط, secondary → ثانوي. Roles: teacher → معلم, admin → إداري. Helpers in `artifacts/aldahia-admin/src/lib/i18n.ts`.
- All admin pages set `dir="rtl"`; sidebar lives on the right.

See the `pnpm-workspace` skill for workspace structure, TypeScript setup, and package details.
