# Issue #14 — Sub-Issue Breakdown Analysis

## Overview

This document analyzes the current sub-issue breakdown for
[Issue #14: Mentee LINE Account Binding](https://github.com/SKT-ChAMP/wise-monorepo/issues/14)
and proposes an alternative breakdown that maps directly to the acceptance criteria.

---

## 1. Main Issue Acceptance Criteria (Reference)

### LINE Account Binding

| ID | Acceptance Criteria | Category |
|----|---------------------|----------|
| AC-1 | System must require mentees to bind LINE before final submission | Mandatory Binding |
| AC-2 | Users must not be able to skip this step | Mandatory Binding |
| AC-3 | System must redirect user to LINE authorization/consent screen | Binding Flow |
| AC-4 | User must explicitly approve required permissions | Binding Flow |
| AC-5 | Mentee account must be linked to a valid LINE user identifier | Successful Outcome |
| AC-6 | System must mark user state as **LINE Bound** | Successful Outcome |
| AC-7 | User must proceed automatically to the final confirmation step | Successful Outcome |
| AC-8 | If failed, users must be allowed to retry until successful | Retry Rules |
| AC-9 | Duplicate LINE binding across accounts must be prevented | Retry Rules |

### Terms Acceptance & Submission

| ID | Acceptance Criteria | Category |
|----|---------------------|----------|
| AC-10 | Users must accept all required program terms before submission | Mandatory Consent |
| AC-11 | Next button must be blocked until all required consent checkboxes are confirmed | Mandatory Consent |

---

## 2. Current Sub-Issue Breakdown (GitHub)

| # | Title | State | Layer |
|---|-------|-------|-------|
| #267 | [Domain] Mentee Entity | OPEN | Domain |
| #268 | [Infrastructure] LINE Integration Provider | OPEN | Infrastructure |
| #269 | [Repository] FindByLineUserId | OPEN | Repository |
| #270 | [Application] BindMenteeLineUseCase | OPEN | Application |
| #271 | [Application] AcceptTermsUseCase | OPEN | Application |
| #272 | [Controllers] Mentee Endpoints | OPEN | Interface |
| #273 | [UI] LINE Redirect & Callback Component | OPEN | Frontend |
| #274 | [UI] Terms & Conditions Consent Form | CLOSED | Frontend |
| #275 | [UI] Final Success Popup | CLOSED | Frontend |
| #278 | [Frontend] Refactor Line Binding Page Layout | CLOSED | Frontend |
| #422 | [Bug] Change font weight in line verification page | OPEN | Frontend |

---

## 3. Analysis of Current Breakdown

### 3.1 AC Coverage Gaps

The following acceptance criteria have **no explicit sub-issue** owning them:

| Acceptance Criteria | Gap |
|---------------------|-----|
| AC-2: Users must not be able to skip | No sub-issue covers a registration guard (backend status check or frontend route guard) that enforces LINE binding is mandatory before proceeding |
| AC-7: Auto-proceed to final confirmation | No sub-issue owns the navigation flow after successful binding |
| AC-8: Retry on failure | No sub-issue explicitly addresses error handling and retry UX |

### 3.2 Redundant Sub-Issue

**#268 [Infrastructure] LINE Integration Provider** is redundant. The LINE OAuth
infrastructure already exists in the codebase:

- Interface: `apps/wise-api/src/modules/line/domain/services/line-login.service.ts`
  (`ILineLoginService` with `getTokens(callbackUrl, code)`)
- Implementation: `apps/wise-api/src/modules/line/infrastructure/services/http/http-line-login.service.ts`
  (`HttpLineLoginService` — exchanges authorization code for tokens via LINE Login API)
- Config: `apps/wise-api/src/config/line.config.ts`
  (includes `menteeCallbackUrl` from `LINE_LOGIN_MENTEE_CALLBACK_URL` env var)

The naming differs from the sub-issue (service vs provider), but the functionality
is complete.

### 3.3 Terms Acceptance API — Not Required by AC

Sub-issues **#271** (`AcceptTermsUseCase`) and the `PATCH /mentees/me/terms`
endpoint in **#272** introduce a dedicated backend API for terms acceptance.

However, the main issue AC only states:

> - Next button must be blocked until all required consent checkboxes are confirmed.

This is a **frontend-only UI gate**. No backend endpoint is required by the
acceptance criteria. If the team wants to persist consent timestamps for
audit/compliance, that requirement should be added to issue #14's AC first.

### 3.4 Structural Problem: Horizontal Slicing

The breakdown is organized by **architecture layer**:

```
Domain (#267) → Infrastructure (#268) → Repository (#269) → UseCase (#270, #271) → Controller (#272) → UI (#273, #274)
```

This creates issues that are **not independently deliverable or testable**:

- **#267** (domain `bindLine` method) is useless without #270 (use case that calls it)
- **#269** (`findByLineUserId` repo method) is useless without #270 (use case that calls it)
- **#270** (use case) is useless without #272 (controller that exposes it)
- **#272** (controller) is useless without #273 (UI that calls it)

Each sub-issue only produces value when combined with others. This means:
- No sub-issue can be independently tested end-to-end
- PRs for individual sub-issues can't be verified against the AC
- The team can't demo progress until all layers are wired together

### 3.5 LINE OAuth Callback Ownership Gap

The LINE OAuth callback flow is split across sub-issues without clear ownership:

1. LINE redirects back to frontend with `?code=xxx&state=yyy` → **#273** (empty body)
2. Frontend extracts `code`, calls backend → gap between **#273** and **#272**
3. Backend receives code → **#272**
4. Backend exchanges code for token → **#270** → **#268** (already exists)
5. Backend stores LINE user ID → **#267**

No single sub-issue owns this flow. **#273** has an empty body and doesn't specify
that it must handle `code` extraction, `state` CSRF validation, or the API call
to the backend.

---

## 4. Proposed Breakdown (4 Sub-Issues)

The alternative is to slice **vertically by behavior** — each sub-issue delivers
a complete, testable piece of the acceptance criteria.

### Sub-Issue 1: [BE] Add `lineUserId` to User Entity + Repository Query

**Covers:** Prerequisite for AC-5, AC-9

**Scope:**

- Add `lineUserId: string | null` field to `User` entity
  (`apps/wise-api/src/modules/user/domain/entities/user.entity.ts` — currently commented out as `line?: UserLine`)
- Add `findByLineUserId(lineUserId: string): Promise<User | null>` to `IUserRepository`
  (`apps/wise-api/src/modules/user/domain/repositories/user.repository.ts`)
- Implement in Firestore repository
  (`apps/wise-api/src/modules/user/infrastructure/persistence/firestore/repositories/user.repository.ts`)
- Include `lineUserId` in `GetMyMenteeProfileOutputDTO` so frontend can check binding state

**Why one issue:** These are small, tightly coupled data-layer changes. The field
and the query method are both prerequisites for the callback endpoint. Splitting
them (like current #267 + #269) creates two issues that can't be independently
delivered or tested.

**Estimated size:** Small

---

### Sub-Issue 2: [BE] LINE OAuth Callback Endpoint

**Covers:** AC-3, AC-5, AC-6, AC-8, AC-9

**Scope:**

- Create `POST /mentees/me/line-binding` endpoint accepting `{ code: string }`
- Protected by `JwtAuthGuard` + `MenteeGuard`
  (existing: `apps/wise-api/src/modules/mentee/interface/rest/guards/mentee.guard.ts`)
- Validate mentee is in `PENDING_LINE_VERIFICATION` status (enforces AC-1/AC-2 at backend level)
- Use existing `HttpLineLoginService.getTokens(menteeCallbackUrl, code)` to exchange code for tokens
- Decode `id_token` JWT → extract `sub` claim as LINE user ID
- Check duplicate binding via `findByLineUserId()` → return 409 Conflict if exists (AC-9)
- Store `lineUserId` on User entity (AC-5)
- Advance mentee `registrationStatus` to `PENDING_PROFILE_COMPLETION` (AC-6)
- Return `{ success: true, registrationStatus }` (AC-8: endpoint is idempotent, frontend can retry)

**Key files to modify/create:**

| File | Action |
|------|--------|
| `modules/mentee/interface/rest/controllers/mentee.controller.ts` | Add POST endpoint |
| `modules/mentee/application/use-cases/bind-mentee-line/` | New use case (domain method + orchestration) |
| `modules/mentee/domain/entities/mentee.entity.ts` | Add `bindLine()` domain method with status guard |

**Why one issue:** The current breakdown splits this into 5 issues (#267 domain,
#268 provider, #269 repo, #270 use case, #272 controller). But this is a single
API call — the domain method, use case, and controller form one vertical slice
that should be delivered and tested together.

**Estimated size:** Medium

---

### Sub-Issue 3: [FE] LINE OAuth Redirect, Callback & Retry

**Covers:** AC-1, AC-2, AC-3, AC-4, AC-7, AC-8

**Scope:**

Replace mock logic in `MenteeLineVerificationPage.tsx`
(`apps/wise-mentorship-web/src/routes/register/mentee/pages/MenteeLineVerificationPage.tsx`)
with real LINE OAuth flow:

**Redirect (AC-3, AC-4):**
- Build LINE authorization URL with parameters:
  - `client_id` (from env/config)
  - `redirect_uri` (mentee callback URL)
  - `response_type=code`
  - `scope=profile openid`
  - `state` (random value stored in `sessionStorage` for CSRF protection)
- "ยืนยันตัวตนผ่าน LINE" button triggers redirect to LINE

**Callback handling:**
- On page load, detect `code` and `state` query params
- Validate `state` matches stored value (CSRF prevention)
- Call `POST /mentees/me/line-binding` with the `code`
- Handle states:
  - **Loading:** show spinner during code exchange
  - **Success:** update UI to show LINE bound state
  - **Error:** show error message with retry button (AC-8)

**Skip prevention (AC-1, AC-2):**
- On page load, check `lineUserId` from `GetMyMenteeProfile` response
- If already bound, show "already verified" state
- If not bound and no callback params, show the binding button

**Auto-proceed (AC-7):**
- After successful binding + terms acceptance, auto-navigate to confirmation page

**Reference:** Mentor implementation at
`apps/wise-mentorship-web/src/routes/register/mentor/pages/MentorLineVerificationPage.tsx`
follows a similar pattern.

**Why one issue:** Redirect and callback are two halves of one OAuth round-trip.
The current #273 has an empty body — this defines the complete frontend binding
flow including the gaps (retry, auto-proceed, skip prevention) that no current
sub-issue covers.

**Estimated size:** Medium

---

### Sub-Issue 4: [FE] Terms Consent Checkbox Gate

**Covers:** AC-10, AC-11

**Scope:**

Wire up the existing checkbox UI in `MenteeLineVerificationPage.tsx`:

- The agreement checkboxes already exist in the mock UI (terms, health assertion,
  truth assertion, consent radio)
- Show terms section only after LINE binding succeeds
- Disable Next/"เข้าร่วมโครงการ" button until all required checkboxes are checked (AC-11)
- On submit, navigate to confirmation page

**Why frontend-only:** The main issue AC says "Next button must be blocked until
all required consent checkboxes are confirmed." This is a UI gate — no backend
endpoint is specified. If the team decides consent must be persisted server-side
(for audit/compliance), that should be added to issue #14's AC first, then scoped
as an additional sub-issue.

**Estimated size:** Small

---

## 5. Dependency Graph

```
Sub-Issue 1: [BE] lineUserId + Repository
       │
       ▼
Sub-Issue 2: [BE] LINE Callback Endpoint
       │
       ▼
Sub-Issue 3: [FE] OAuth Redirect & Callback ──→ Sub-Issue 4: [FE] Terms Gate
```

- **Phase 1:** Sub-Issue 1 (data layer prerequisite)
- **Phase 2:** Sub-Issue 2 (backend endpoint) — can parallel with Sub-Issue 3 if frontend mocks the API
- **Phase 3:** Sub-Issue 3 + Sub-Issue 4 (frontend integration) — Sub-Issue 4 can be developed in parallel

---

## 6. Comparison

| Dimension | Current (8 open sub-issues) | Proposed (4 sub-issues) |
|-----------|-----------------------------|-------------------------|
| **Organized by** | Architecture layer | Acceptance criteria |
| **Each sub-issue independently testable?** | No | Yes |
| **Covers skip prevention (AC-1, AC-2)?** | No | Yes (Sub-Issue 2 backend + Sub-Issue 3 frontend) |
| **Covers retry (AC-8)?** | No | Yes (Sub-Issue 3) |
| **Covers auto-proceed (AC-7)?** | No | Yes (Sub-Issue 3) |
| **Redundant issues?** | Yes (#268 already implemented) | None |
| **Terms API required?** | Yes (#271, #272) | No (AC doesn't require it) |
| **AC traceability** | Hard — each AC spans 3-5 sub-issues | Clear — each sub-issue maps to specific ACs |
| **PR review complexity** | Simple per PR, but no PR is meaningful alone | Each PR delivers verifiable behavior |

---

## 7. Recommendation

1. **Close #268** — LINE infrastructure already exists.
2. **Consolidate #267 + #269** into one data-layer sub-issue (proposed Sub-Issue 1).
3. **Consolidate #270 + #272** (+ domain method from #267) into one endpoint sub-issue (proposed Sub-Issue 2).
4. **Rewrite #273** with a detailed body covering the full OAuth round-trip, retry, and auto-proceed (proposed Sub-Issue 3).
5. **Re-evaluate #271** — remove if the team agrees terms acceptance is frontend-only per the current AC. Add it back with updated AC if backend persistence is needed.
6. **Keep #422** (font weight bug) as-is — it's a standalone bug fix, not part of the feature breakdown.
