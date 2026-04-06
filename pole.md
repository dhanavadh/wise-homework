# Mentee Status Flow

## State Diagram

```mermaid
stateDiagram-v2
    [*] --> REGISTERED : register (email + password)
    REGISTERED --> EMAIL_VERIFIED : verify email
    EMAIL_VERIFIED --> LINE_VERIFIED : verify LINE
    LINE_VERIFIED --> PROFILE_COMPLETED : complete profile
    PROFILE_COMPLETED --> PENDING_APPROVAL : submit for approval
    PENDING_APPROVAL --> APPROVED : admin approves
    PENDING_APPROVAL --> REJECTED : admin rejects
    APPROVED --> [*]
    REJECTED --> [*]
```

## Status Descriptions

| Status | Description |
|---|---|
| `REGISTERED` | Mentee has registered with email and password |
| `EMAIL_VERIFIED` | Mentee has verified their email address |
| `LINE_VERIFIED` | Mentee has verified their LINE account |
| `PROFILE_COMPLETED` | Mentee has completed their profile information |
| `PENDING_APPROVAL` | Mentee has submitted their profile for admin review |
| `APPROVED` | Admin has approved the mentee |
| `REJECTED` | Admin has rejected the mentee |
