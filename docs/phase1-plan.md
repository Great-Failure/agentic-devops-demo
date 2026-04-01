# GiftEx 2.0 — Phase 1 Implementation Plan

## Problem Statement

The current codebase has a strong foundation (CRUD, matching, email, deployment) but lacks the **deadline system**, **game lifecycle states**, **RSVP flow**, and **automated reminders** required by the Phase 1 PRD. Templates are deferred to Phase 2.

## Approach

Deliver Phase 1 in **5 phases**, with API and frontend work running in parallel after the shared data model is established. The plan assumes two engineers (Brooke & Camille) working concurrently.

## Scope Decisions

- ✅ All 3 deadline types (RSVP, wishlist, reveal)
- ✅ Minimal draft→publish (status field + publish action, no draft-editing UI)
- ✅ Automated timer triggers for all deadline reminders
- ❌ Templates deferred to Phase 2
- ❌ Full draft management UI deferred to Phase 2

---

## Phase 1: Data Model & Types (Foundation — must complete first)

**Goal**: Extend the Game and Participant types to support deadlines, lifecycle states, and RSVP.

**Owner**: Paired (both engineers align on the contract)

### Tasks

- **1a. Extend `Game` type** (`api/src/shared/types.ts` + `src/lib/types.ts`)
  - Add `status: 'draft' | 'published' | 'revealed' | 'completed'`
  - Add `rsvpDeadline: string` (ISO date)
  - Add `wishlistDeadline: string` (ISO date)
  - Add `revealDate: string` (ISO date)
  - Add `publishedAt?: number`
  - Add `revealedAt?: number`
  - Default `status` to `'draft'` on creation

- **1b. Extend `Participant` type** (both `types.ts` files)
  - Add `rsvpStatus: 'pending' | 'yes' | 'no'`
  - Add `rsvpRespondedAt?: number`

- **1c. Add new action payload types** (`api/src/shared/types.ts`)
  - `PublishGamePayload` — transitions draft → published
  - `ConfirmRsvpPayload` — participant RSVP yes/no
  - `RevealAssignmentsPayload` — manual reveal trigger

- **1d. Add deadline validation utility** (`api/src/shared/game-utils.ts`)
  - `validateDeadlineSequence(rsvpDeadline, wishlistDeadline, revealDate, eventDate)` — ensures dates are in valid order
  - `isBeforeDeadline(deadline: string): boolean` — checks if current time is before deadline
  - `getGamePhase(game: Game): 'draft' | 'rsvp_open' | 'wishlists_open' | 'ready_to_reveal' | 'revealed' | 'completed'` — derives current phase from dates + status

- **1e. Update API unit tests for new types**
  - Update `types.test.ts` with new fields
  - Add `game-utils.test.ts` tests for deadline validation

---

## Phase 2: API — Lifecycle & Deadline Enforcement (Backend workstream)

**Goal**: Add state transitions, deadline enforcement, and RSVP endpoints.

**Owner**: Engineer A (e.g., Brooke)

### Tasks

- **2a. Update `createGame.ts`**
  - Accept new deadline fields in request body
  - Validate deadline sequence (RSVP < wishlist < reveal ≤ event date)
  - Set `status: 'draft'` by default
  - Do NOT generate assignments at creation (defer to publish)

- **2b. Add publish action to `updateGame.ts`**
  - New action: `action: 'publish'`
  - Validate: game is in `draft` status
  - Validate: at least 3 participants
  - Generate assignments on publish
  - Set `status: 'published'`, `publishedAt: Date.now()`
  - Send invitation emails to all participants

- **2c. Add RSVP action to `updateGame.ts`**
  - New action: `action: 'confirmRsvp'`
  - Accept `participantId` and `rsvpStatus: 'yes' | 'no'`
  - Validate: game is published and before RSVP deadline
  - Update participant's `rsvpStatus` and `rsvpRespondedAt`
  - Send RSVP confirmation email to participant
  - Notify organizer of RSVP

- **2d. Add reveal action to `updateGame.ts`**
  - New action: `action: 'revealAssignments'`
  - Validate: game is published, past reveal date or organizer manual trigger
  - Set `status: 'revealed'`, `revealedAt: Date.now()`
  - Send assignment emails to all participants who RSVP'd yes

- **2e. Add deadline enforcement to existing actions**
  - `updateWish`: reject if past `wishlistDeadline`
  - `joinInvitation`: reject if past `rsvpDeadline`
  - `addParticipant`: reject if game is not in draft/published status
  - Return clear error messages with deadline info

- **2f. Update `getGame.ts`**
  - Include `status` and deadline fields in response
  - Hide assignments from participants until `status === 'revealed'`
  - Include RSVP stats for organizer view (total, yes, no, pending)

- **2g. Update existing API tests + add new tests**
  - Test lifecycle transitions (draft → published → revealed)
  - Test deadline enforcement (reject actions past deadline)
  - Test RSVP flow
  - Test assignment hiding before reveal

---

## Phase 3: API — Automated Reminder Triggers (Backend workstream, cont.)

**Goal**: Add timer-triggered reminder emails for all deadlines.

**Owner**: Engineer A (e.g., Brooke)

### Tasks

- **3a. Create `rsvpDeadlineReminder.ts`** (`api/src/functions/`)
  - Timer: daily at 9:00 AM UTC (`0 0 9 * * *`)
  - Query: published games where RSVP deadline is tomorrow
  - Send reminder to participants with `rsvpStatus: 'pending'`
  - Track metrics via Application Insights

- **3b. Create `wishlistDeadlineReminder.ts`** (`api/src/functions/`)
  - Timer: daily at 9:00 AM UTC
  - Query: published games where wishlist deadline is tomorrow
  - Send reminder to participants who haven't added a wish
  - Track metrics

