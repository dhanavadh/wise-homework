# Wise API - Architecture Documentation

> **App:** `apps/wise-api`
> **Framework:** NestJS (v10) with TypeScript
> **Database:** Google Cloud Firestore
> **Build:** Nx monorepo + Webpack
> **Architecture:** Clean Architecture / Domain-Driven Design (DDD)

---

## Table of Contents

1. [Project Overview](#project-overview)
2. [Directory Structure](#directory-structure)
3. [Architecture Layers](#architecture-layers)
4. [Module Overview](#module-overview)
5. [High-Level Architecture Graph](#high-level-architecture-graph)
6. [Request Lifecycle](#request-lifecycle)
7. [Module Dependency Graph](#module-dependency-graph)
8. [Auth Module Flow](#auth-module-flow)
9. [Mentee Module Flow](#mentee-module-flow)
10. [Mentor Module Flow](#mentor-module-flow)
11. [Clean Architecture Layer Graph](#clean-architecture-layer-graph)
12. [Infrastructure Services](#infrastructure-services)
13. [Firestore Collections](#firestore-collections)
14. [Configuration System](#configuration-system)
15. [Lazy Loading Pattern](#lazy-loading-pattern)

---

## Project Overview

Wise API is a mentorship platform backend that connects **Mentors** and **Mentees**. It provides:

- **Authentication** via email/password with JWT sessions stored in Firestore
- **Mentor & Mentee registration** with multi-step registration flows
- **Profile management** including profile pictures via GCS signed URLs
- **LINE Login integration** for social authentication
- **Email service** via SMTP with Handlebars templates
- **Admin management** for platform administrators
- **Master data** management (Faculties, Topics, Industries, Hobbies, Invite Codes)

The API is served at `/api` with Swagger documentation at `/api/explorer` (dev/staging only).

---

## Directory Structure

```
apps/wise-api/src/
|-- main.ts                          # Bootstrap & server startup
|-- app/
|   |-- app.module.ts                # Root NestJS module
|   |-- app.controller.ts            # Health-check controller
|   |-- app.service.ts               # Health-check service
|
|-- config/                          # Environment configuration (registerAs)
|   |-- auth.config.ts
|   |-- firestore.config.ts
|   |-- line.config.ts
|   |-- mail.config.ts
|   |-- mentor.config.ts
|   |-- mentee.config.ts
|   |-- storage.config.ts
|
|-- infrastructure/                  # Shared infrastructure
|   |-- persistence/firestore/
|       |-- firestore.module.ts      # Firestore client provider
|       |-- repositories/
|           |-- firestore-base.repository.ts  # Base repository class
|
|-- modules/                         # Feature modules (DDD bounded contexts)
    |-- auth/                        # Authentication & session management
    |-- mentee/                      # Mentee profiles & registration
    |-- mentor/                      # Mentor profiles & registration
    |-- user/                        # User accounts & profile pictures
    |-- admin/                       # Admin management
    |-- mail/                        # Email service (SMTP)
    |-- storage/                     # File storage (GCS)
    |-- line/                        # LINE Login integration
    |-- master-data/                 # Reference data (faculties, topics, etc.)
```

Each feature module follows the same internal structure:

```
modules/<feature>/
|-- <feature>.module.ts              # Module root
|-- domain/                          # Domain layer (pure business logic)
|   |-- entities/                    # Domain entities (extend Entity<T>)
|   |-- repositories/               # Repository interfaces (ports)
|   |-- enums/                       # Domain enumerations
|   |-- configs/                     # Config type interfaces
|   |-- services/                    # Domain service interfaces
|
|-- application/                     # Application layer (use cases & DTOs)
|   |-- use-cases/                   # Use case implementations
|   |   |-- <use-case>/
|   |       |-- <use-case>.use-case.ts
|   |       |-- <use-case>.use-case.module.ts
|   |-- dtos/                        # Input/Output DTOs with validation
|   |-- <feature>-use-cases.module.ts
|
|-- interface/                       # Interface layer (HTTP controllers)
|   |-- <feature>.interface.module.ts
|   |-- rest/
|   |   |-- controllers/            # REST controllers
|   |   |-- guards/                  # Route guards
|   |   |-- decorators/             # Parameter decorators
|   |   |-- interceptors/           # Request/response interceptors
|   |   |-- pipes/                  # Validation pipes
|   |-- interfaces/rest/            # Controller interface contracts
|
|-- infrastructure/                  # Infrastructure layer (adapters)
    |-- persistence/firestore/
    |   |-- repositories/            # Firestore repository implementations
    |   |-- <feature>-repositories.module.ts
    |-- services/                    # External service implementations
```

---

## Architecture Layers

The project strictly follows **Clean Architecture** with four concentric layers:

| Layer | Location | Responsibility |
|-------|----------|---------------|
| **Domain** | `domain/` | Entities, repository interfaces, enums, config types. Zero framework dependencies. |
| **Application** | `application/` | Use cases (business workflows), DTOs with validation decorators. |
| **Interface** | `interface/` | REST controllers, guards, interceptors, decorators. Framework-specific. |
| **Infrastructure** | `infrastructure/` | Firestore repositories, SMTP mail, GCS storage, HTTP LINE client. |

**Dependency Rule:** Dependencies point inward only. Domain depends on nothing. Infrastructure implements domain interfaces.

---

## Module Overview

| Module | Type | Purpose |
|--------|------|---------|
| **Auth** | Feature | Email/password login, JWT token issuance, session validation, logout |
| **Mentee** | Feature | Mentee registration, profile retrieval, role-based guard |
| **Mentor** | Feature | Mentor registration, profile retrieval, role-based guard |
| **User** | Support | User account persistence, profile picture management |
| **Admin** | Feature | Admin entity and persistence (currently minimal) |
| **Mail** | Infrastructure | SMTP email sending with Handlebars templates |
| **Storage** | Infrastructure | Google Cloud Storage file operations & signed URLs |
| **Line** | Infrastructure | LINE Login OAuth2 token exchange |
| **Master Data** | Support | Reference data: faculties, topics, industries, hobbies, invite codes |

---

## High-Level Architecture Graph

```mermaid
graph TB
    subgraph Client["Client (Browser)"]
        Browser["Web App"]
    end

    subgraph API["Wise API (NestJS)"]
        direction TB
        Main["main.ts<br/>Bootstrap + CORS + Swagger"]
        AppModule["AppModule"]

        subgraph Modules["Feature Modules"]
            Auth["Auth Module"]
            Mentee["Mentee Module"]
            Mentor["Mentor Module"]
            Admin["Admin Module"]
        end

        subgraph Support["Support Modules"]
            User["User Module"]
            MasterData["Master Data Module"]
        end

        subgraph Infra["Infrastructure Modules"]
            Mail["Mail Module"]
            Storage["Storage Module"]
            Line["Line Module"]
            Firestore["Firestore Module"]
        end
    end

    subgraph External["External Services"]
        GCPFirestore["Google Cloud Firestore"]
        GCS["Google Cloud Storage"]
        SMTP["SMTP Server"]
        LineAPI["LINE API"]
    end

    Browser -->|"HTTP REST"| Main
    Main --> AppModule
    AppModule --> Auth
    AppModule --> Mentee
    AppModule --> Mentor
    AppModule --> Admin

    Auth -->|"uses"| User
    Auth -->|"uses"| Firestore
    Mentee -->|"uses"| Firestore
    Mentor -->|"uses"| Firestore
    User -->|"uses"| Firestore
    Admin -->|"uses"| Firestore
    MasterData -->|"uses"| Firestore

    Firestore -->|"read/write"| GCPFirestore
    Storage -->|"upload/download"| GCS
    Mail -->|"send"| SMTP
    Line -->|"OAuth2"| LineAPI

    style Client fill:#e1f5fe
    style API fill:#f3e5f5
    style External fill:#fff3e0
```

---

## Request Lifecycle

```mermaid
sequenceDiagram
    participant C as Client
    participant MW as NestJS Middleware<br/>(CORS, Global Prefix)
    participant G as JwtAuthGuard<br/>(Global)
    participant RG as Role Guard<br/>(MenteeGuard/MentorGuard)
    participant CT as Controller
    participant UC as Use Case<br/>(Lazy Loaded)
    participant RP as Repository<br/>(Firestore)
    participant DB as Firestore DB

    C->>MW: HTTP Request
    MW->>G: Route to Guard

    alt Public Route (@Public)
        G-->>CT: Skip auth
    else Protected Route
        G->>G: Extract token from<br/>cookie or header
        G->>UC: ValidateLoginSessionUseCase
        UC->>RP: findById(sessionId)
        RP->>DB: Query login_sessions
        DB-->>RP: Session data
        RP-->>UC: LoginSession entity
        UC->>UC: Check valid + not expired
        UC-->>G: { userId }
        G->>G: Attach userId to request
    end

    opt Role-specific Route
        RG->>RP: findByUserId(userId)
        RP->>DB: Query mentees/mentors
        DB-->>RP: Entity data
        RP-->>RG: Mentee/Mentor entity
        RG->>RG: Attach entity to request
    end

    CT->>CT: Lazy load use case module
    CT->>UC: execute(inputDTO)
    UC->>UC: Validate input (decorators)
    UC->>RP: Repository operations
    RP->>DB: Firestore CRUD
    DB-->>RP: Result
    RP-->>UC: Domain Entity
    UC->>UC: Build OutputDTO
    UC-->>CT: Result<OutputDTO>
    CT-->>C: HTTP Response
```

---

## Module Dependency Graph

```mermaid
graph LR
    subgraph Root
        AppModule
    end

    subgraph Feature["Feature Modules"]
        Auth["AuthModule"]
        Mentee["MenteeModule"]
        Mentor["MentorModule"]
        Admin["AdminModule"]
    end

    subgraph Support["Support Modules"]
        User["UserModule"]
        MasterData["MasterDataModule"]
    end

    subgraph Infrastructure["Shared Infrastructure"]
        FirestoreModule
    end

    subgraph Repositories["Repository Modules"]
        AuthRepo["AuthRepositoriesModule<br/><small>ILoginSessionRepository<br/>IMailVerificationRepository</small>"]
        MenteeRepo["MenteeRepositoriesModule<br/><small>IMenteeRepository</small>"]
        MentorRepo["MentorRepositoriesModule<br/><small>IMentorRepository</small>"]
        UserRepo["UserRepositoriesModule<br/><small>IUserRepository</small>"]
        AdminRepo["AdminRepositoriesModule<br/><small>IAdminRepository</small>"]
        MasterDataRepo["MasterDataRepositoriesModule<br/><small>IFacultyRepository</small>"]
    end

    AppModule --> Auth
    AppModule --> Mentee
    AppModule --> Mentor
    AppModule --> Admin

    Auth --> AuthRepo
    Auth -->|"LoginUserUseCase needs"| UserRepo
    Mentee --> MenteeRepo
    Mentor --> MentorRepo
    User --> UserRepo
    Admin --> AdminRepo
    MasterData --> MasterDataRepo

    AuthRepo --> FirestoreModule
    MenteeRepo --> FirestoreModule
    MentorRepo --> FirestoreModule
    UserRepo --> FirestoreModule
    AdminRepo --> FirestoreModule
    MasterDataRepo --> FirestoreModule

    style Root fill:#e8eaf6
    style Feature fill:#e8f5e9
    style Support fill:#fff8e1
    style Infrastructure fill:#fce4ec
    style Repositories fill:#f3e5f5
```

---

## Auth Module Flow

```mermaid
graph TB
    subgraph Interface["Interface Layer"]
        AuthController["AuthController<br/>/api/auth"]
        JwtAuthGuard["JwtAuthGuard<br/>(Global APP_GUARD)"]
        SetCookie["SetAccessTokenCookie<br/>Interceptor"]
        ClearCookie["ClearAccessTokenCookie<br/>Interceptor"]
    end

    subgraph Application["Application Layer"]
        LoginUC["LoginUserUseCase"]
        LogoutUC["LogoutUserUseCase"]
        ValidateUC["ValidateLoginSessionUseCase"]
        LogoutAllUC["LogoutUserAllSessionsUseCase"]
    end

    subgraph Domain["Domain Layer"]
        LoginSession["LoginSession Entity"]
        MailVerification["MailVerification Entity"]
        ILoginSessionRepo["ILoginSessionRepository"]
        IMailVerifRepo["IMailVerificationRepository"]
    end

    subgraph Infrastructure["Infrastructure Layer"]
        FSLoginSession["FirestoreLoginSession<br/>Repository"]
        FSMailVerif["FirestoreMailVerification<br/>Repository"]
    end

    subgraph External
        JwtService["JwtService<br/>(@nestjs/jwt)"]
        UserRepo["IUserRepository<br/>(from User module)"]
    end

    %% Login flow
    AuthController -->|"POST /login"| LoginUC
    LoginUC --> UserRepo
    LoginUC --> ILoginSessionRepo
    LoginUC --> JwtService
    LoginUC -->|"creates"| LoginSession
    AuthController -.->|"response"| SetCookie

    %% Logout flow
    AuthController -->|"POST /logout"| LogoutUC
    LogoutUC --> JwtService
    LogoutUC --> ILoginSessionRepo
    LogoutUC -->|"revokes"| LoginSession
    AuthController -.->|"response"| ClearCookie

    %% Validation flow (every protected request)
    JwtAuthGuard -->|"on every request"| ValidateUC
    ValidateUC --> JwtService
    ValidateUC --> ILoginSessionRepo

    %% Logout all sessions
    LogoutAllUC --> ILoginSessionRepo

    %% Infrastructure implements domain
    ILoginSessionRepo -.->|"implemented by"| FSLoginSession
    IMailVerifRepo -.->|"implemented by"| FSMailVerif

    style Interface fill:#e3f2fd
    style Application fill:#e8f5e9
    style Domain fill:#fff8e1
    style Infrastructure fill:#fce4ec
```

### Auth Endpoints

| Method | Path | Auth | Description |
|--------|------|------|-------------|
| `POST` | `/api/auth/login` | Public | Login with email + password, returns JWT in HttpOnly cookie |
| `POST` | `/api/auth/logout` | Public | Revoke current session, clear cookie |

### Auth Flow Detail

1. **Login:** Client sends `{email, password}` -> verify user in Firestore `users` collection -> hash check with argon2 -> create `LoginSession` entity -> store in `login_sessions` -> sign JWT with `{loginSessionId}` -> set `accessToken` HttpOnly cookie
2. **Request Auth:** `JwtAuthGuard` (global) extracts token from cookie/header -> decode JWT -> load `LoginSession` from Firestore -> check not revoked & not expired -> attach `userId` to request
3. **Logout:** Decode JWT (ignore expiration) -> find session -> `loginSession.revoke()` -> save -> clear cookie

---

## Mentee Module Flow

```mermaid
graph TB
    subgraph Interface["Interface Layer"]
        MenteeController["MenteeController<br/>/api/mentees"]
        MenteeGuard["MenteeGuard"]
        CurrentMentee["@CurrentMentee()<br/>Decorator"]
    end

    subgraph Application["Application Layer"]
        GetMyProfile["GetMyMenteeProfile<br/>UseCase"]
        GetById["GetMenteeById<br/>UseCase"]
        Register["RegisterMentee<br/>UseCase<br/><small>(not implemented)</small>"]
    end

    subgraph Domain["Domain Layer"]
        MenteeEntity["Mentee Entity"]
        IMenteeRepo["IMenteeRepository"]
        MenteeEnums["MenteeStatus<br/>MenteeRegistrationStatus"]
    end

    subgraph Infrastructure["Infrastructure Layer"]
        FSMentee["FirestoreMentee<br/>Repository"]
    end

    MenteeController -->|"GET /me"| GetMyProfile
    MenteeController -->|"GET /:id"| GetById
    MenteeController -.->|"guarded by"| MenteeGuard

    MenteeGuard -->|"findByUserId"| IMenteeRepo
    MenteeGuard -->|"attaches to request"| CurrentMentee

    GetMyProfile -->|"findById"| IMenteeRepo
    GetById -->|"findById"| IMenteeRepo
    Register -->|"create"| IMenteeRepo

    IMenteeRepo -.->|"implemented by"| FSMentee
    FSMentee -->|"Firestore collection"| MenteesDB["mentees"]

    IMenteeRepo -->|"returns"| MenteeEntity

    style Interface fill:#e3f2fd
    style Application fill:#e8f5e9
    style Domain fill:#fff8e1
    style Infrastructure fill:#fce4ec
```

### Mentee Endpoints

| Method | Path | Auth | Guard | Description |
|--------|------|------|-------|-------------|
| `GET` | `/api/mentees/me` | JWT | MenteeGuard | Get own mentee profile |
| `GET` | `/api/mentees/:id` | JWT | - | Get mentee by ID |

---

## Mentor Module Flow

```mermaid
graph TB
    subgraph Interface["Interface Layer"]
        MentorController["MentorController<br/>/api/mentors"]
        MentorGuard["MentorGuard"]
        CurrentMentor["@CurrentMentor()<br/>Decorator"]
    end

    subgraph Application["Application Layer"]
        GetMyProfile["GetMyMentorProfile<br/>UseCase"]
        GetById["GetMentorById<br/>UseCase"]
    end

    subgraph Domain["Domain Layer"]
        MentorEntity["Mentor Entity"]
        IMentorRepo["IMentorRepository"]
        MentorEnums["MentorStatus<br/>MentorRegistrationStatus<br/>MentorCertificationType<br/>MentorEducationDegree"]
    end

    subgraph Infrastructure["Infrastructure Layer"]
        FSMentor["FirestoreMentor<br/>Repository"]
    end

    MentorController -->|"GET /me"| GetMyProfile
    MentorController -->|"GET /:id"| GetById
    MentorController -.->|"guarded by"| MentorGuard

    MentorGuard -->|"findByUserId"| IMentorRepo
    MentorGuard -->|"attaches to request"| CurrentMentor

    GetMyProfile -->|"findById"| IMentorRepo
    GetById -->|"findById"| IMentorRepo

    IMentorRepo -.->|"implemented by"| FSMentor
    FSMentor -->|"Firestore collection"| MentorsDB["mentors"]

    IMentorRepo -->|"returns"| MentorEntity

    style Interface fill:#e3f2fd
    style Application fill:#e8f5e9
    style Domain fill:#fff8e1
    style Infrastructure fill:#fce4ec
```

### Mentor Endpoints

| Method | Path | Auth | Guard | Description |
|--------|------|------|-------|-------------|
| `GET` | `/api/mentors/me` | JWT | MentorGuard | Get own mentor profile |
| `GET` | `/api/mentors/:id` | JWT | - | Get mentor by ID |

---

## Clean Architecture Layer Graph

```mermaid
graph TB
    subgraph L4["Interface Layer (Outermost)"]
        direction LR
        Controllers["Controllers<br/><small>AuthController, MenteeController,<br/>MentorController, UserController</small>"]
        Guards["Guards<br/><small>JwtAuthGuard, MenteeGuard,<br/>MentorGuard, OwnerOrAdminGuard</small>"]
        Interceptors["Interceptors<br/><small>SetAccessTokenCookie,<br/>ClearAccessTokenCookie</small>"]
        Decorators["Decorators<br/><small>@CurrentMentee, @CurrentMentor,<br/>@Public, @AccessToken</small>"]
    end

    subgraph L3["Application Layer"]
        direction LR
        UseCases["Use Cases<br/><small>LoginUser, LogoutUser,<br/>ValidateLoginSession,<br/>GetMenteeById, GetMyMenteeProfile,<br/>GetMentorById, GetMyMentorProfile,<br/>RegisterMentee, GetProfilePicture</small>"]
        DTOs["DTOs<br/><small>Input/Output DTOs with<br/>class-validator decorators<br/>& Swagger annotations</small>"]
    end

    subgraph L2["Domain Layer (Core)"]
        direction LR
        Entities["Entities<br/><small>User, LoginSession,<br/>MailVerification, Mentee,<br/>Mentor, Admin, Faculty,<br/>Topic, Industry, Hobby,<br/>InviteCode</small>"]
        RepoInterfaces["Repository Interfaces<br/><small>IUserRepository,<br/>ILoginSessionRepository,<br/>IMenteeRepository,<br/>IMentorRepository, etc.</small>"]
        ServiceInterfaces["Service Interfaces<br/><small>IMailService,<br/>IStorageService,<br/>ILineLoginService</small>"]
        Enums["Enums<br/><small>MenteeStatus, MentorStatus,<br/>RegistrationStatus, etc.</small>"]
    end

    subgraph L1["Infrastructure Layer (Outermost)"]
        direction LR
        FirestoreRepos["Firestore Repositories<br/><small>FirestoreUserRepository,<br/>FirestoreLoginSessionRepository,<br/>FirestoreMenteeRepository,<br/>FirestoreMentorRepository, etc.</small>"]
        GCSService["GCS Storage Service"]
        SMTPService["SMTP Mail Service"]
        HTTPLine["HTTP LINE Login Service"]
    end

    L4 -->|"depends on"| L3
    L3 -->|"depends on"| L2
    L1 -->|"implements"| L2

    L4 ~~~ L1

    style L4 fill:#e3f2fd
    style L3 fill:#e8f5e9
    style L2 fill:#fff8e1
    style L1 fill:#fce4ec
```

---

## Infrastructure Services

```mermaid
graph LR
    subgraph Storage["Storage Module"]
        IStorageService["IStorageService<br/><small>(domain interface)</small>"]
        GCSStorage["GcsStorageService<br/><small>storeFile()<br/>deleteFile()<br/>getReadSignedUrl()</small>"]
    end

    subgraph Mail["Mail Module"]
        IMailService["IMailService<br/><small>(domain interface)</small>"]
        SMTPMail["SmtpMailService<br/><small>sendMail()</small>"]
        Templates["Handlebars Templates<br/><small>mentee/welcome<br/>mentor/welcome<br/>mentor/mentee-request<br/>core/reset_password</small>"]
    end

    subgraph Line["Line Module"]
        ILineLogin["ILineLoginService<br/><small>(domain interface)</small>"]
        HTTPLine["HttpLineLoginService<br/><small>getTokens()</small>"]
    end

    subgraph Firestore["Firestore Module"]
        FSProvider["FIRESTORE Provider<br/><small>new Firestore({<br/>  projectId, databaseId<br/>})</small>"]
        BaseRepo["FirestoreBaseRepository<br/><small>getDocument()<br/>getQuery()<br/>setDocument()<br/>deleteDocument()<br/>updateDocument()<br/>convertTimestamp()</small>"]
    end

    IStorageService -.->|"impl"| GCSStorage
    GCSStorage -->|"@google-cloud/storage"| GCS["Google Cloud Storage"]

    IMailService -.->|"impl"| SMTPMail
    SMTPMail -->|"@nestjs-modules/mailer"| SMTP["SMTP Server"]
    SMTPMail --> Templates

    ILineLogin -.->|"impl"| HTTPLine
    HTTPLine -->|"fetch()"| LineAPI["LINE OAuth2 API"]

    BaseRepo -->|"uses"| FSProvider

    style Storage fill:#e8f5e9
    style Mail fill:#fff3e0
    style Line fill:#e1f5fe
    style Firestore fill:#fce4ec
```

---

## Firestore Collections

| Collection | Used By | Entity | Key Fields |
|------------|---------|--------|------------|
| `users` | UserRepository, AdminRepository | `User`, `Admin` | email, password, type, status, name, contact, line |
| `login_sessions` | LoginSessionRepository | `LoginSession` | userId, createdAt, expiresAt, revoked, revokedAt, lastUsedAt |
| `mail_verifications` | MailVerificationRepository | `MailVerification` | userId, otp, createdAt, expiresAt |
| `mentees` | MenteeRepository | `Mentee` | userId, status, registrationStatus, educationProfile, documents |
| `mentors` | MentorRepository | `Mentor` | userId, invitationCodeId, status, registrationStatus, educations, certificates |
| `faculties` | FacultyRepository | `Faculty` | name (en/th), icon, sortOrder, status |

---

## Configuration System

```mermaid
graph LR
    subgraph EnvVars["Environment Variables"]
        ENV["process.env.*"]
    end

    subgraph ConfigFiles["Config Files (src/config/)"]
        AuthCfg["auth.config.ts<br/><small>registerAs('auth')</small>"]
        FirestoreCfg["firestore.config.ts<br/><small>plain object</small>"]
        LineCfg["line.config.ts<br/><small>registerAs('line')</small>"]
        MailCfg["mail.config.ts<br/><small>registerAs('mail')</small>"]
        MentorCfg["mentor.config.ts<br/><small>registerAs('mentor')</small>"]
        MenteeCfg["mentee.config.ts<br/><small>registerAs('mentee')</small>"]
        StorageCfg["storage.config.ts<br/><small>registerAs('storage')</small>"]
    end

    subgraph TypeInterfaces["Domain Config Interfaces"]
        IAuth["IAuthConfig<br/><small>jwt, accessTokenCookie,<br/>loginSession, mailVerification</small>"]
        ILine["ILineConfig<br/><small>login.channelId,<br/>login.channelSecret</small>"]
        IMail["IMailConfig<br/><small>transport, defaults</small>"]
        IMentor["IMentorConfig<br/><small>storage.staticFolder</small>"]
        IMentee["IMenteeConfig<br/><small>storage.staticFolder</small>"]
        IStorage["IStorageConfig<br/><small>projectId, bucketName,<br/>maxFileSize, readSignedUrlMaxAge</small>"]
    end

    ENV --> ConfigFiles
    AuthCfg -->|"typed as"| IAuth
    LineCfg -->|"typed as"| ILine
    MailCfg -->|"typed as"| IMail
    MentorCfg -->|"typed as"| IMentor
    MenteeCfg -->|"typed as"| IMentee
    StorageCfg -->|"typed as"| IStorage

    style EnvVars fill:#e1f5fe
    style ConfigFiles fill:#e8f5e9
    style TypeInterfaces fill:#fff8e1
```

All configs (except `firestore.config.ts`) use NestJS `registerAs()` with `ConfigurationUtils` from `@new-api-core/shared` for type-safe environment variable resolution with defaults.

---

## Lazy Loading Pattern

Use cases are **lazily loaded** at runtime using NestJS `LazyModuleLoader`. Controllers extend `LazyBaseController` from `@new-api-core/interface/nestjs`.

```mermaid
sequenceDiagram
    participant C as Controller
    participant LML as LazyModuleLoader
    participant UCM as UseCaseModule
    participant UC as UseCase

    C->>C: this.load<UseCase>(<br/>  moduleImport,<br/>  useCaseImport<br/>)
    C->>LML: Dynamic import() module
    LML->>UCM: Load & register module
    UCM-->>LML: Module reference
    LML->>UC: moduleRef.get(UseCase)
    UC-->>C: UseCase instance

    Note over C,UC: Module is cached after first load.<br/>Subsequent requests reuse the instance.

    C->>UC: execute(inputDTO)
    UC->>UC: @ValidateInput + @UseResult
    UC-->>C: Result<OutputDTO>
```

This pattern provides:
- **Reduced startup time:** Modules are loaded on-demand, not at bootstrap
- **Code splitting:** Webpack can split use case modules into separate chunks
- **Memory efficiency:** Unused use cases don't consume memory

The `JwtAuthGuard` also uses this pattern for `ValidateLoginSessionUseCase`, caching the instance after first resolution.

---

## Complete System Flow (End-to-End)

```mermaid
graph TB
    subgraph Client
        Browser["Web Browser"]
    end

    subgraph NestJS["NestJS Application"]
        subgraph Bootstrap["Bootstrap (main.ts)"]
            CORS["CORS Middleware"]
            Swagger["Swagger (dev/staging)"]
            GlobalPrefix["/api prefix"]
        end

        subgraph AuthFlow["Auth Pipeline"]
            JwtGuard["JwtAuthGuard<br/>(Global)"]
            RoleGuard["Role Guards<br/>(MenteeGuard / MentorGuard)"]
        end

        subgraph Controllers["REST Controllers"]
            AC["AuthController<br/>POST /auth/login<br/>POST /auth/logout"]
            MC["MenteeController<br/>GET /mentees/me<br/>GET /mentees/:id"]
            MTC["MentorController<br/>GET /mentors/me<br/>GET /mentors/:id"]
            UC["UserController<br/>GET /users/:id/profile-picture"]
        end

        subgraph UseCases["Use Cases (Lazy Loaded)"]
            Login["LoginUser"]
            Logout["LogoutUser"]
            Validate["ValidateLoginSession"]
            GetMentee["GetMenteeById<br/>GetMyMenteeProfile"]
            GetMentor["GetMentorById<br/>GetMyMentorProfile"]
            GetPic["GetProfilePicture"]
        end

        subgraph Repositories["Repositories"]
            UserRepo["UserRepo<br/>(users)"]
            SessionRepo["SessionRepo<br/>(login_sessions)"]
            MenteeRepo["MenteeRepo<br/>(mentees)"]
            MentorRepo["MentorRepo<br/>(mentors)"]
        end
    end

    subgraph ExternalServices["External Services"]
        Firestore["Google Cloud<br/>Firestore"]
        GCS["Google Cloud<br/>Storage"]
        SMTP["SMTP<br/>Email Server"]
        LINE["LINE<br/>OAuth2 API"]
    end

    Browser -->|"HTTP"| CORS
    CORS --> GlobalPrefix
    GlobalPrefix --> JwtGuard
    JwtGuard -->|"public"| AC
    JwtGuard -->|"authenticated"| RoleGuard
    JwtGuard -->|"authenticated"| UC
    RoleGuard --> MC
    RoleGuard --> MTC

    AC --> Login
    AC --> Logout
    JwtGuard --> Validate
    MC --> GetMentee
    MTC --> GetMentor
    UC --> GetPic

    Login --> UserRepo
    Login --> SessionRepo
    Validate --> SessionRepo
    Logout --> SessionRepo
    GetMentee --> MenteeRepo
    GetMentor --> MentorRepo
    GetPic --> UserRepo

    UserRepo --> Firestore
    SessionRepo --> Firestore
    MenteeRepo --> Firestore
    MentorRepo --> Firestore

    GetPic -.->|"signed URL"| GCS

    style Client fill:#e1f5fe
    style NestJS fill:#f3e5f5
    style ExternalServices fill:#fff3e0
```
