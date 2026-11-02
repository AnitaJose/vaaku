# SPEC.md: Speech Data, mobile app v1

## 1. Purpose

A mobile app for a speech therapist to keep each child's case history and assessment results, and to record what was done in each daily therapy session. Version 1 is a test app for one therapist. Data is stored only on the phone.

## 2. Scope

In scope for v1:
- Set up a profile and a PIN, and unlock with the PIN
- Patient list with search, view, add, edit and archive
- Add and edit a patient through an 11-step form
- Patient details screen with tabs
- Daily assessment: a checklist of activities for a given day, and a summary once it is saved (with edit)
- Master data lists the therapist can edit: activities, birth history options, milestones, linguistic tests, prelinguistic checklist items
- Backup to a file and restore from a file
- Basic reports

Out of scope for v1 (do not build): server or API, user accounts, multiple users, cloud sync, push notifications, payments, web version, PDF export, photo or audio attachments.

## 3. Users

One speech therapist using one phone. Later versions may add institutions with several therapists, so IDs and timestamps are designed for sync (see section 6).

## 4. Screens

The design canvas is the source of truth for layout. This list is the source of truth for behaviour.

### 4.1 Sign up (first launch only)
Therapist name, clinic or practice name (optional), a 4 to 6 digit PIN and a confirmation. Saves the profile, stores the PIN hash, seeds the master data, then goes to Home.

### 4.2 Unlock
Enter the PIN. Wrong PIN shows an error; after 5 wrong attempts, wait 30 seconds. Optional: biometric unlock if the device supports it (to confirm). No "forgot PIN" in v1; the only way out is to erase the app data, so Sign up must warn about this and point to Backup.

### 4.3 Home
Greeting with the therapist's name, today's date, a short summary (patients, sessions recorded today) and quick actions: Add patient, Open patient list. Shows today's sessions recorded and patients without a session today (to confirm).

### 4.4 Patient list (tab)
Search by name or patient ID. Each row shows name, age, last session date and a status tag. Tap a row for details. A "+" button starts Add patient. Archived patients are hidden by default with a filter to show them.

### 4.5 Patient details
Fixed top card: initials, name, patient code, age, sex, status, a **Daily assessment** button and an **Edit** button. Below it, four tabs:
- **Overview**: basic details, concerns, family, observations
- **History**: birth history, development history, educational history, medical history
- **Assessment**: linguistics tests and results, prelinguistics checklist, skills and senses
- **Sessions**: list of daily assessments; tap one to open its summary

Sections with no data show a short "Not added" line with a link to the matching edit step.

### 4.6 Add and edit patient
A section list (hub) plus one screen per section. The same screens are used for Add and Edit. Only the name is required to save. A section can be filled in now or later. The hub shows which sections have data. Each step shows "Step n of 11", a progress bar, Back and Next, and a grid icon that returns to the hub. The last step's button is **Save patient**.

| Step | Section | Group | Content |
|---|---|---|---|
| 1 | Basic details | Basics | Name (required), birth date, age (computed, shown read-only), sex, address |
| 2 | Family | Basics | Father: name, birth date. Mother: name, birth date. Siblings: any number of rows with name and birth date, add and remove |
| 3 | Concerns | Basics | One text area |
| 4 | Birth history | History | Quick-select chips from master data (multi-select), plus a notes text area. "Edit list" opens master data |
| 5 | Development history | History | Milestones from master data: tick if reached, and enter the age in months |
| 6 | Educational history | History | One text area |
| 7 | Medical history | History | One text area |
| 8 | Linguistics | Assessment | List of tests from master data. Tick a test to reveal Result (text) and Date done. "Edit list" opens master data |
| 9 | Prelinguistics | Assessment | Checklist from master data (tick items the child shows) |
| 10 | Observations | Assessment | One text area |
| 11 | Skills and senses | Assessment | Five text areas: reading, writing, hearing, vision, voice |

Extensibility: adding a future section means adding one row to the hub list, one step screen and one migration. Build the hub and steps from a single typed list of sections so this stays cheap.

Leaving a step with unsaved changes must not lose data: either save on Next or Back, or confirm before discarding. Saving on Next is preferred (a patient record is created as soon as a name exists).

### 4.7 Daily assessment (session)
Opened from a patient. Date defaults to today and can be changed.
- **Not entered yet for that date**: show the activity checklist, grouped (for example Language techniques, Attention techniques, Articulation), from master data. The therapist ticks the activities done, may add a note per activity, may add a new activity on the spot (it is saved to master data under a chosen group), and a general note for the day. Save creates the session.
- **Already entered for that date**: show the summary (what was done, notes, "n of m done") with an **Edit** button that reopens the checklist with the saved values.
- One session per patient per date in v1 (to confirm).

### 4.8 Master data
Reached from Settings, and from "Edit list" links inside forms. Five tabs: Activities (grouped), Birth history, Milestones (with domain and typical age range), Tests, Prelinguistics. Each list supports add, rename and delete (soft delete). Deleting an item does not remove it from past patient or session records; past records keep the stored name.

### 4.9 Reports (tab)
v1 is minimal: sessions per patient over a chosen period, and number of patients added. Anything beyond this is deferred (to confirm with the therapist).

### 4.10 Settings (tab)
Profile (name, clinic), change PIN, Master data, Backup now, Restore from backup, About. Backup and restore are described in section 7.

## 5. Navigation

- Auth stack: Sign up (first launch) or Unlock, then the main app.
- Bottom tabs: Home, Patients, Reports, Settings.
- Patient details, Add and Edit patient, Session and Master data open above the tabs as stack screens with a back arrow.
- Re-lock the app after it has been in the background for more than 1 minute (to confirm).

