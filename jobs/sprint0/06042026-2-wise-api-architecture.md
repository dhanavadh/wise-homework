# Wise API - Architecture Guide

> A beginner-friendly guide to understanding the `apps/wise-api` codebase.
> No prior knowledge of DDD, Clean Architecture, or Firestore required.

---

## Table of Contents

1. [What is this app?](#1-what-is-this-app)
2. [Start Here - Reading Order](#2-start-here---reading-order)
3. [The Big Picture (in plain words)](#3-the-big-picture-in-plain-words)
4. [Concept: What is Firestore?](#4-concept-what-is-firestore)
5. [Concept: What is Clean Architecture?](#5-concept-what-is-clean-architecture)
6. [Concept: What is DDD?](#6-concept-what-is-ddd)
7. [Concept: What is Lazy Loading?](#7-concept-what-is-lazy-loading)
8. [Why is Mentee separate from User?](#8-why-is-mentee-separate-from-user)
9. [How a Request Flows (step by step)](#9-how-a-request-flows-step-by-step)
10. [Module-by-Module Guide](#10-module-by-module-guide)
11. [Folder Pattern Cheat Sheet](#11-folder-pattern-cheat-sheet)
12. [Full System Graph](#12-full-system-graph)

---

## 1. What is this app?

Wise API is the **backend server** for a mentorship platform. Think of it like a school alumni network where:

- **Mentors** (alumni/professionals) sign up to guide students
- **Mentees** (students) sign up to get guidance
- **Admins** manage the platform

The server handles: user registration, login/logout, profile management, file uploads (profile pictures, documents), email notifications, and LINE social login.

**Tech stack:**
- **NestJS** = the web framework (like Express, but with more structure)
- **TypeScript** = the language
- **Firestore** = the database (Google's cloud database)
- **Google Cloud Storage** = file storage (for profile pictures, documents)

---

## 2. Start Here - Reading Order

Don't try to read everything at once. Follow this path:

```mermaid
graph TD
    Step1["Step 1: src/main.ts<br/><i>Where the server starts</i>"]
    Step2["Step 2: src/app/app.module.ts<br/><i>What modules are loaded</i>"]
    Step3["Step 3: Pick ONE module to trace<br/><i>Recommended: auth module</i>"]
    Step4["Step 3a: auth/interface/rest/controllers/auth.controller.ts<br/><i>What URLs exist? What do they do?</i>"]
    Step5["Step 3b: auth/application/use-cases/login-user/login-user.use-case.ts<br/><i>What happens when someone logs in?</i>"]
    Step6["Step 3c: auth/domain/entities/login-session.entity.ts<br/><i>What data does a login session have?</i>"]
    Step7["Step 3d: auth/infrastructure/.../login-session.repository.ts<br/><i>How is it saved to the database?</i>"]
    Step8["Step 4: Read mentee or mentor module<br/><i>Same pattern, now it will feel familiar</i>"]

    Step1 --> Step2
    Step2 --> Step3
    Step3 --> Step4
    Step4 --> Step5
    Step5 --> Step6
    Step6 --> Step7
    Step7 --> Step8

    style Step1 fill:#e3f2fd
    style Step2 fill:#e3f2fd
    style Step3 fill:#e8f5e9
    style Step4 fill:#e8f5e9
    style Step5 fill:#e8f5e9
    style Step6 fill:#e8f5e9
    style Step7 fill:#e8f5e9
    style Step8 fill:#fff8e1
```

**Key insight:** Every module follows the SAME folder pattern. Once you understand one module (auth), you understand them all.

---

## 3. The Big Picture (in plain words)

Here's what this app looks like, explained simply:

```mermaid
graph LR
    Browser["User's Browser"] -->|"sends HTTP request<br/>(e.g. POST /api/auth/login)"| Server["Wise API Server<br/>(NestJS)"]
    Server -->|"reads/writes data"| DB["Firestore<br/>(Database)"]
    Server -->|"uploads/downloads files"| GCS["Google Cloud Storage<br/>(File Storage)"]
    Server -->|"sends emails"| SMTP["Email Server"]
    Server -->|"social login"| LINE["LINE API"]
```

That's it. The browser talks to our server. Our server talks to Google's services.

Inside the server, the code is organized into **modules** — think of them like departments in a company:

```mermaid
graph TB
    subgraph Server["Our Server"]
        Auth["Auth Department<br/><i>Handles login/logout</i>"]
        Mentee["Mentee Department<br/><i>Handles mentee profiles</i>"]
        Mentor["Mentor Department<br/><i>Handles mentor profiles</i>"]
        User["User Department<br/><i>Handles user accounts</i>"]
        Admin["Admin Department<br/><i>Handles admin stuff</i>"]
        Mail["Mail Department<br/><i>Sends emails</i>"]
        Storage["Storage Department<br/><i>Manages files</i>"]
        Line["LINE Department<br/><i>Social login via LINE</i>"]
        MasterData["Master Data Department<br/><i>Reference lists: faculties, topics, etc.</i>"]
    end
```

Each department is independent. They have their own folders, their own code, their own database tables. But they can talk to each other when needed (e.g., Auth needs User to verify passwords).

---

## 4. Concept: What is Firestore?

**Firestore** is Google's cloud database. It's a **NoSQL document database** — think of it like a giant JSON file in the cloud.

Instead of tables with rows (like MySQL), Firestore has **collections** with **documents**:

```
Firestore Database
├── users/                          <-- collection (like a folder)
│   ├── user-id-001                 <-- document (like a file)
│   │   ├── email: "alice@mail.com"
│   │   ├── password: "$argon2..."
│   │   └── type: "mentee"
│   └── user-id-002
│       ├── email: "bob@mail.com"
│       └── ...
│
├── login_sessions/
│   └── session-id-001
│       ├── userId: "user-id-001"
│       ├── expiresAt: 2026-08-01
│       └── revoked: false
│
├── mentees/
│   └── mentee-id-001
│       ├── userId: "user-id-001"
│       ├── status: "ACTIVE"
│       └── educationProfile: { ... }
│
├── mentors/
│   └── mentor-id-001
│       ├── userId: "user-id-002"
│       ├── status: "ACTIVE"
│       └── educations: [ ... ]
│
└── faculties/
    └── faculty-id-001
        ├── name: { en: "Engineering", th: "วิศวกรรมศาสตร์" }
        └── icon: "..."
```

In the code, you'll see **Repository** classes that read/write to these collections. That's all a repository does — it's the bridge between your code and the database.

---

## 5. Concept: What is Clean Architecture?

Clean Architecture is just a rule about **who can talk to whom**.

Imagine 4 boxes stacked like a cake:

```mermaid
graph TB
    subgraph Layer4["Layer 4: INTERFACE (top — talks to the outside world)"]
        L4["Controllers, Guards, Interceptors<br/><i>Receives HTTP requests, sends HTTP responses</i>"]
    end

    subgraph Layer3["Layer 3: APPLICATION (business workflows)"]
        L3["Use Cases, DTOs<br/><i>'When a user logs in, do X then Y then Z'</i>"]
    end

    subgraph Layer2["Layer 2: DOMAIN (core business rules)"]
        L2["Entities, Enums, Repository Interfaces<br/><i>'A LoginSession has a userId and can be revoked'</i>"]
    end

    subgraph Layer1["Layer 1: INFRASTRUCTURE (bottom — talks to databases/services)"]
        L1["Firestore Repositories, SMTP Mail, GCS Storage<br/><i>The actual code that reads/writes to Firestore, sends emails, etc.</i>"]
    end

    Layer4 -->|"can use"| Layer3
    Layer3 -->|"can use"| Layer2
    Layer1 -->|"implements"| Layer2

    style Layer4 fill:#e3f2fd
    style Layer3 fill:#e8f5e9
    style Layer2 fill:#fff8e1
    style Layer1 fill:#fce4ec
```

**The ONE rule:** Each layer can ONLY depend on the layer below it. Never upward.

- Controllers (Layer 4) call Use Cases (Layer 3) -- OK
- Use Cases (Layer 3) use Entities (Layer 2) -- OK
- Use Cases (Layer 3) call Controllers (Layer 4) -- FORBIDDEN
- Domain (Layer 2) imports Firestore code (Layer 1) -- FORBIDDEN

**Why?** So you can change the database (e.g., switch from Firestore to PostgreSQL) without touching any business logic. The Domain layer doesn't know or care about Firestore.

**How?** The Domain layer defines **interfaces** (contracts):

```typescript
// Domain says: "I need someone who can do these things"
interface IUserRepository {
  findById(id: string): Promise<User | null>
  findByEmail(email: string): Promise<User | null>
}

// Infrastructure says: "I can do those things using Firestore!"
class FirestoreUserRepository implements IUserRepository {
  async findById(id: string) { /* ... Firestore code ... */ }
  async findByEmail(email: string) { /* ... Firestore code ... */ }
}
```

---

## 6. Concept: What is DDD?

**DDD (Domain-Driven Design)** is a way to organize code around **business concepts** instead of technical concerns.

Instead of organizing by file type:

```
BAD (organized by tech):
├── controllers/
│   ├── auth.controller.ts
│   ├── mentee.controller.ts
│   └── mentor.controller.ts
├── services/
│   ├── auth.service.ts
│   └── mentee.service.ts
└── models/
    ├── user.model.ts
    └── mentee.model.ts
```

DDD organizes by **business domain** (what the business cares about):

```
GOOD (organized by domain):
├── modules/
│   ├── auth/           <-- everything about authentication
│   │   ├── domain/
│   │   ├── application/
│   │   ├── interface/
│   │   └── infrastructure/
│   ├── mentee/         <-- everything about mentees
│   │   ├── domain/
│   │   ├── application/
│   │   ├── interface/
│   │   └── infrastructure/
│   └── mentor/         <-- everything about mentors
```

**Why?** When someone says "change how mentee registration works," you go to `modules/mentee/` and everything you need is there. You don't have to hunt across 10 different folders.

Key DDD terms used in this project:

| Term | What it means | Example in this project |
|------|--------------|------------------------|
| **Entity** | A thing with an identity | `User`, `Mentee`, `Mentor`, `LoginSession` |
| **Repository** | Where entities are stored/retrieved | `IUserRepository`, `IMenteeRepository` |
| **Use Case** | A business action | `LoginUserUseCase`, `GetMenteeByIdUseCase` |
| **DTO** | Data shape for input/output | `LoginUserInputDTO { email, password }` |
| **Bounded Context** | A boundary around related concepts | Each module is a bounded context |

---

## 7. Concept: What is Lazy Loading?

Normally in NestJS, when the server starts, it loads ALL code into memory — even code that nobody has requested yet.

**Lazy loading** means: "Don't load this code until someone actually needs it."

```mermaid
graph LR
    subgraph Normal["Normal Loading (eager)"]
        direction TB
        Start1["Server starts"] --> Load1["Load ALL use cases<br/>into memory"]
        Load1 --> Ready1["Server ready<br/><i>(slow start, uses more memory)</i>"]
    end

    subgraph Lazy["Lazy Loading"]
        direction TB
        Start2["Server starts"] --> Ready2["Server ready<br/><i>(fast start, low memory)</i>"]
        Ready2 --> Request["First request to /login"]
        Request --> Load2["NOW load LoginUserUseCase"]
        Load2 --> Cached["Cache it for next time"]
    end

    style Normal fill:#fce4ec
    style Lazy fill:#e8f5e9
```

In the code, it looks like this:

```typescript
// In the controller:
async login(@Body() body: LoginUserInputDTO) {
  // This LAZY LOADS the use case the first time it's called
  const loginUserUseCase = await this.load<LoginUserUseCase>(
    () => import('.../login-user.use-case.module'),  // load the module
    () => import('.../login-user.use-case')           // get the use case from it
  )

  return loginUserUseCase.execute(body)
}
```

**Why?** Faster server startup. If nobody ever calls the mentee endpoints, that code is never loaded. Especially useful when the app has many modules.

---

## 8. Why is Mentee separate from User?

This is the most important design question. Here's why:

### The design: User + Mentee/Mentor are SEPARATE entities

```mermaid
graph TB
    subgraph CurrentDesign["Current Design: Separate Entities"]
        User["User<br/><i>id, email, password</i><br/><small>Collection: users</small>"]
        Mentee["Mentee<br/><i>userId, status, educationProfile,<br/>documents, registrationStatus</i><br/><small>Collection: mentees</small>"]
        Mentor["Mentor<br/><i>userId, status, educations,<br/>certificates, workExperiences</i><br/><small>Collection: mentors</small>"]

        User --- |"1 User can be"| Mentee
        User --- |"1 User can be"| Mentor
    end

    style User fill:#e3f2fd
    style Mentee fill:#e8f5e9
    style Mentor fill:#fff8e1
```

### Why not just extend User? (class Mentee extends User)

You might think:

```typescript
// "Why not just do this?"
class Mentee extends User {
  educationProfile: EducationProfile
  documents: MenteeDocuments
  // ...
}
```

Here are the reasons this project chose NOT to do that:

#### Reason 1: A User can be BOTH a Mentor AND a Mentee

```mermaid
graph LR
    Alice["User: Alice<br/>email: alice@mail.com"] --> MenteeAlice["Mentee: Alice<br/>status: ACTIVE<br/>studentId: 6730000021"]
    Alice --> MentorAlice["Mentor: Alice<br/>status: ACTIVE<br/>certificates: [...]"]

    style Alice fill:#e3f2fd
    style MenteeAlice fill:#e8f5e9
    style MentorAlice fill:#fff8e1
```

If Mentee extended User, Alice would need TWO User objects with the same email — that's messy. With separate entities, Alice has ONE User record and both a Mentee AND Mentor record pointing to it via `userId`.

#### Reason 2: Different lifecycles

```mermaid
graph TD
    subgraph UserLifecycle["User Lifecycle"]
        U1["Created<br/>(email + password)"] --> U2["Active<br/>(can login)"]
    end

    subgraph MenteeLifecycle["Mentee Registration Lifecycle"]
        M1["PENDING_EMAIL_VERIFICATION"] --> M2["PENDING_LINE_VERIFICATION"]
        M2 --> M3["PENDING_PROFILE_COMPLETION"]
        M3 --> M4["PENDING_APPROVAL"]
        M4 --> M5["APPROVED"]
        M4 --> M6["REJECTED"]
    end

    subgraph MentorLifecycle["Mentor Registration Lifecycle"]
        MT1["PENDING_LINE_VERIFICATION"] --> MT2["PENDING_PROFILE_COMPLETION"]
        MT2 --> MT3["COMPLETED"]
    end

    style UserLifecycle fill:#e3f2fd
    style MenteeLifecycle fill:#e8f5e9
    style MentorLifecycle fill:#fff8e1
```

User registration is simple (email + password). But Mentee registration has 6 steps! And Mentor registration has a different 3-step flow. These are completely different processes — mixing them into one class would make it very complex.

#### Reason 3: Different data shapes

| User | Mentee | Mentor |
|------|--------|--------|
| email | educationProfile | invitationCodeId |
| password | documents (citizenIdCard, resume, transcript) | educations[] |
| | referralSource | certificates[] |
| | facultyId, studentId | currentWorkExperiences[] |
| | | previousWorkExperiences[] |
| | | hobbies[], mentoringTopicsIds[] |

These are vastly different data models. Forcing them into one inheritance tree would create a "god object" with dozens of nullable fields.

#### Reason 4: Separate Firestore collections

In Firestore, each entity type has its own collection. This means:
- `users` collection = login credentials only (fast auth queries)
- `mentees` collection = mentee-specific data (no irrelevant fields)
- `mentors` collection = mentor-specific data

This is more efficient for queries and keeps each collection clean.

#### Summary

```mermaid
graph TB
    subgraph Wrong["If Mentee extended User"]
        BigUser["class Mentee extends User<br/><i>email, password,<br/>educationProfile, documents,<br/>registrationStatus, ...</i><br/><br/>Problems:<br/>- Can't be both Mentor AND Mentee<br/>- One giant class with 30+ fields<br/>- Mentee lifecycle mixed with User lifecycle<br/>- All in one DB collection"]
    end

    subgraph Right["Current Design: Composition"]
        SmallUser["User<br/><i>email, password</i>"]
        SmallMentee["Mentee<br/><i>userId, educationProfile,<br/>documents, ...</i>"]
        SmallMentor["Mentor<br/><i>userId, educations,<br/>certificates, ...</i>"]
        SmallUser ---|"linked by userId"| SmallMentee
        SmallUser ---|"linked by userId"| SmallMentor
    end

    style Wrong fill:#fce4ec
    style Right fill:#e8f5e9
```

---

## 9. How a Request Flows (step by step)

Let's trace what happens when a mentee opens their profile page (`GET /api/mentees/me`):

```mermaid
sequenceDiagram
    actor User as Mentee's Browser
    participant Guard1 as JwtAuthGuard<br/>(checks login)
    participant Guard2 as MenteeGuard<br/>(checks mentee role)
    participant Ctrl as MenteeController
    participant UC as GetMyMenteeProfile<br/>UseCase
    participant Repo as MenteeRepository
    participant DB as Firestore

    Note over User,DB: Step 1: Browser sends request with cookie

    User->>Guard1: GET /api/mentees/me<br/>Cookie: accessToken=eyJ...

    Note over Guard1: Step 2: Check if user is logged in

    Guard1->>Guard1: Extract JWT from cookie
    Guard1->>Guard1: Decode JWT -> { loginSessionId }
    Guard1->>DB: Find login session
    DB-->>Guard1: Session found, not expired
    Guard1->>Guard1: Attach userId to request

    Note over Guard2: Step 3: Check if user is a mentee

    Guard2->>DB: Find mentee by userId
    DB-->>Guard2: Mentee found!
    Guard2->>Guard2: Attach mentee to request

    Note over Ctrl: Step 4: Controller calls use case

    Ctrl->>Ctrl: @CurrentMentee() extracts<br/>mentee from request
    Ctrl->>UC: Lazy load & execute({ id: mentee.id })

    Note over UC: Step 5: Use case fetches data

    UC->>Repo: findById(menteeId)
    Repo->>DB: Get document from<br/>'mentees' collection
    DB-->>Repo: Document data
    Repo-->>UC: Mentee entity

    Note over UC: Step 6: Build response

    UC->>UC: Convert to OutputDTO
    UC-->>Ctrl: Result with profile data
    Ctrl-->>User: 200 OK { id, status, educationProfile, ... }
```

### The same flow, simplified

```
Browser
  |
  v
[JwtAuthGuard] -- "Are you logged in?" -- checks cookie + Firestore session
  |
  v
[MenteeGuard] -- "Are you a mentee?" -- checks Firestore mentees collection
  |
  v
[MenteeController] -- "What do you want?" -- routes to the right use case
  |
  v
[GetMyMenteeProfileUseCase] -- "Let me get your data" -- calls repository
  |
  v
[FirestoreMenteeRepository] -- "Here's the Firestore query" -- talks to DB
  |
  v
[Firestore] -- returns data -- back up the chain -- HTTP response to browser
```

---

## 10. Module-by-Module Guide

### Auth Module (start here!)

**What it does:** Handles login, logout, and checking if a user is logged in.

**Files to read (in order):**

| Order | File | What you'll learn |
|-------|------|-------------------|
| 1 | `modules/auth/interface/rest/controllers/auth.controller.ts` | What endpoints exist (`POST /login`, `POST /logout`) |
| 2 | `modules/auth/application/use-cases/login-user/login-user.use-case.ts` | The login logic: find user -> check password -> create session -> sign JWT |
| 3 | `modules/auth/domain/entities/login-session.entity.ts` | What a session looks like: userId, expiresAt, revoked |
| 4 | `modules/auth/interface/rest/guards/jwt-auth.guard.ts` | How EVERY request gets authenticated |
| 5 | `modules/auth/interface/rest/interceptors/set-access-token-cookie.interceptor.ts` | How the JWT gets stored in a browser cookie |

**How login works:**

```mermaid
graph TD
    A["Client sends<br/>{ email, password }"] --> B["Find user by email<br/>in Firestore"]
    B --> C{"User exists?"}
    C -->|No| D["401 Unauthorized"]
    C -->|Yes| E["Verify password<br/>with argon2 hash"]
    E --> F{"Password correct?"}
    F -->|No| D
    F -->|Yes| G["Create LoginSession entity<br/>(userId, expiresAt)"]
    G --> H["Save session to<br/>Firestore login_sessions"]
    H --> I["Sign JWT with<br/>{ loginSessionId }"]
    I --> J["Set JWT in<br/>HttpOnly cookie"]
    J --> K["Return 200 OK"]

    style D fill:#fce4ec
    style K fill:#e8f5e9
```

---

### Mentee Module

**What it does:** Manages mentee profiles and registration.

**Files to read:**

| Order | File | What you'll learn |
|-------|------|-------------------|
| 1 | `modules/mentee/interface/rest/controllers/mentee.controller.ts` | Endpoints: `GET /mentees/me`, `GET /mentees/:id` |
| 2 | `modules/mentee/interface/rest/guards/mentee.guard.ts` | How it checks "is this user a mentee?" |
| 3 | `modules/mentee/domain/entities/mentee.entity.ts` | What a Mentee looks like (status, documents, education) |
| 4 | `modules/mentee/domain/enums/mentee.enum.ts` | Registration steps (PENDING_EMAIL -> ... -> APPROVED) |

---

### Mentor Module

**What it does:** Same pattern as Mentee, but for mentors.

**Files to read:**

| Order | File | What you'll learn |
|-------|------|-------------------|
| 1 | `modules/mentor/interface/rest/controllers/mentor.controller.ts` | Endpoints: `GET /mentors/me`, `GET /mentors/:id` |
| 2 | `modules/mentor/domain/entities/mentor.entity.ts` | What a Mentor looks like (education, certificates, work experience) |
| 3 | `modules/mentor/domain/enums/mentor.enum.ts` | Registration steps, education degrees, certificate types |

---

### User Module

**What it does:** The base user account (email + password). Used by Auth for login.

**Key file:** `modules/user/domain/entities/user.entity.ts` — currently minimal (just email + password).

---

### Support Modules (Mail, Storage, Line)

These are "utility" modules. Other modules use them when needed.

```mermaid
graph TB
    subgraph Mail["Mail Module"]
        direction LR
        IMailService["IMailService<br/><i>(interface: 'I can send emails')</i>"]
        SmtpMailService["SmtpMailService<br/><i>(implementation: 'I send via SMTP')</i>"]
        Templates["Email Templates<br/><i>welcome, mentee-request,<br/>reset_password</i>"]
        IMailService -.->|"implemented by"| SmtpMailService
        SmtpMailService --> Templates
    end

    subgraph Storage["Storage Module"]
        direction LR
        IStorageService["IStorageService<br/><i>(interface: 'I can store files')</i>"]
        GcsService["GcsStorageService<br/><i>(implementation: 'I use Google Cloud')</i>"]
        IStorageService -.->|"implemented by"| GcsService
    end

    subgraph Line["Line Module"]
        direction LR
        ILineService["ILineLoginService<br/><i>(interface: 'I can do LINE OAuth')</i>"]
        HttpLineService["HttpLineLoginService<br/><i>(implementation: 'I call LINE API')</i>"]
        ILineService -.->|"implemented by"| HttpLineService
    end

    style Mail fill:#fff3e0
    style Storage fill:#e8f5e9
    style Line fill:#e3f2fd
```

---

## 11. Folder Pattern Cheat Sheet

Every module follows the SAME 4-folder structure. Here's a cheat sheet:

```mermaid
graph TB
    subgraph Module["Any Module (e.g. mentee/)"]
        subgraph Interface["interface/ = HTTP Layer"]
            direction TB
            Controllers["controllers/<br/><i>Receives requests, returns responses</i>"]
            Guards["guards/<br/><i>Blocks unauthorized access</i>"]
            Decorators["decorators/<br/><i>Extracts data from requests</i>"]
            Interceptors["interceptors/<br/><i>Modifies request/response</i>"]
        end

        subgraph Application["application/ = Business Workflows"]
            direction TB
            UseCases["use-cases/<br/><i>The actual logic<br/>(login, register, get profile)</i>"]
            DTOs["dtos/<br/><i>Input/output shapes<br/>with validation rules</i>"]
        end

        subgraph Domain["domain/ = Core Business Rules"]
            direction TB
            Entities["entities/<br/><i>The 'things' in your system<br/>(User, Mentee, LoginSession)</i>"]
            Repos["repositories/<br/><i>Interfaces for data access<br/>(NOT the actual DB code)</i>"]
            Enums_["enums/<br/><i>Status values, types</i>"]
        end

        subgraph Infra["infrastructure/ = External Connections"]
            direction TB
            FirestoreRepos["persistence/firestore/repositories/<br/><i>Actual Firestore read/write code</i>"]
            Services["services/<br/><i>Actual email/storage/API code</i>"]
        end
    end

    Interface -->|"calls"| Application
    Application -->|"uses"| Domain
    Infra -->|"implements"| Domain

    style Interface fill:#e3f2fd
    style Application fill:#e8f5e9
    style Domain fill:#fff8e1
    style Infra fill:#fce4ec
```

**Quick reference:**
- "Where are the API endpoints?" -> `interface/rest/controllers/`
- "Where is the business logic?" -> `application/use-cases/`
- "What does the data look like?" -> `domain/entities/`
- "Where is the database code?" -> `infrastructure/persistence/firestore/repositories/`

---

## 12. Full System Graph

Everything connected together:

```mermaid
graph TB
    subgraph Client["Browser"]
        Web["Web App"]
    end

    subgraph NestJS["Wise API Server"]
        Main["main.ts<br/><small>Starts server on port 3000<br/>Sets up CORS, Swagger</small>"]

        subgraph AppModule_["AppModule loads these modules:"]
            direction TB

            subgraph AuthMod["Auth Module"]
                AuthCtrl["POST /auth/login<br/>POST /auth/logout"]
                JwtGuard["JwtAuthGuard<br/><small>(runs on EVERY request)</small>"]
                LoginUC["LoginUserUseCase"]
                LogoutUC["LogoutUserUseCase"]
                ValidateUC["ValidateLoginSession"]
                SessionRepo["LoginSession Repository"]
            end

            subgraph MenteeMod["Mentee Module"]
                MenteeCtrl["GET /mentees/me<br/>GET /mentees/:id"]
                MenteeGuard["MenteeGuard"]
                GetMenteeUC["GetMenteeById<br/>GetMyMenteeProfile"]
                MenteeRepo["Mentee Repository"]
            end

            subgraph MentorMod["Mentor Module"]
                MentorCtrl["GET /mentors/me<br/>GET /mentors/:id"]
                MentorGuard["MentorGuard"]
                GetMentorUC["GetMentorById<br/>GetMyMentorProfile"]
                MentorRepo["Mentor Repository"]
            end

            subgraph UserMod["User Module"]
                UserRepo["User Repository"]
            end
        end
    end

    subgraph External["Google Cloud"]
        Firestore["Firestore DB"]
        GCS["Cloud Storage"]
    end

    Web -->|"HTTP"| Main
    Main --> AuthCtrl
    Main --> MenteeCtrl
    Main --> MentorCtrl

    AuthCtrl --> LoginUC
    AuthCtrl --> LogoutUC
    JwtGuard --> ValidateUC

    LoginUC --> UserRepo
    LoginUC --> SessionRepo
    ValidateUC --> SessionRepo
    LogoutUC --> SessionRepo

    MenteeCtrl --> GetMenteeUC
    MenteeGuard --> MenteeRepo
    GetMenteeUC --> MenteeRepo

    MentorCtrl --> GetMentorUC
    MentorGuard --> MentorRepo
    GetMentorUC --> MentorRepo

    SessionRepo --> Firestore
    UserRepo --> Firestore
    MenteeRepo --> Firestore
    MentorRepo --> Firestore

    style Client fill:#e1f5fe
    style NestJS fill:#f3e5f5
    style External fill:#fff3e0
    style AuthMod fill:#e3f2fd
    style MenteeMod fill:#e8f5e9
    style MentorMod fill:#fff8e1
    style UserMod fill:#fce4ec
```

---

## Quick Glossary

| Term | Simple Meaning |
|------|---------------|
| **NestJS** | A framework for building servers in TypeScript (like Express with structure) |
| **Module** | A group of related code (like a department in a company) |
| **Controller** | The code that receives HTTP requests and returns responses |
| **Guard** | A bouncer — blocks requests that don't have permission |
| **Interceptor** | Middleware that can modify the request or response |
| **Use Case** | A single business action (e.g., "log in", "get mentee profile") |
| **DTO** | Data Transfer Object — defines the shape of input/output data |
| **Entity** | A business object with an ID (e.g., User, Mentee) |
| **Repository** | The code that reads/writes entities to/from the database |
| **Firestore** | Google's cloud NoSQL database |
| **JWT** | JSON Web Token — a signed string that proves you're logged in |
| **Lazy Loading** | Loading code only when it's first needed, not at startup |
| **DDD** | Organizing code by business domain (auth, mentee, mentor) |
| **Clean Architecture** | A rule that inner layers can't depend on outer layers |