- **3c. Create `revealDateTrigger.ts`** (`api/src/functions/`)
  - Timer: daily at 9:00 AM UTC
  - Query: published games where reveal date is today
  - Auto-transition status to `'revealed'`
  - Send assignment reveal emails to all RSVP'd participants
  - Track metrics

- **3d. Create `eventReminder.ts`** (`api/src/functions/`)
  - Timer: daily at 9:00 AM UTC
  - Query: revealed games where event date is tomorrow
  - Send "event is tomorrow" reminder to all participants
  - Track metrics

- **3e. Add email templates for reminders** (`api/src/shared/email-service.ts`)
  - RSVP deadline reminder template (9 languages)
  - Wishlist deadline reminder template (9 languages)
  - Event upcoming reminder (leverage existing `sendEventUpcomingEmails`)

- **3f. Add unit tests for all timer triggers**
  - Test query logic (correct games selected)
  - Test email sending (correct recipients)
  - Test idempotency (don't double-send)

---

## Phase 4: Frontend — Deadline UI & RSVP Flow (Frontend workstream)

**Goal**: Update creation flow, add RSVP screen, enforce deadlines in UI.

**Owner**: Engineer B (e.g., Camille)

### Tasks

- **4a. Update `CreateGameView.tsx` — add deadline fields**
  - Add date pickers for RSVP deadline, wishlist deadline, reveal date
  - Add validation: dates must be in sequence (RSVP < wishlist < reveal ≤ event)
  - Show helpful labels ("Participants must RSVP by this date")
  - Wire to updated `createGameAPI()` call

- **4b. Update `CreateGameView.tsx` — draft behavior**
  - "Create" button now creates as draft (no immediate assignment)
  - Add "Publish" action on `GameCreatedView` or `OrganizerPanelView`
  - Show draft status badge
  - Disable share links until published

- **4c. Create RSVP screen** (new component or extend `JoinInvitationView`)
  - After joining via invite link, show RSVP prompt (Yes / No)
  - Gate wishlist submission behind RSVP = yes
  - Show RSVP deadline countdown
  - Disable RSVP after deadline with "RSVP closed" message

- **4d. Update `OrganizerPanelView.tsx` — RSVP tracking**
  - Add RSVP stats card (X confirmed, Y declined, Z pending)
  - Show per-participant RSVP status badge
  - Add "Publish" button (draft → published transition)
  - Add "Reveal Assignments" button (manual reveal trigger)
  - Show deadline dates in game details section

- **4e. Update `AssignmentView.tsx` — deadline enforcement**
  - Disable wish editing after wishlist deadline
  - Show "Wishlist closed" message after deadline
  - Hide assignment details until game status is `'revealed'`
  - Show "Assignments will be revealed on [date]" placeholder

- **4f. Update `ParticipantSelectionView.tsx`**
  - Show RSVP status per participant
  - Gate participant selection on RSVP completion

- **4g. Add translations** (`src/lib/translations.ts` + language files)
  - All new UI strings for deadlines, RSVP, draft/publish states
  - 9 languages

- **4h. Update frontend types** (`src/lib/types.ts`)
  - Mirror all type changes from Phase 1 (1a, 1b)
  - Keep in sync with API types

---

## Phase 5: Integration Testing & Polish

**Goal**: End-to-end validation, E2E tests, and deployment verification.

**Owner**: Both engineers

### Tasks

- **5a. Add E2E tests for new flows** (`e2e/`)
  - Test: Create draft game → publish → RSVP → add wish → reveal → view assignment
  - Test: Deadline enforcement (can't RSVP after deadline)
  - Test: Organizer publish and reveal actions
  - Test: Automated reveal (mock timer trigger)

- **5b. Update existing E2E tests**
  - Existing tests may break due to draft status default — update to include publish step
  - Verify all existing flows still pass

- **5c. QA environment validation**
  - Deploy to QA (`zavaexchangegift-qa`)
  - Test full flow with email delivery
  - Verify timer triggers fire correctly
  - Test deadline enforcement with real dates

- **5d. Update documentation**
  - Update `docs/api-reference.md` with new endpoints/actions
  - Update `docs/getting-started.md` if local dev flow changes
  - Update README if needed

---

## Parallel Workstream Summary

```
Phase 1 (Data Model)    ──── Both engineers pair ────────────────┐
                                                                  │
Phase 2 (API Lifecycle)  ──── Engineer A (Brooke) ───────┐       │
Phase 3 (API Triggers)   ──── Engineer A (Brooke) ──┐    │       │
                                                     │    │       │
Phase 4 (Frontend)       ──── Engineer B (Camille) ──┼────┘       │
                                                     │            │
Phase 5 (Integration)    ──── Both engineers ────────┘────────────┘
```

- **Phase 1** must complete before Phase 2/3/4 can start
- **Phase 2 + Phase 4** run in parallel (API and frontend)
- **Phase 3** can start once Phase 2a-2b are done (needs lifecycle in place)
- **Phase 5** starts when Phase 2-4 are feature-complete

---

## Risk Mitigations

| Risk | Mitigation |
|---|---|
| Type sync between frontend & API | Phase 1 is paired; one PR touches both files |
| Breaking existing E2E tests | Phase 5b explicitly addresses this; run tests early |
| Timer trigger reliability | Reuse proven `cleanupExpiredGames` pattern; add idempotency |
| Scope creep (templates, draft UI) | Explicitly deferred — hold the line |
| Deadline edge cases (timezones) | Use UTC consistently; document timezone handling |

## Out of Scope (Phase 2+)

- Template-based game creation
- Full draft management UI (edit draft, preview, etc.)
- Exclusion rules / no-pair rules
- Anonymous messaging
- Shipping & address handling
- Advanced analytics / dashboards
- Integrations (calendar, Slack)