## 6. Data model (SQLite)

Conventions for every table: `id TEXT PRIMARY KEY` (UUID v4), `created_at TEXT NOT NULL`, `updated_at TEXT NOT NULL`, `deleted_at TEXT NULL`. The names below omit these four columns. Foreign keys are enforced (`PRAGMA foreign_keys = ON`).

**profile** (one row): therapist_name, clinic_name

**patients**: code (human-friendly, for example PT-0001, unique), name, sex ('male' | 'female' | 'other' | null), birth_date, address, status ('active' | 'archived'), concerns, birth_notes, education_history, medical_history, observations, reading_skills, writing_skills, hearing, vision, voice

**patient_family**: patient_id, relation ('father' | 'mother' | 'sibling'), name, birth_date, sort_order

**patient_birth_options**: patient_id, birth_option_id, label_snapshot

**patient_milestones**: patient_id, milestone_id, label_snapshot, reached (0/1), age_months (nullable)

**patient_tests**: patient_id, test_id, label_snapshot, result, done_on

**patient_prelinguistic**: patient_id, item_id, label_snapshot, present (0/1)

**sessions**: patient_id, session_date, general_note. Unique on (patient_id, session_date) where deleted_at IS NULL.

**session_activities**: session_id, activity_id, label_snapshot, group_snapshot, done (0/1), note

Master tables, each with a `name` and `sort_order`:
- **activity_groups**: name
- **activities**: group_id, name
- **birth_options**: name
- **milestones**: name, domain ('Motor' | 'Speech' | 'Language' | other), typical_range (text, for example "6-9 months")
- **tests**: name
- **prelinguistic_items**: name

Why the `label_snapshot` columns: master data can be renamed or deleted later, but a past record must keep reading the way it was written.

Seed data (inserted once, on first run, editable afterwards). These are samples to be confirmed with the therapist:
- Activity groups and activities: Language techniques (Picture naming, Story retelling, Following two-step commands); Attention techniques (Joint attention play, Sustained attention task, Turn-taking game); Articulation (Sound isolation drills, Minimal pairs)
- Birth options: Full term, Preterm, Normal delivery, C-section, Cried immediately, Delayed cry, NICU stay, Jaundice, Low birth weight
- Milestones: Head control (Motor, 3-4 months), Sits without support (Motor, 6-8), Walks alone (Motor, 12-15), Cooing (Speech, 2-4), Babbling (Speech, 6-9), First words (Speech, 10-14), Two-word phrases (Language, 18-24), Short sentences (Language, 30-36)
- Tests: Linguistic Profile Test (LPT), Preschool Language Scale (PLS-5), REEL-3 (Receptive-Expressive Emergent Language), Reynell Developmental Language Scales, Peabody Picture Vocabulary Test (PPVT)
- Prelinguistic items: Eye contact, Responds to name, Social smile, Joint attention, Turn-taking, Imitates actions, Imitates sounds, Pointing, Gestures (waving, reaching), Object play, Cooing, Babbling

## 7. Security, backup and restore

- PIN: 4 to 6 digits, salted hash in expo-secure-store. The database itself is not encrypted in v1 (to confirm; SQLCipher is an option later).
- Backup: export all tables to one JSON file `speechdata-backup-YYYY-MM-DD.json` with a `schema_version` field, then open the system share sheet so the therapist can save it to Drive, WhatsApp, email, and so on. Show a warning that the file contains patient data.
- Restore: pick a backup file, validate `schema_version` and structure, show how many patients it contains, require confirmation, then replace all local data in a single transaction. If anything fails, roll back and keep the existing data.
- Remind the therapist to back up if the last backup is more than 7 days old (a banner on Home).

## 8. Non-functional requirements

- Works fully offline; no network permission needed.
- Android first (the friend's phone). Keep the code iOS-compatible.
- Text inputs must not be covered by the keyboard; forms scroll.
- Large touch targets (44px minimum) and readable text (16px for inputs).
- App start to Unlock screen in under 2 seconds on a mid-range phone.

## 9. Build order (milestones)

Each milestone must run on a real phone before starting the next.

1. **Project setup**: Expo project, folder layout, theme, fonts, shared components (Button, Card, Chip, Checkbox, Tabs, StepHeader).
2. **Database**: client, migrations, all tables, seed data, repository functions with a few unit tests.
3. **Auth**: Sign up, Unlock, PIN storage, re-lock on background.
4. **Shell**: bottom tabs and empty Home, Patients, Reports, Settings screens.
5. **Master data**: the five lists with add, rename and delete. (Built early so the forms can use them.)
6. **Patients**: list with search, Add and Edit through the hub and 11 steps, patient details with tabs.
7. **Daily assessment**: checklist, add-on-the-go activity, save, summary, edit.
8. **Home and Reports**: the summaries described above.
9. **Backup and restore**: export, import, backup reminder banner.
10. **Polish and release**: empty states, error states, app icon and name, Android APK build for the therapist to try.

## 10. Open questions for the therapist

- Which linguistic tests and prelinguistic items does she actually use? Which activity groups?
- Is one session per patient per day enough, or can there be several?
- What does she want to see in Reports?
- Are there other fields she needs at the first visit (languages spoken at home, family history of speech or hearing problems, whether the parents are related)?
- Does she want biometric unlock?

## 11. Later (not v1)

API and server sync, accounts for organisations with several therapists, roles, cloud backup, PDF reports and shareable summaries for parents, reminders, attachments (audio recordings of speech samples), web app.
