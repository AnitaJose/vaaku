# CLAUDE.md

Guidance for Claude Code when working in this repository. Read `SPEC.md` first for what to build.

## Project

Speech Data: an offline-first mobile app for a speech therapist to manage child patients, their case history, assessment results and daily therapy sessions. Version 1 is a single-user test app. All data stays on the phone. There is no server, no API, no account system and no network calls.

## Stack

- Expo (React Native) with TypeScript in strict mode
- expo-router for navigation (file-based, in `app/`)
- expo-sqlite for storage (the only database)
- expo-secure-store for the PIN hash
- expo-file-system, expo-sharing and expo-document-picker for backup and restore
- expo-crypto for UUIDs and PIN hashing
- No state library at first. Use React state and small hooks that call the repository layer.

## Commands

```
npm install
npx expo start            # dev server, open with Expo Go
npx expo start --clear    # after changing dependencies or config
npx tsc --noEmit          # type check
npm run lint
```

Run `npx tsc --noEmit` and `npm run lint` before saying a task is done.

## Folder layout

```
app/                  routes only (expo-router); keep screens thin
  (auth)/             signup.tsx, unlock.tsx
  (tabs)/             index (Home), patients, reports, settings
  patient/[id]/       index (details with tabs), session/..., edit hub and steps
src/
  db/                 client.ts, migrations.ts, schema, seed.ts
  repo/               one file per table group: patients.ts, sessions.ts, master.ts
  features/           patient-form, session, master-data, backup, auth
  components/         shared UI: Button, Card, Chip, Checkbox, StepHeader, Tabs
  theme/              colors.ts, spacing.ts, typography.ts
  lib/                id.ts, dates.ts, validation.ts
docs/                 design notes
SPEC.md
CLAUDE.md
```

Rules:
- Screens in `app/` only compose components and call hooks. No SQL in screens or components.
- All SQL lives in `src/repo/`. Repository functions are async, typed, and return plain objects.
- Schema changes go through a numbered migration in `src/db/migrations.ts`. Never edit an applied migration; add a new one. Track the version with `PRAGMA user_version`.

## Data rules (important for a later server version)

- Every row has `id` (UUID v4 as text, created on the device), `created_at`, `updated_at` (ISO 8601 UTC text) and `deleted_at` (nullable, soft delete). Do not use auto-increment integer IDs as identifiers.
- Deleting a patient, session or master item is a soft delete (`deleted_at`). Queries must filter `deleted_at IS NULL`.
- Dates of birth and session dates are stored as `YYYY-MM-DD`. Timestamps are ISO 8601 UTC.
- Age is computed from the birth date at display time. Never store age.
- Master data is editable by the user. Seed defaults once on first run; never overwrite user changes.
- Only the patient name is required. Every other field and section is optional.

## Privacy and safety

- This is health data about children. Do not add analytics, crash reporters, logging of patient data, or any network call.
- The app is protected by a PIN. Store only a salted hash, in expo-secure-store.
- Never put patient data in `console.log`, file names, or error messages.
- Backups are the user's responsibility. The backup file is plain JSON, so show a clear warning before sharing it.

## UI and design

- The approved design is the Design canvas for this project (mobile screens, 390px wide). Match its layout, copy and flow. If the code and the design disagree, ask.
- Palette "Navy & Rose": navy `#1E3A5F`, navy hover `#142843`, navy tint `#E8EEF6`, background `#F6F8FB`, border `#E1E6EE`, divider `#E8ECF2`, text `#15233A`, secondary text `#566277`, input border `#7B8CA3`, rose `#8F2F57` on `#F8E8EE` (used for "Recorded"), amber pill `#FDF0DC` / `#8A4B08`, destructive `#9A2B1B`. Define these once in `src/theme/colors.ts`; do not hard-code hex values in components.
- Font: Manrope (load with `@expo-google-fonts/manrope`).
- Cards have 12px radius. Every touch target is at least 44x44.
- Every input has a visible label. Do not rely on placeholder text alone.
- Add patient and Edit patient share the same hub and the same 11 step screens. Keep one implementation, in `src/features/patient-form`.

## Working style

- Build in small vertical slices (one screen with its data layer), and run the app after each slice.
- Follow the build order in `SPEC.md`. Do not start the next milestone before the current one works end to end.
- Prefer simple code over clever code. Avoid adding libraries unless the task needs one; say why when you add one.
- When something in `SPEC.md` is marked "to confirm", pick the simplest option, note it in the final message, and move on.
- Write a few tests for pure logic (age calculation, migrations, backup serialize and restore). Do not aim for full UI test coverage in v1.
