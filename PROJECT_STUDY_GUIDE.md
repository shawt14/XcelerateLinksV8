# XcelerateLinks – Comprehensive Project Study Guide

> **For presentation prep.** This document covers every class, controller, service, and flow in the project. Use the table of contents to jump to any section.

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Solution Structure](#2-solution-structure)
3. [Technology Stack](#3-technology-stack)
4. [Role System](#4-role-system)
5. [Authentication & Login Flow](#5-authentication--login-flow)
6. [Session Management](#6-session-management)
7. [Database & Models](#7-database--models)
8. [API Project – Controllers](#8-api-project--controllers)
9. [API Project – Services](#9-api-project--services)
10. [API Project – Middleware](#10-api-project--middleware)
11. [API Project – SignalR Hub (Real-time Chat)](#11-api-project--signalr-hub-real-time-chat)
12. [MVC (Web) Project – Controllers](#12-mvc-web-project--controllers)
13. [MVC (Web) Project – Services](#13-mvc-web-project--services)
14. [Subscription Plans & Limits](#14-subscription-plans--limits)
15. [Match Score Algorithm](#15-match-score-algorithm)
16. [Notification System](#16-notification-system)
17. [File Upload System](#17-file-upload-system)
18. [Key Data Flow Diagrams](#18-key-data-flow-diagrams)

---

## 1. Project Overview

**XcelerateLinks** is a professional networking and job-matching platform — think LinkedIn for a school project, written in ASP.NET Core. It connects:

- **Job Seekers** – create profiles, apply for jobs, manage connections, chat.
- **Employers** – post job opportunities, review applications, manage interview rounds.
- **Admins** – manage all users, approve employer requests, oversee the platform.

The project is split into **two applications** that talk to each other over HTTP:

| App | Type | Purpose |
|-----|------|---------|
| `APIPSI16` | ASP.NET Core Web API | All business logic, data access, JWT authentication |
| `XcelerateLinks` | ASP.NET Core MVC | Web frontend (Razor Views), calls the API |

---

## 2. Solution Structure

```
APIPSI16_6_5/
├── APIPSI16/                   ← REST API (backend)
│   ├── Controllers/            ← All API endpoints
│   ├── Data/                   ← DbContext (EF Core)
│   ├── Filters/                ← Swagger helpers
│   ├── Hubs/                   ← SignalR real-time chat
│   ├── Middleware/             ← Session validation middleware
│   ├── Migrations/             ← EF Core database migrations
│   ├── Models/                 ← Entity classes + DTOs
│   │   └── DTOs/               ← Data transfer objects
│   ├── Services/               ← Business logic services
│   └── Program.cs              ← App startup & DI configuration
│
├── XcelerateLinks/             ← MVC Web App (frontend)
│   ├── Controllers/            ← MVC controllers (call the API)
│   ├── Models/ViewModels/      ← View-specific models
│   ├── Services/               ← HTTP clients, token handling
│   ├── Views/                  ← Razor (.cshtml) pages
│   └── Program.cs              ← App startup & cookie auth
│
└── XcelerateLinks_DTOs/        ← Shared DTO library (UserDTO)
```

---

## 3. Technology Stack

| Layer | Technology |
|-------|-----------|
| Backend API | ASP.NET Core 8 Web API |
| Frontend | ASP.NET Core 8 MVC (Razor Views) |
| Database ORM | Entity Framework Core 8 (SQL Server) |
| Authentication (API) | JWT Bearer Tokens |
| Authentication (MVC) | Cookie Authentication (stores the JWT in a cookie) |
| Real-time | SignalR (WebSockets) |
| Password hashing | ASP.NET Core Identity `PasswordHasher<T>` |
| Swagger/OpenAPI | Swashbuckle |
| Email (password reset) | Custom `IEmailSender` |

---

## 4. Role System

Roles are stored as an integer (`Role` column) on the `User` table:

| Value | Role | Description |
|-------|------|-------------|
| `0` | **Admin** | Full access to everything |
| `1` | **User** (Job Seeker) | Default role on registration |
| `2` | **Employer** | Can post jobs, approve members |
| `3` | **Pending Employer** | Requested employer role, awaiting approval |

**Company Member roles** (separate table `CompanyMembers.Role`):

| Value | Role |
|-------|------|
| `0` | Pending |
| `1` | Recruiter |
| `2` | HR Manager |
| `3` | Company Admin |

---

## 5. Authentication & Login Flow

### Overview

The API issues **JWT tokens**. The MVC site stores the token in an **HttpOnly cookie** and also creates an ASP.NET **cookie authentication** session for Razor view access.

### Step-by-step Login Flow

```
User submits login form
        ↓
[MVC] AccountController.Login()
        ↓ HTTP POST to API
[API]  AuthController.Login()
    1. Finds user by email OR username (case-insensitive)
    2. Verifies password with PasswordHasher.VerifyHashedPassword()
    3. Invalidates all previous active sessions for that user
    4. Creates JWT token (via TokenService.CreateToken())
    5. Creates a Session record in the DB (via SessionService.CreateSessionAsync())
    6. Returns { token, expiresAt }
        ↓
[MVC]  Receives token
    1. Decodes JWT to extract userId, username, role claims
    2. Calls SessionService.CreateSessionAsync() again (MVC creates its own session record)
    3. Sets HttpOnly cookie "ApiAccessToken" = JWT token
    4. Signs in with ASP.NET Cookie authentication (ClaimsPrincipal)
    5. Redirects to home (or returnUrl)
```

### JWT Token Structure

The token contains these **claims**:
- `sub` (subject) = userId
- `jti` = unique token ID (GUID)
- `ClaimTypes.Name` = user's name
- `ClaimTypes.NameIdentifier` = userId
- `ClaimTypes.Role` = role number (0, 1, or 2)

**Token settings** (from `appsettings.json` / env vars):
- Key: base64-encoded secret (`XCELERATE_JWT_KEY` env var or `Jwt:Key`)
- Issuer: `xcelerate-links-api`
- Audience: `xcelerate-links-clients`
- Expiry: 60 minutes (configurable via `Jwt:ExpireMinutes`)
- Algorithm: `HmacSha256`
- Clock skew: 30 seconds tolerance

### How the MVC site attaches the token to API calls

`TokenHandler` (a `DelegatingHandler`) runs on every outgoing HTTP request from the MVC app:

```csharp
// Reads the "ApiAccessToken" cookie and injects it as a Bearer token header
request.Headers.Authorization = new AuthenticationHeaderValue("Bearer", token);
```

This means all `HttpClient` calls from MVC automatically include the JWT.

### Logout Flow

```
User clicks Logout
        ↓
[MVC] AccountController.Logout()
    1. Reads "ApiAccessToken" cookie
    2. Calls SessionService.InvalidateSessionAsync(token) → marks Session.IsActive = false
    3. Deletes the cookie
    4. Signs out of ASP.NET cookie auth
        ↓
(Separately, the API has its own logout endpoint)
[API] AuthController.Logout()
    1. Reads Authorization header
    2. Calls SessionService.InvalidateSessionAsync(token)
```

---

## 6. Session Management

Sessions provide server-side tracking **on top of** JWT tokens. This allows force-logout and single-session enforcement.

### Session Model (`Models/Session.cs`)

| Property | Type | Description |
|----------|------|-------------|
| `SessionId` | int | Primary key |
| `UserId` | int | FK to Users |
| `Token` | string | The full JWT string |
| `CreatedAt` | DateTime | When session started |
| `ExpiresAt` | DateTime | When session expires |
| `IsActive` | bool | `false` = logged out or invalidated |
| `InvalidatedAt` | DateTime? | When it was invalidated |

### `ISessionService` interface & `SessionService` class

**`CreateSessionAsync(userId, token, expiresAt)`**
- First: marks all existing active sessions for the user as inactive (enforces single session)
- Then: inserts a new `Session` row with `IsActive = true`

**`IsSessionValidAsync(userId, token)`**
- Checks that a session exists for this user+token, is still `IsActive`, and hasn't expired
- Used by the (currently disabled) middleware

**`InvalidateSessionAsync(token)`**
- Finds the session by token, sets `IsActive = false` and `InvalidatedAt = now`
- Called on logout

**`InvalidateAllUserSessionsAsync(userId)`**
- Marks all of a user's active sessions as inactive
- Called at login start (to kick out old sessions) and in MVC login

### `SessionValidationMiddleware`

> ⚠️ **Currently disabled in production** (`app.UseMiddleware<SessionValidationMiddleware>()` is commented out in `Program.cs`)

When enabled, it runs on every request and:
1. Skips `/api/auth/login`, `/api/auth/register`, `/api/auth/login-debug` routes
2. Extracts `userId` from JWT claims and `token` from `Authorization` header
3. Looks up the session in the DB
4. Returns `401` if: session not found, session inactive, or session expired

---

## 7. Database & Models

The database context is `xcleratesystemslinks_SampleDBContext` (EF Core).

### Complete Model Reference

---

#### `User`
The central entity. Everything connects back to a User.

| Property | Type | Notes |
|----------|------|-------|
| `UserId` | int | PK, auto-increment |
| `Name` | string? | Display name |
| `Email` | string? | Login identifier |
| `Username` | string? | Alternate login handle (`@john_doe`) |
| `PhoneNumber` | string? | |
| `PasswordHash` | string? | BCrypt-style hash via Identity |
| `Role` | int? | 0=Admin, 1=User, 2=Employer, 3=Pending |
| `Nationality` | int? | FK to Nationalities lookup |
| `JobPreference` | int? | Used for opportunity matching |
| `ProfileBio` | string? | |
| `DoB` | DateOnly? | |
| `Location` | string? | Legacy text field |
| `LocationId` | int? | FK to Locations table |
| `CountryId` | int? | FK to Countries table |
| `ProfilePictureUrl` | string? | Path in wwwroot/uploads/profiles |
| `BannerUrl` | string? | Path in wwwroot/uploads/banners |
| `SubscriptionPlan` | int | 0=Free, 1=Pro, 2=Enterprise |
| `IsOpenToWork` | bool | Whether seeking jobs |
| `EmployerRequestDocumentUrl` | string? | Document uploaded for employer request |
| `EmployerRequestNote` | string? | Note with employer request |

**Navigation collections**: `AuditLogs`, `ChatMessages`, `ChatUsers`, `Chats`, `CompanyMembers`, `JobApplications`, `Notifications`, `Opportunities`, `Posts`, `ProfileEducations`, `ProfileExperiences`, `Sessions`, `Skills`, `UserSkills`

---

#### `Session`
Server-side session record (see Section 6).

---

#### `Company`

| Property | Type | Notes |
|----------|------|-------|
| `CompanyId` | int | PK |
| `Name` | string | Required |
| `Industry` | string? | |
| `Location` | string? | |
| `CreatedAt` | DateTime? | |
| `CompanyLogoUrl` | string? | |

**Navigation**: `Opportunities`, `CompanyMembers`, `EmployerCandidateHistories`

---

#### `CompanyMember`
Junction between `Company` and `User`, with a role.

| Property | Type | Notes |
|----------|------|-------|
| `CompanyMemberId` | int | PK |
| `CompanyId` | int | FK |
| `UserId` | int | FK |
| `Role` | int | 0=Pending, 1=Recruiter, 2=HRManager, 3=CompanyAdmin |
| `StartDate` | DateOnly? | |
| `Title` | string? | Job title within company |

---

#### `Opportunity`
A job opening posted by an employer.

| Property | Type | Notes |
|----------|------|-------|
| `Id` | int | PK |
| `Title` | string? | Job title |
| `CreatorId` | int? | FK to User who created it |
| `CompanyId` | int? | FK to Company |
| `EmploymentType` | byte? | e.g. full-time, part-time |
| `SeniorityLevel` | byte? | |
| `Location` | string? | Legacy text |
| `LocationId` | int? | Structured location |
| `CountryId` | int? | |
| `RemoteOption` | byte? | |
| `OpportunityType` | byte? | 0=Standard, 1=Guided, 2=LongTerm |
| `ApplicationScope` | byte? | 0=External, 1=Internal, 2=Mixed |
| `RequiredJobRoleIds` | string? | Comma-separated JobRole IDs for matching |

---

#### `JobApplication`
Represents a user applying to an opportunity.

| Property | Type | Notes |
|----------|------|-------|
| `JobApplicationId` | int | PK |
| `OpportunityId` | int | FK |
| `UserId` | int | FK (applicant) |
| `Status` | byte | 0=Pending, 1=Reviewed, 2=Interview, 3=Offered, 4=Rejected, etc. |
| `AppliedAt` | DateTime | |
| `UpdatedAt` | DateTime? | |
| `Name` | string? | Applicant's display name |
| `CoverLetter` | string? | |
| `PhoneNumber` | string? | |
| `ProfessionalUrl` | string? | LinkedIn/portfolio |
| `PortfolioUrl` | string? | |
| `YearsOfExperience` | int? | |
| `OpenToRemote` | bool? | |
| `SelectedJobRoleIds` | string? | Comma-separated selected roles |
| `ApplicantResponse` | byte? | Response to employer offer |
| `LatestEmployerMessage` | string? | Message from employer |

**Navigation**: `Opportunity`, `User`, `InterviewRounds`

---

#### `InterviewRound`
A round of interviews for a job application.

| Property | Type | Notes |
|----------|------|-------|
| `InterviewRoundId` | int | PK |
| `JobApplicationId` | int | FK |
| `InterviewerUserId` | int | FK to interviewer |
| `ScheduledAt` | DateTime? | When interview is scheduled |
| `Notes` | string? | |
| `Status` | byte? | |

---

#### `Connection`
A LinkedIn-style connection between two users.

| Property | Type | Notes |
|----------|------|-------|
| `ConnectionId` | int | PK |
| `RequesterUserId` | int | Who sent the request |
| `AddresseeUserId` | int | Who received it |
| `Status` | byte | 0=Pending, 1=Accepted |
| `CreatedAt` | DateTime | |
| `AcceptedAt` | DateTime? | |

---

#### `Post`

| Property | Type | Notes |
|----------|------|-------|
| `PostId` | int | PK |
| `UserId` | int | FK (author) |
| `Content` | string? | Post text |
| `CreatedAt` | DateTime | |
| `UpdatedAt` | DateTime? | |
| `Visibility` | byte | 0=Public, 1=Friends only, 2=Private |

**Navigation**: `PostComments`, `PostReactions`, `User`

---

#### `PostComment`

| Property | Type | Notes |
|----------|------|-------|
| `CommentId` | int | PK |
| `PostId` | int | FK |
| `UserId` | int | FK (commenter) |
| `Content` | string? | |
| `CreatedAt` | DateTime | |
| `ParentCommentId` | int? | For nested/reply comments |
| `IsModerated` | bool | |
| `IsDeleted` | bool | Soft-delete flag |

---

#### `PostReaction`
Like/emoji reaction to a post.

| Property | Type | Notes |
|----------|------|-------|
| `ReactionId` | int | PK |
| `PostId` | int | FK |
| `UserId` | int | FK |
| `ReactionType` | byte? | 0=Like, etc. |
| `CreatedAt` | DateTime | |

---

#### `Chat`
A conversation thread (1-to-1 or group).

| Property | Type | Notes |
|----------|------|-------|
| `ChatId` | int | PK |
| `Type` | byte | 0=Direct, 1=Group |
| `CreatedAt` | DateTime | |
| `CreatedByUserId` | int? | FK |

**Navigation**: `ChatUsers`, `ChatMessages`, `CreatedByUser`

---

#### `ChatUser`
Who participates in a chat.

| Property | Type | Notes |
|----------|------|-------|
| `ChatUserId` | int | PK |
| `ChatId` | int | FK |
| `UserId` | int | FK |
| `JoinedAt` | DateTime | |
| `Role` | byte? | e.g. Admin of group chat |

---

#### `ChatMessage`

| Property | Type | Notes |
|----------|------|-------|
| `MessageId` | int | PK |
| `ChatId` | int | FK |
| `SenderUserId` | int | FK |
| `MessageText` | string? | |
| `CreatedAt` | DateTime | |
| `DeliveredAt` | DateTime? | |
| `ReadAt` | DateTime? | When recipient read it |

---

#### `Notification`

| Property | Type | Notes |
|----------|------|-------|
| `NotificationId` | int | PK |
| `UserId` | int | FK (recipient) |
| `ActorUserId` | int? | FK (who triggered it) |
| `Type` | string | "ConnectionRequest", "JobApplied", "PostComment", etc. |
| `Payload` | string? | JSON string with extra context |
| `IsRead` | bool | |
| `CreatedAt` | DateTime | |

---

#### `Skill` & `UserSkill`
- `Skill`: global list of skills (`SkillId`, `Name`)
- `UserSkill`: links a `User` to a `Skill`, tracks `EndorsementCount` and `AddedAt`

---

#### `SkillEndorsement`
When another user endorses someone's skill.

| Property | Type | Notes |
|----------|------|-------|
| `EndorsementId` | int | PK |
| `UserSkillId` | int | FK to UserSkill |
| `EndorserUserId` | int | FK to endorser |
| `CreatedAt` | DateTime | |

---

#### `ProfileExperience`
Work history entry on a user's profile.

| Property | Type | Notes |
|----------|------|-------|
| `ExperienceId` | int | PK |
| `UserId` | int | FK |
| `Title` | string? | Job title |
| `CompanyName` | string? | |
| `Location` | string? | |
| `StartDate` | DateOnly? | |
| `EndDate` | DateOnly? | null = current |
| `IsCurrent` | bool? | |
| `Description` | string? | |

---

#### `ProfileEducation`
Education history entry.

| Property | Type | Notes |
|----------|------|-------|
| `EducationId` | int | PK |
| `UserId` | int | FK |
| `School` | string? | Institution name |
| `Degree` | string? | |
| `FieldOfStudy` | string? | |
| `StartYear` | int? | |
| `EndYear` | int? | |

---

#### `Rating`
User/company ratings.

| Property | Type | Notes |
|----------|------|-------|
| `RatingId` | int | PK |
| `RatedByUserId` | int | FK (who rated) |
| `RatedEntityId` | int | ID of what was rated |
| `EntityType` | string | "User", "Company", etc. |
| `Score` | int | 1–5 |
| `Review` | string? | Text review |
| `CreatedAt` | DateTime? | |
| `UpdatedAt` | DateTime? | |

---

#### `AuditLog`
Tracks sensitive admin actions.

| Property | Type | Notes |
|----------|------|-------|
| `AuditLogId` | int | PK |
| `UserId` | int | Who performed the action |
| `Action` | string | e.g. "ApproveEmployerRole" |
| `TargetType` | string? | e.g. "User" |
| `TargetId` | int? | ID of target |
| `CreatedAt` | DateTime | |

---

#### `UserJobPreference`
Many-to-many: which `JobRole`s a user is interested in.

| Property | Type | Notes |
|----------|------|-------|
| `PreferenceId` | int | PK |
| `UserId` | int | FK |
| `JobRoleId` | int | FK |

---

#### `JobRole`
Lookup table of job roles (e.g., "Software Engineer", "Designer").

---

#### `Location` / `Country`
Structured location lookup tables used for matching.

---

#### `EmployerCandidateHistory`
Records when an employer viewed/contacted a candidate for a specific opportunity.

---

#### `SkillValidationRequest`
Used when a user requests validation of a skill (admin workflow).

---

## 8. API Project – Controllers

All API controllers live in `APIPSI16/Controllers/`. They share a base class:

### `ApiControllerBase`
Base class for controllers that need session validation (Companies, etc.).

```csharp
protected async Task<IActionResult?> ValidateSessionAsync()
// Reads userId + token from HttpContext, calls ISessionService.IsSessionValidAsync()
// Returns 401 if invalid, null if valid (caller continues)

protected int? GetCurrentUserId()
// Reads ClaimTypes.NameIdentifier from JWT

protected string? GetCurrentUserRole()
// Reads ClaimTypes.Role from JWT
```

---

### `AuthController` — `/api/auth`

Handles authentication. **No `[Authorize]` needed on anonymous endpoints.**

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| POST | `/api/auth/login` | Anonymous | Verifies credentials, creates session, returns JWT |
| POST | `/api/auth/logout` | Required | Invalidates current session |
| POST | `/api/auth/register` | Anonymous | Creates new user (role forced to 1 unless admin caller) |
| POST | `/api/auth/reset-password` | Anonymous | Resets password by email |
| POST | `/api/auth/login-debug-verify` | Anonymous | Debug: verify password without creating session |
| POST | `/api/auth/login-debug-token` | Anonymous | Debug: generate admin token for testing |

**Key logic in `Login()`:**
1. Normalise username to lowercase, find user by email OR username
2. `PasswordHasher.VerifyHashedPassword()` → if `SuccessRehashNeeded`, update the hash
3. Invalidate all previous sessions
4. Generate JWT via `TokenService.CreateToken()`
5. Persist session via `SessionService.CreateSessionAsync()`
6. Return `{ token, expiresAt }`

---

### `UsersController` — `/api/users`

Manages user profiles and admin actions. Requires `[Authorize]`.

| Method | Route | Who | Description |
|--------|-------|-----|-------------|
| GET | `/api/users` | Admin only (no filters); Admin+Employer (with filters) | List users, optionally filter by `jobPreference` or `nationality` |
| GET | `/api/users/{id}` | Admin or own user | Get single user (returns `UserDTO`) |
| GET | `/api/users/{id}/profile` | Any authenticated | Full profile: skills, experiences, educations, job preferences |
| POST | `/api/users/{id}/upload-picture` | Admin or own user | Upload profile picture |
| POST | `/api/users/{id}/upload-banner` | Admin or own user | Upload banner image |
| POST | `/api/users/me/request-employer` | Any user | Submit employer role request with document |
| POST | `/api/users/{id}/approve-employer` | Admin, CompanyAdmin | Approve employer request, optionally set company role |
| POST | `/api/users/{id}/reject-employer` | Admin, CompanyAdmin | Reject employer request |
| GET | `/api/users/pending-employers` | Admin, Employer | List users with role=3 (pending) |
| GET | `/api/users/stats` | Public | Platform stats: user count, company count, opportunities, connections |
| GET | `/api/users/network` | Any | People discovery — users with shared connections |
| GET | `/api/users/lookups/nationalities` | Any | Nationality lookup list |
| GET | `/api/users/lookups/jobroles` | Any | Job roles lookup list |
| PUT | `/api/users/{id}` | Admin or own user | Update user profile |
| DELETE | `/api/users/{id}` | Admin only | Delete user |
| GET | `/api/users/employer-matches` | Employer only | Find candidates matching posted opportunities |

---

### `OpportunitiesController` — `/api/opportunities`

Manages job postings.

| Method | Route | Who | Description |
|--------|-------|-----|-------------|
| GET | `/api/opportunities` | Any authenticated | List all opportunities |
| GET | `/api/opportunities/{id}` | Any authenticated | Get single opportunity with company info |
| GET | `/api/opportunities/recommended` | Any authenticated | Personalised recommendations based on user's `JobPreference` |
| POST | `/api/opportunities` | Admin, Employer | Create a new opportunity (employer must be company member) |
| PUT | `/api/opportunities/{id}` | Admin, Employer | Update opportunity |
| DELETE | `/api/opportunities/{id}` | Admin, Employer | Delete opportunity |
| GET | `/api/opportunities/{id}/match-score` | Any authenticated | Compute match score for current user vs. this opportunity |
| GET | `/api/opportunities/{id}/applicants` | Admin, Employer | List applicants with optional match scores |

**On creation:** sends notifications to users whose job preferences perfectly match the new opportunity.

---

### `JobApplicationsController` — `/api/jobapplications`

Manages the application lifecycle.

| Method | Route | Who | Description |
|--------|-------|-----|-------------|
| POST | `/api/jobapplications/apply` | Any user | Apply to an opportunity |
| GET | `/api/jobapplications/{id}` | Applicant, Admin, Company member | Get single application with interview rounds |
| POST | `/api/jobapplications/{id}/status` | Employer/Admin | Update application status |
| GET | `/api/jobapplications/my` | Current user | List own applications |
| POST | `/api/jobapplications/{id}/schedule` | Employer/Admin | Schedule an interview round |
| POST | `/api/jobapplications/{id}/respond` | Applicant | Respond to an offer |
| POST | `/api/jobapplications/{id}/employer-message` | Employer/Admin | Send message to applicant |

**Free plan limit:** Max 5 applications per calendar month. Returns `429` if exceeded.

**Status values:** 0=Pending, 1=Reviewed, 2=Interview, 3=Offered, 4=Rejected (and others depending on flow)

---

### `CompaniesController` — `/api/companies`

Manages company profiles.

| Method | Route | Who | Description |
|--------|-------|-----|-------------|
| GET | `/api/companies` | Any authenticated | List all companies |
| GET | `/api/companies/{id}` | Any authenticated | Get single company |
| GET | `/api/companies/{id}/profile` | Any authenticated | Full profile: members + opportunities |
| POST | `/api/companies` | Admin, Employer | Create a company |
| PUT | `/api/companies/{id}` | Admin, CompanyAdmin | Update company |
| DELETE | `/api/companies/{id}` | Admin only | Delete company |
| POST | `/api/companies/{id}/upload-logo` | Admin, CompanyAdmin | Upload company logo |

**Note:** Uses `ValidateSessionAsync()` from `ApiControllerBase` for extra session checks.

---

### `CompanyMembersController` — `/api/companymembers`

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/companymembers/company/{companyId}` | List members of a company |
| POST | `/api/companymembers` | Add a member to a company (Admin/CompanyAdmin) |
| PUT | `/api/companymembers/{id}` | Update member role |
| DELETE | `/api/companymembers/{id}` | Remove member |

---

### `ConnectionsController` — `/api/connections`

Manages LinkedIn-style connections.

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/connections/request` | Send connection request (creates notification) |
| POST | `/api/connections/{id}/accept` | Accept a pending request (creates notification) |
| GET | `/api/connections/{id}` | Get a connection (AllowAnonymous) |
| GET | `/api/connections/my` | My accepted connections |
| GET | `/api/connections/my-with-users` | Accepted connections with user details (optimised JOIN) |
| GET | `/api/connections/pending` | Incoming pending requests with requester details |
| DELETE | `/api/connections/{id}` | Remove/decline a connection |

**Free plan limit:** Max 50 accepted connections. Returns `429` if exceeded.

---

### `ChatController` — `/api/chat`

REST endpoints for chat (real-time is handled by the SignalR Hub).

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/chat` | List my chats (admin sees all) |
| GET | `/api/chat/{id}` | Get chat detail with participants and messages |
| GET | `/api/chat/{id}/messages` | Paginated message history |
| POST | `/api/chat` | Create a new chat |
| POST | `/api/chat/with-participants` | Create chat and add participants in one step |
| POST | `/api/chat/{id}/participants` | Add participant to existing chat |
| DELETE | `/api/chat/{id}/participants/{userId}` | Remove participant |

---

### `ChatUsersController` — `/api/chatusers`

Supplementary endpoints for managing chat participants.

---

### `PostsController` — `/api/posts`

| Method | Route | Auth | Description |
|--------|-------|------|-------------|
| GET | `/api/posts/{id}` | Anonymous | Get a single post with comments and reactions |
| GET | `/api/posts/feed` | Required | Personalised feed: posts from connections + public posts |
| POST | `/api/posts` | Required | Create a post |
| PUT | `/api/posts/{id}` | Own user | Edit a post |
| DELETE | `/api/posts/{id}` | Own user or Admin | Delete a post |

**Feed logic:** Shows posts that are Public (`Visibility=0`), or owned by the user, or Friends-only (`Visibility=1`) where the author is a connection.

---

### `PostCommentsController` — `/api/postcomments`

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/postcomments` | Add a comment (supports nested replies via `ParentCommentId`), notifies post owner |
| GET | `/api/postcomments/{id}` | Get a comment (AllowAnonymous) |
| DELETE | `/api/postcomments/{id}` | Soft-delete (sets `IsDeleted = true`) |

---

### `PostReactionsController` — `/api/postreactions`

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/postreactions` | Add/change a reaction to a post |
| DELETE | `/api/postreactions/{id}` | Remove a reaction |
| GET | `/api/postreactions/post/{postId}` | Get all reactions for a post |

---

### `NotificationsController` — `/api/notifications`

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/notifications/my` | Get latest 100 notifications for current user |
| POST | `/api/notifications/{id}/markread` | Mark a notification as read |

---

### `RatingsController` — `/api/ratings`

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/ratings` | Create or update (upsert) a rating for a User or Company |
| PUT | `/api/ratings/{id}` | Update a specific rating (own rating or admin) |
| GET | `/api/ratings/my/{entityType}/{entityId}` | Check if current user rated this entity |
| GET | `/api/ratings/{entityType}/{entityId}` | Get all ratings + average for an entity |

---

### `SkillsController` — `/api/skills`

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/skills` | Get all skills in the system |
| GET | `/api/skills/{id}` | Get a specific skill |
| POST | `/api/skills` | Create a skill (Admin only) |
| DELETE | `/api/skills/{id}` | Delete a skill (Admin only) |

---

### `UserSkillsController` — `/api/userskills`

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/userskills/user/{userId}` | Get skills for a user (AllowAnonymous) |
| POST | `/api/userskills` | Add a skill to a user |
| POST | `/api/userskills/{userSkillId}/endorse` | Endorse someone's skill |
| DELETE | `/api/userskills/{id}` | Remove a skill |
| GET | `/api/userskills/{userSkillId}/validation-requests` | Get validation requests for a skill |
| POST | `/api/userskills/{userSkillId}/request-validation` | Request skill validation |
| POST | `/api/userskills/{userSkillId}/validate` | Admin validates a skill |

---

### `SkillEndorsementsController` — `/api/skillendorsements`

| Method | Route | Description |
|--------|-------|-------------|
| POST | `/api/skillendorsements` | Endorse a user's skill |
| DELETE | `/api/skillendorsements/{id}` | Remove an endorsement |

---

### `ProfileExperiencesController` — `/api/profileexperiences`

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/profileexperiences/user/{userId}` | Get experiences for a user (AllowAnonymous), ordered by StartDate desc |
| POST | `/api/profileexperiences` | Add experience (own or admin) |
| PUT | `/api/profileexperiences/{id}` | Update experience (own or admin) |
| DELETE | `/api/profileexperiences/{id}` | Delete experience (own or admin) |

---

### `ProfileEducations` — `/api/profileeducations`

Same CRUD pattern as ProfileExperiences but for education entries.

---

### `InterviewRoundsController` — `/api/interviewrounds`

| Method | Route | Description |
|--------|-------|-------------|
| GET | `/api/interviewrounds/application/{applicationId}` | Get rounds for an application |
| POST | `/api/interviewrounds` | Create interview round (Employer/Admin) |
| PUT | `/api/interviewrounds/{id}` | Update round (Employer/Admin) |
| DELETE | `/api/interviewrounds/{id}` | Delete round |

---

## 9. API Project – Services

### `TokenService` (Singleton)

Registered as `ITokenService`. Creates and signs JWT tokens.

**`CreateToken(subject, additionalClaims)`**
- Reads JWT key from env var `XCELERATE_JWT_KEY` (falls back to `Jwt:Key`)
- Reads `Jwt:Issuer`, `Jwt:Audience`, `Jwt:ExpireMinutes` (default: 60)
- Creates HMAC-SHA256 signed token
- Stores expiry in `_lastExpiry` for caller to retrieve

**`GetLastExpiry()`** — returns the expiry of the most recently created token.

---

### `SessionService` (Scoped)

Registered as `ISessionService`. See Section 6 for full details.

---

### `FileStorageService` (Scoped)

Registered as `IFileStorageService`. Handles file uploads to `wwwroot/uploads/`.

**`SaveFileAsync(file, folder)`**
- Generates a GUID filename to prevent conflicts
- Saves to `wwwroot/uploads/{folder}/{guid}.{ext}`
- Returns the relative URL

**`DeleteFileAsync(fileUrl)`**
- Deletes the physical file if it exists

**`ValidateImageFile(file, out errorMessage)`**
- Max size: 5 MB
- Allowed extensions: `.jpg`, `.jpeg`, `.png`, `.gif`

**`ValidateDocumentOrImageFile(file, out errorMessage)`**
- Same as above but also allows `.pdf`

---

### `MatchScoreHelper` (Static)

Utility class for computing candidate-opportunity match scores. See Section 15.

---

### `InputValidation` (Static)

Contains shared validation constants, e.g. `PhonePattern` regex for phone number validation.

---

## 10. API Project – Middleware

### `SessionValidationMiddleware`

> Currently **disabled** in `Program.cs`. Kept for future use.

**Purpose:** Add an extra layer of server-side session validation on every request (DB lookup).

**How it works:**
- Skips public routes (login, register, debug endpoints)
- Extracts `userId` from JWT, `token` from `Authorization` header
- Checks DB session: returns `401` if not found, inactive, or expired
- Passes through to next middleware if valid

**Why it's disabled:** Performance concern (DB lookup on every request). The JWT itself already provides security. Session management is still used for force-logout purposes at login time.

---

## 11. API Project – SignalR Hub (Real-time Chat)

### `ChatHub` — `/hubs/chat`

Provides real-time, bidirectional messaging. Requires `[Authorize]` (JWT in connection handshake).

**Connection tracking:**
- Static `ConcurrentDictionary<int, HashSet<string>> _userConnections` maps `userId → set of SignalR connectionIds`
- Supports multiple connections per user (e.g., multiple browser tabs)

**Groups:**
- `user_{userId}` — for personal notifications to a specific user
- `chat_{chatId}` — for messages within a chat room

---

#### Hub Methods (called by the CLIENT):

**`OnConnectedAsync()`** — called automatically when client connects:
1. Adds connection to `_userConnections`
2. Joins `user_{userId}` group
3. Notifies all of the user's chats that this user is online

**`OnDisconnectedAsync()`** — called automatically when client disconnects:
1. Removes connection from `_userConnections`
2. If no more connections left for this user → notifies chats that user is offline
3. Leaves `user_{userId}` group

**`JoinChat(chatId)`** — client joins a specific chat room:
1. Verifies user is a participant (DB check)
2. Adds to `chat_{chatId}` group
3. Returns online status of other participants to the caller

**`LeaveChat(chatId)`** — client leaves a chat room (removes from group)

**`SendMessage(chatId, messageText)`** — send a message:
1. Verifies user is a participant
2. Saves message to DB (with `DeliveredAt = now`)
3. Broadcasts `ReceiveMessage` event to all in `chat_{chatId}`

**`TypingIndicator(chatId, isTyping)`** — broadcast typing state to others in the chat

**`MarkMessageAsRead(chatId, messageId)`**:
1. Sets `message.ReadAt = now` in DB
2. Sends `MessageRead` event to the message sender's personal group (`user_{senderId}`)

---

#### Events the CLIENT receives (SignalR sends to client):

| Event | When | Payload |
|-------|------|---------|
| `UserOnline` | User connects/disconnects | `{ userId, isOnline }` |
| `ParticipantPresence` | After JoinChat | `[{ userId, isOnline }]` |
| `ReceiveMessage` | New message sent | `{ messageId, chatId, senderUserId, senderName, messageText, createdAt, deliveredAt }` |
| `UserTyping` | Typing indicator | `{ userId, userName, isTyping }` |
| `MessageRead` | Message was read | `{ messageId, chatId, readBy, readAt }` |
| `Error` | Unauthorized / not participant | Error string |

---

## 12. MVC (Web) Project – Controllers

The MVC project is the browser-facing app. All controllers call the API via `HttpClient`.

### `BaseController`
Base for MVC controllers with utility methods:

```csharp
protected bool IsAdmin()
// Checks if Role claim == "0"

protected bool IsAuthenticated()
// Checks User.Identity?.IsAuthenticated

protected async Task<bool> ValidateSessionAsync()
// Reads "ApiAccessToken" cookie, calls ISessionService.IsSessionValidAsync()
// Returns false (and should redirect to login) if session is invalid

protected HttpClient CreateAuthorizedClient()
// Creates HttpClient "Api" — TokenHandler automatically injects the JWT cookie as Bearer header
```

---

### `AccountController` — `/Account`

Handles user authentication in the MVC layer.

| Action | Route | Method | Description |
|--------|-------|--------|-------------|
| `Login` | `/Account/Login` | GET | Show login form (redirects home if already logged in) |
| `Login` | `/Account/Login` | POST | Calls API, stores cookie, signs in with cookie auth |
| `Logout` | `/Account/Logout` | POST | Invalidates session, deletes cookie, signs out |
| `Register` | `/Account/Register` | GET | Show registration form |
| `Register` | `/Account/Register` | POST | Calls API to register, redirects to Login |
| `ForgotPassword` | `/Account/ForgotPassword` | GET/POST | Generates reset token, sends email |
| `ResetPassword` | `/Account/ResetPassword` | GET | Validates token, shows new password form |
| `ResetPassword` | `/Account/ResetPassword` | POST | Calls API reset-password endpoint |

**Session flow in MVC Login:**
1. Call API `/api/auth/login` → get JWT
2. Decode JWT to extract claims
3. Call `SessionService.InvalidateAllUserSessionsAsync()` then `CreateSessionAsync()` 
4. Set `ApiAccessToken` cookie (HttpOnly, Secure, SameSite=Lax, 1 hour)
5. Call `HttpContext.SignInAsync()` with cookie authentication + claims

---

### `HomeController` — `/`

| Action | Route | Description |
|--------|-------|-------------|
| `Index` | `/` | Landing page, fetches platform stats from API (`/api/users/stats`) |
| `AdminIndex` | `/Home/AdminIndex` | Admin dashboard |
| `Privacy` | `/Home/Privacy` | Privacy page |
| `Error` | `/Home/Error` | Error page |

---

### `UsersController` — `/Users`

| Action | Route | Description |
|--------|-------|-------------|
| `Index` | `/Users` | Admin: filterable user table; Regular user: people/network grid |
| `Profile` | `/Users/Profile/{id}` | View a user's full profile |
| `Edit` | `/Users/Edit/{id}` | Edit profile |
| `RequestEmployer` | `/Users/RequestEmployer` | Upload employer request form |

---

### `OpportunitiesController` — `/Opportunities`

| Action | Route | Description |
|--------|-------|-------------|
| `Index` | `/Opportunities` | List opportunities (with match scores for users) |
| `Details` | `/Opportunities/Details/{id}` | View opportunity details |
| `Create` | `/Opportunities/Create` | Create new opportunity (Employer) |
| `Edit` | `/Opportunities/Edit/{id}` | Edit opportunity |
| `Delete` | `/Opportunities/Delete/{id}` | Delete opportunity |

---

### `ApplicationsController` — `/Applications`

| Action | Route | Description |
|--------|-------|-------------|
| `Index` | `/Applications` | List own applications (or all for admin/employer) |
| `Apply` | `/Applications/Apply/{opportunityId}` | Application form |
| `Details` | `/Applications/Details/{id}` | Application detail with interview rounds |
| `UpdateStatus` | POST | Update application status |

---

### `ChatsController` — `/Chats`

| Action | Route | Description |
|--------|-------|-------------|
| `Index` | `/Chats` | List chats |
| `Details` | `/Chats/{id}` | Chat view (loads messages, connects to SignalR) |

---

### `CompaniesController` — `/Companies`

| Action | Route | Description |
|--------|-------|-------------|
| `Index` | `/Companies` | List companies |
| `Profile` | `/Companies/Profile/{id}` | Company profile with members and opportunities |
| `Create` | `/Companies/Create` | Create company form |

---

### `ConnectionsController` — `/Connections`

| Action | Route | Description |
|--------|-------|-------------|
| `Index` | `/Connections` | My connections + pending requests |
| `Request` | POST | Send connection request |
| `Accept` | POST `/{id}/Accept` | Accept a request |

---

### `SubscriptionsController` — `/Subscriptions`

Displays subscription plan info and upgrade options.

---

### `EmployerContactsController` — `/EmployerContacts`

Employer-specific views for managing candidate contacts and histories.

---

## 13. MVC (Web) Project – Services

### `TokenHandler` (DelegatingHandler)

Automatically adds the JWT token to all outgoing API requests.

```csharp
// Reads "ApiAccessToken" from the current request's cookies
// Attaches it as: Authorization: Bearer {token}
```

Registered as a message handler on the `"Api"` named `HttpClient`.

---

### `ApiClient` / `IApiClient`

Generic typed HTTP client for making API calls. Used for generic GET/POST operations.

---

### `UsersApiClient` / `IUsersApiClient`

Scoped service with specific methods for user-related API calls (search, profile fetch, etc.).

---

### `PasswordResetService` (Singleton)

Uses ASP.NET Core **Data Protection** to generate and validate password reset tokens.

**`GeneratePasswordResetToken(email)`**
- Creates protected string: `email|unixTimestamp`
- Returns encrypted token (URL-safe)

**`TryValidatePasswordResetToken(token, out email)`**
- Decrypts token, checks it's not older than 2 hours
- Returns `false` if expired or tampered

---

### `EmailSender` / `IEmailSender`

Sends emails for password reset. Registered as Singleton.

---

### `UserService` / `IUserService`

Scoped service for user-related operations in MVC (e.g., `FindByEmailAsync(email)` — queries the DB directly via `xcleratesystemslinks_SampleDBContext`).

---

### `LoggingHandler` (DelegatingHandler)

Logs outgoing HTTP requests and responses for debugging.

---

## 14. Subscription Plans & Limits

| Feature | Free (0) | Pro (1) | Enterprise (2) |
|---------|----------|---------|----------------|
| Job applications / month | 5 | Unlimited | Unlimited |
| Accepted connections | 50 | Unlimited | Unlimited |
| Chat | ✓ | ✓ | ✓ |

When a limit is exceeded, the API returns **HTTP 429** with:
```json
{
  "message": "...",
  "limitReached": true,
  "plan": "Free",
  "limit": 5
}
```

---

## 15. Match Score Algorithm

Implemented in `MatchScoreHelper` (static class).

### Purpose
Score how well a job opportunity matches a user's preferences (0–100%).

### Weights
- **Role match:** 70% weight
- **Location match:** 30% weight

### Role Score (`ComputeRoleScore`)
Uses **F1 score** (harmonic mean of precision and recall):
```
Precision = roles user prefers that the opportunity needs / opportunity required roles
Recall    = roles opportunity needs that user prefers / user preferred roles
F1        = 2 * precision * recall / (precision + recall)
RoleScore = F1 * 100
```

This prevents a user with many preferences from always scoring 100%.

### Location Score (`ComputeLocationScore`)
Uses structured `LocationId` and country codes:

| Scenario | Score |
|----------|-------|
| Same `LocationId` (exact match) | 100 |
| Same region, small country (PT, NL, BE, etc.) | 65 |
| Different region, small country | 35 |
| Same region, large country (US, etc.) | 50 |
| Different region, large country | 0 |
| Different country | 0 |
| Missing location data | -1 (not counted) |

### Final Weighted Score (`ComputeWeightedScore`)
```
if both role + location data available:
    score = roleScore * 0.7 + locationScore * 0.3

if only role data:
    score = roleScore

if only location data:
    score = locationScore

if no data:
    score = 0
```

---

## 16. Notification System

Notifications are created automatically by various controllers:

| Event | Trigger | Recipient |
|-------|---------|-----------|
| `ConnectionRequest` | User sends connection request | Addressee |
| `ConnectionAccepted` | User accepts connection | Requester |
| `JobApplied` | User applies to opportunity | Opportunity creator |
| `PostComment` | User comments on a post | Post author |
| `EmployerRequest` | User requests employer role | All admins |
| `OpportunityMatch` | New opportunity matches user perfectly | Matching users |

**Notification delivery:** REST API only (no push notifications). The MVC front-end polls `/api/notifications/my` to display the notification badge.

---

## 17. File Upload System

All uploads are stored on disk in `wwwroot/uploads/` under the API project.

| Folder | Usage |
|--------|-------|
| `profiles/` | User profile pictures |
| `banners/` | User banner images |
| `companies/` | Company logos |
| `employer-requests/` | Documents for employer role applications |

**Rules:**
- Max file size: 5 MB
- Images: `.jpg`, `.jpeg`, `.png`, `.gif`
- Documents: above + `.pdf`
- Files are named with a GUID to avoid collisions and prevent path traversal

**Swagger integration:** `FileUploadOperation` filter adds `multipart/form-data` support to Swagger UI for file upload endpoints marked with `[SwaggerFileUpload]`.

---

## 18. Key Data Flow Diagrams

### User Registration → Login → API Call

```
Browser → POST /Account/Register
              → MVC AccountController.Register()
              → HTTP POST /api/auth/register (API)
              → User saved to DB
              → Redirect to Login

Browser → POST /Account/Login
              → MVC AccountController.Login()
              → HTTP POST /api/auth/login (API)
                  → Verify password
                  → Invalidate old sessions
                  → Create JWT + Session record
                  → Return { token, expiresAt }
              → MVC: decode JWT, set "ApiAccessToken" cookie
              → MVC: cookie SignIn (ClaimsPrincipal)
              → Redirect to Home

Browser → GET /Opportunities
              → MVC OpportunitiesController.Index()
              → HttpClient "Api" (with TokenHandler)
                  → TokenHandler reads "ApiAccessToken" cookie
                  → Attaches Authorization: Bearer {jwt}
              → HTTP GET /api/opportunities (API)
                  → JWT validated by middleware
                  → Data returned
              → View rendered
```

### Real-time Chat Flow

```
User opens chat page
    → Browser connects to SignalR /hubs/chat (with JWT)
    → ChatHub.OnConnectedAsync() fires
        → User added to user_{userId} group
        → Chat groups notified of online status

User opens a chat room
    → Client calls hub.JoinChat(chatId)
    → Verified as participant
    → Added to chat_{chatId} group
    → Receives online status of other participants

User types message
    → Client calls hub.TypingIndicator(chatId, true)
    → Others in chat_{chatId} receive UserTyping event

User sends message
    → Client calls hub.SendMessage(chatId, text)
    → Message saved to DB
    → All in chat_{chatId} receive ReceiveMessage event

User reads a message
    → Client calls hub.MarkMessageAsRead(chatId, messageId)
    → ReadAt set in DB
    → Sender receives MessageRead event via user_{senderId} group
```

### Employer Approval Flow

```
User (Role=1) → POST /api/users/me/request-employer
    → Uploads document
    → User.Role set to 3 (Pending)
    → Notifications sent to all admins

Admin → GET /api/users/pending-employers
    → Sees pending requests

Admin → POST /api/users/{id}/approve-employer
    → User.Role set to 2 (Employer)
    → User.IsOpenToWork set to false
    → Pending company membership activated

OR

Admin → POST /api/users/{id}/reject-employer
    → User.Role set back to 1
    → Pending memberships removed
```

---

*This document was auto-generated from source code in the `APIPSI16_6_5.zip` archive.*
