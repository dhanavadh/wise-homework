# Mentee Account Registration

## Resource
- [Mentee Account Registration #2](https://github.com/SKT-ChAMP/wise-monorepo/issues/2#issue-3890995008)
- [Figma Design File](https://www.figma.com/design/AOlg1BT2bc1Zt4TBtxWLOD/%F0%9F%8E%A8--WISE--Design-File?node-id=1-347)

## Background
To participate in the WISE Mentoring Program, mentees must first create an account securely.
This ensures each applicant is uniquely registered and can proceed through verification and onboarding before entering the mentoring approval process.

## Acceptance Criteria
**Mentee Account Registration**
- Users must be able to register as a mentee using email and password.
- The system must require the user to input:
  - Email address
  - Password
  - Confirm password

**Validation Rules**
- Registration must fail if:
  - Email format is invalid
  - Email is already registered
  - Password and confirm password do not match

**Password Rules**
- Password requirements must be clearly displayed on the signup form.
- Users must not be able to continue unless password rules are satisfied.

**Successful Registration Outcome**
- When registration succeeds:
  - A mentee account must be created in an unverified state
  - The user must be directed to the Email Verification step

---

## Limitations / Out of Scope
- This scope does not include email OTP verification itself (handled in the next story).
- LINE binding, identity verification, profile setup, and document upload are not included in this story.
- Invitation codes are not part of the registration process, nor student id
