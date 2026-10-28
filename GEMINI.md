# EduFlow: SaaS Platform Architecture Blueprint & Technical Specification

> **System**: EduFlow — Multi-Tenant Education Center Management SaaS  
> **Target Audience**: Commercial Education Centers, Language Academies, Prep Schools, Vocational Academies  
> **Architecture Paradigm**: Modular Monolith with Clean Architecture & Domain-Driven Design (DDD)  
> **Status**: Approved Architectural Baseline (Phase 0)

---

## 1. Product Architecture

### 1.1 Architectural Topology: Modular Monolith
EduFlow is architected as a high-performance **Modular Monolith** running on **Java 21** and **Spring Boot 3.3+**. It offers the team productivity and transactional simplicity of a single deployable unit while enforcing the strict physical and logical boundaries of microservices.

```mermaid
flowchart TD
    subgraph ClientLayer["Frontend Client (React 19 + TypeScript + Vite)"]
        UI["FSD Architecture: App -> Pages -> Widgets -> Features -> Entities -> Shared"]
        State["State Management: TanStack Query (Server) + Zustand (Client UI)"]
        I18n["Localization: i18next (UZ / RU / EN)"]
    end

    subgraph GatewayLayer["Edge & Security Layer (Spring Security 6.3)"]
        Filter["Tenant & Auth Filters (JWT RS256/HS512 + Redis Token Blacklist)"]
        ContextHolder["TenantContext & SecurityContext Injection"]
    end

    subgraph ModularMonolith["EduFlow Modular Monolith (Spring Boot 3.3 + Java 21)"]
        direction TB
        
        subgraph CoreContexts["Core Platform Contexts"]
            M_Identity["Identity & Access Module"]
            M_Org["Organization & Branch Module"]
            M_Audit["Audit & Activity Module"]
        end

        subgraph AcademicContexts["Academic Operations Contexts"]
            M_Student["Student Lifecycle Module"]
            M_Teacher["Teacher & Staff Module"]
            M_Course["Course & Curriculum Module"]
            M_Group["Group & Schedule Module"]
            M_Lesson["Lesson & Attendance Module"]
            M_Academic["Homework, Grades & Exam Module"]
        end

        subgraph FinancialContexts["Financial & Business Contexts"]
            M_Billing["Payment & Invoicing Module"]
            M_Expense["Branch Expense Module"]
        end

        subgraph EngagementContexts["Analytics & Engagement Contexts"]
            M_Notification["Notification Module (SMS, TG, Mail)"]
            M_Report["Analytics & Reporting Module"]
        end

        EventBus["Spring Application Events (In-Process Transactional Outbox)"]
    end

    subgraph InfrastructureLayer["Infrastructure & Storage Layer"]
        Postgres[(PostgreSQL 16 Multi-Tenant RLS)]
        Redis[(Redis 7: Session, Cache, Rate Limits)]
        RabbitMQ[RabbitMQ: Asynchronous Tasks, SMS Queues]
    end

    ClientLayer -->|HTTPS / REST v1| GatewayLayer
    GatewayLayer --> ContextHolder
    ContextHolder --> CoreContexts
    ContextHolder --> AcademicContexts
    ContextHolder --> FinancialContexts
    ContextHolder --> EngagementContexts

    M_Lesson -->|Domain Event| EventBus
    M_Billing -->|Domain Event| EventBus
    EventBus -->|Transactional Event Listener| M_Notification
    EventBus -->|Transactional Event Listener| M_Report
    EventBus -.->|Publish Task| RabbitMQ

    ModularMonolith --> Postgres
    ModularMonolith --> Redis
    RabbitMQ --> ModularMonolith
```

### 1.2 Module Layering & Clean Architecture
Each module in EduFlow strictly adheres to Clean Architecture layers with strict dependency inversion:

```
module-root (e.g., group/)
├── domain/                  # PURE JAVA: Aggregates, Entities, Value Objects, Domain Events, Domain Exceptions
│   ├── model/               # Group, GroupSchedule, GroupEnrollment, StudentLimit
│   ├── event/               # GroupActivatedEvent, GroupCompletedEvent
│   ├── exception/           # GroupCapacityExceededException, ScheduleConflictException
│   └── repository/          # GroupRepository (Interface ONLY, no Spring Data JPA)
├── application/             # APPLICATION LOGIC: Use cases, CQRS Command/Query Handlers, DTOs, Outbound Ports
│   ├── usecase/             # CreateGroupUseCase, EnrollStudentUseCase
│   ├── dto/                 # CreateGroupCommand, GroupDetailResponse, GroupSearchCriteria
│   └── port/                # Outbound SPIs (e.g., CourseQueryPort, BranchQueryPort)
├── infrastructure/          # ADAPTERS: Persistence, Spring Data JPA, External APIs, Mappers
│   ├── persistence/         # GroupJpaEntity, SpringDataGroupRepository, GroupRepositoryImpl
│   ├── mapper/              # GroupMapStructMapper
│   └── config/              # Module-specific Spring bean configuration
└── presentation/            # DELIVERY: REST Controllers, OpenAPI Annotations, Request Validators
    ├── controller/          # GroupController, GroupEnrollmentController
    ├── request/             # CreateGroupRequest, UpdateScheduleRequest
    └── response/            # GroupSummaryResponse, GroupScheduleResponse
```

### 1.3 Communication Rules Across Modules
1. **Zero Direct Entity Access**: Module `A` must NEVER inject JPA entities or repositories from Module `B`.
2. **Synchronous In-Process API**: If Module `A` needs data or operations from Module `B`, it calls a declared public interface (`*ModuleApi` or `*QueryPort`) exposed in `B`'s application layer.
3. **Asynchronous Decoupling**: Cross-module state transitions (e.g., `PaymentSucceeded` -> `UpdateStudentBalance` -> `SendReceiptSMS`) communicate through domain events using Spring's `@ApplicationModuleListener` / `TransactionalEventListener` and RabbitMQ.
4. **Architectural Guardrails**: Enforced via **ArchUnit** tests in CI to ensure that domain layers never import presentation or infrastructure packages, and cross-module boundaries are inviolate.

---

## 2. Domain Model & Invariants

```mermaid
classDiagram
    class Organization {
        +UUID id
        +String name
        +String slug
        +SubscriptionTier tier
        +String currency
        +ZoneId timezone
        +activate()
        +suspend()
    }

    class Branch {
        +UUID id
        +UUID organizationId
        +String name
        +String address
        +String phone
        +boolean isHeadquarters
    }

    class UserAccount {
        +UUID id
        +UUID organizationId
        +String email
        +String phone
        +String passwordHash
        +UserStatus status
        +Set~Role~ roles
    }

    class Student {
        +UUID id
        +UUID organizationId
        +UUID branchId
        +String firstName
        +String lastName
        +String phone
        +LocalDate birthDate
        +StudentStatus status
        +Money balance
        +credit(Money amount)
        +debit(Money amount)
    }

    class Teacher {
        +UUID id
        +UUID organizationId
        +String firstName
        +String lastName
        +String phone
        +Specialization specialization
        +SalaryScheme salaryScheme
    }

    class Course {
        +UUID id
        +UUID organizationId
        +String title
        +String code
        +int durationMonths
        +Money defaultPrice
        +CourseLevel level
    }

    class Group {
        +UUID id
        +UUID organizationId
        +UUID branchId
        +UUID courseId
        +UUID teacherId
        +String name
        +GroupStatus status
        +Schedule schedule
        +int maxCapacity
        +enrollStudent(StudentId)
        +complete()
    }

    class Lesson {
        +UUID id
        +UUID organizationId
        +UUID groupId
        +UUID teacherId
        +LocalDate date
        +TimeRange timeRange
        +String topic
        +LessonStatus status
        +start()
        +complete()
    }

    class Attendance {
        +UUID id
        +UUID lessonId
        +UUID studentId
        +AttendanceStatus status
        +String note
    }

    class Payment {
        +UUID id
        +UUID organizationId
        +UUID branchId
        +UUID studentId
        +Money amount
        +PaymentMethod method
        +PaymentStatus status
        +String receiptNumber
        +approve()
        +refund()
    }

    Organization "1" *-- "many" Branch
    Organization "1" *-- "many" UserAccount
    Branch "1" *-- "many" Student
    Organization "1" *-- "many" Teacher
    Organization "1" *-- "many" Course
    Branch "1" *-- "many" Group
    Course "1" *-- "many" Group
    Teacher "1" *-- "many" Group
    Group "1" *-- "many" Lesson
    Lesson "1" *-- "many" Attendance
    Student "1" *-- "many" Attendance
    Student "1" *-- "many" Payment
```

### 2.1 Domain Invariants & Business Rules
- **Tenant Isolation**: Every aggregate root must carry an immutable `organizationId`. Any state transition must verify that target child entities belong to the same tenant.
- **Group Capacity**: A group cannot exceed its configured `maxCapacity` without an explicit managerial override privilege (`group:override_capacity`).
- **Schedule Clashing**: A teacher or room cannot be booked for overlapping time windows within the same branch.
- **Attendance Locking**: Completed lessons lock attendance records from modification after 48 hours unless unlocked by a Branch Manager or Owner.
- **Financial Auditability**: Payments are immutable records. Once completed, they cannot be updated or deleted; corrections require explicit `Refund` or `CreditAdjustment` transactions.
- **Student Balance Invariant**: A student's billing balance is an event-sourced or ledger-backed projection calculated from approved payments, discounts, and lesson/group tuition charges.

---

## 3. Bounded Contexts & Context Mapping

```mermaid
graph TD
    subgraph IdentityContext["Identity & Access Context"]
        Auth[Auth / JWT / MFA / RBAC]
    end

    subgraph OrgContext["Organization & Branch Context"]
        OrgHierarchy[Tenant / Subscription / Branch Management]
    end

    subgraph AcademicContext["Academic Core Context"]
        CourseMgmt[Courses / Levels / Syllabi]
        GroupSchedule[Groups / Room Allocation / Schedules]
        LessonLifecycle[Lessons / Topics / Journals]
    end

    subgraph PeopleContext["Student & Teacher Context"]
        StudentProfiles[Student Lifecycle & Enrollment]
        TeacherProfiles[Teacher Contracts & Specialization]
    end

    subgraph AssessmentContext["Assessment & Learning Context"]
        HW[Homework & Submissions]
        Grading[Grades & Criteria]
        Exams[Exams & Test Engine]
    end

    subgraph FinancialContext["Billing & Financial Context"]
        Billing[Tuition Invoicing & Receipts]
        Expense[Branch Operating Expenses]
    end

    subgraph NotificationContext["Notification & Dispatch Context"]
        SMS[SMS Gateway: PlayMobile/Eskiz]
        TG[Telegram Bot Webhooks]
        Email[Email Dispatcher]
    end

    subgraph AnalyticsContext["Analytics & Reporting Context"]
        BI[Attendance Rates / Churn / Revenue Projections]
    end

    OrgContext -->|Upstream / Shared Kernel| IdentityContext
    OrgContext -->|Upstream| AcademicContext
    OrgContext -->|Upstream| PeopleContext
    PeopleContext -->|Customer / Supplier| AcademicContext
    AcademicContext -->|Publishes Events| AssessmentContext
    AcademicContext -->|Publishes Lessons| FinancialContext
    FinancialContext -->|Publishes Transaction Events| NotificationContext
    AcademicContext -->|Publishes Event Stream| NotificationContext
    AcademicContext -.->|Read Model Projection| AnalyticsContext
    FinancialContext -.->|Read Model Projection| AnalyticsContext
```

### 3.1 Upstream/Downstream Relationships
- **Customer/Supplier (People -> Academic)**: Group scheduling depends on active Student and Teacher profiles.
- **Published Language / Domain Events (Academic -> Billing)**: Attendance marking or enrollment completion triggers billing events processed by the Billing module asynchronously.
- **Anti-Corruption Layer (ACL) for External Gateways**: The Notification module encapsulates vendor-specific SMS APIs (e.g. Eskiz.uz, PlayMobile) behind a clean `SmsNotificationPort`.

---

## 4. Database ERD Concept & Multi-Tenancy Strategy

### 4.1 Multi-Tenancy Model: Partitioned Row-Level Isolation
- **Approach**: Shared Database, Shared Schema with strict Row-Level Security (RLS) and Application-Level Filters (`@TenantId` / Hibernate Filters).
- **Primary Keys**: **UUIDv7** across all tables (time-ordered, index-friendly, secure against enumeration attacks).
- **Tenant Key**: `organization_id UUID NOT NULL` indexed on all tenant-scoped tables.
- **Branch Scope**: `branch_id UUID NULL` on branch-specific records (allows multi-branch or org-wide entities).

### 4.2 Standard Audit Columns (`BaseTenantEntity`)
Every database table inherits:
```sql
id              UUID PRIMARY KEY,
organization_id UUID NOT NULL,
created_at      TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at      TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP,
created_by      UUID,
updated_by      UUID,
version         BIGINT NOT NULL DEFAULT 0,
is_deleted      BOOLEAN NOT NULL DEFAULT FALSE,
deleted_at      TIMESTAMPTZ
```

### 4.3 Core Relational Schema

```
+----------------------------------------------------------------------------------------------------+
|                                      ORGANIZATIONS & BRANCHES                                      |
+----------------------------------------------------------------------------------------------------+
| organizations                                  branches                                            |
|   id: UUID (PK, v7)                              id: UUID (PK, v7)                                 |
|   name: VARCHAR(255)                             organization_id: UUID (FK -> organizations.id)    |
|   slug: VARCHAR(100) UNIQUE                      name: VARCHAR(255)                                |
|   status: VARCHAR(32)                            code: VARCHAR(50)                                 |
|   tier: VARCHAR(32)                              address: TEXT                                     |
|   currency: VARCHAR(3) (e.g., 'UZS', 'USD')      phone: VARCHAR(32)                                |
|   timezone: VARCHAR(64)                          is_headquarters: BOOLEAN                          |
|   settings: JSONB                                is_active: BOOLEAN                                |
|   created_at, updated_at, version                created_at, updated_at, version                   |
+----------------------------------------------------------------------------------------------------+
                                                   |
+--------------------------------------------------+-------------------------------------------------+
|                                            PEOPLE & IDENTITY                                       |
+----------------------------------------------------------------------------------------------------+
| users                                          students                 teachers                   |
|   id: UUID (PK)                                  id: UUID (PK)            id: UUID (PK)            |
|   organization_id: UUID (FK)                     organization_id: UUID    organization_id: UUID    |
|   branch_id: UUID (FK, NULLable)                 branch_id: UUID          user_id: UUID (FK)       |
|   email: VARCHAR(255)                            user_id: UUID (FK, NULL) first_name: VARCHAR(100) |
|   phone: VARCHAR(32)                             first_name: VARCHAR(100) last_name: VARCHAR(100)  |
|   password_hash: VARCHAR(255)                    last_name: VARCHAR(100)  phone: VARCHAR(32)       |
|   status: VARCHAR(32)                            phone: VARCHAR(32)       specialization: VARCHAR  |
|   system_role: VARCHAR(32)                       parent_name: VARCHAR     hourly_rate: NUMERIC     |
|   created_at, updated_at                         parent_phone: VARCHAR    created_at, updated_at   |
|                                                  status: VARCHAR(32)                               |
|                                                  balance: NUMERIC(15,2)                            |
+----------------------------------------------------------------------------------------------------+
                                                   |
+--------------------------------------------------+-------------------------------------------------+
|                                           ACADEMIC STRUCTURE                                       |
+----------------------------------------------------------------------------------------------------+
| courses                    groups                          group_students (Enrollments)            |
|   id: UUID (PK)              id: UUID (PK)                   id: UUID (PK)                         |
|   organization_id: UUID      organization_id: UUID           group_id: UUID (FK -> groups.id)      |
|   title: VARCHAR(255)        branch_id: UUID (FK)            student_id: UUID (FK -> students.id)  |
|   code: VARCHAR(50)          course_id: UUID (FK)            enrolled_at: TIMESTAMPTZ              |
|   default_price: NUMERIC     teacher_id: UUID (FK)           status: VARCHAR(32) (ACTIVE/DROPPED)  |
|   lessons_count: INT         name: VARCHAR(100)              custom_discount_percent: INT          |
|   is_active: BOOLEAN         status: VARCHAR(32)                                                   |
|                              max_capacity: INT             lessons                                 |
|                              room_number: VARCHAR(50)        id: UUID (PK)                         |
|                              days_of_week: INT[] (1..7)      group_id: UUID (FK -> groups.id)      |
|                              start_time: TIME                teacher_id: UUID (FK)                 |
|                              end_time: TIME                  date: DATE                            |
|                              start_date: DATE                status: VARCHAR(32)                   |
|                              end_date: DATE                  topic: VARCHAR(255)                   |
+----------------------------------------------------------------------------------------------------+
                                                               |
+--------------------------------------------------------------+-------------------------------------+
|                                          ATTENDANCE & ASSESSMENTS                                  |
+----------------------------------------------------------------------------------------------------+
| attendances                                    homeworks                 homework_submissions      |
|   id: UUID (PK)                                  id: UUID (PK)             id: UUID (PK)           |
|   lesson_id: UUID (FK -> lessons.id)             lesson_id: UUID (FK)      homework_id: UUID (FK)  |
|   student_id: UUID (FK -> students.id)           title: VARCHAR(255)       student_id: UUID (FK)   |
|   status: VARCHAR(20) (PRESENT, ABSENT, LATE)    description: TEXT         content: TEXT           |
|   note: VARCHAR(255)                             due_date: TIMESTAMPTZ     score: INT              |
|   marked_at: TIMESTAMPTZ                         max_score: INT            graded_at: TIMESTAMPTZ  |
+----------------------------------------------------------------------------------------------------+
                                                   |
+--------------------------------------------------+-------------------------------------------------+
|                                            FINANCE & AUDIT                                         |
+----------------------------------------------------------------------------------------------------+
| payments                                       expenses                  audit_logs                |
|   id: UUID (PK)                                  id: UUID (PK)             id: UUID (PK)           |
|   organization_id: UUID                          organization_id: UUID     organization_id: UUID   |
|   branch_id: UUID (FK)                           branch_id: UUID (FK)      user_id: UUID (FK)      |
|   student_id: UUID (FK)                          category: VARCHAR(50)     action: VARCHAR(100)    |
|   amount: NUMERIC(15,2)                          amount: NUMERIC(15,2)     resource_type: VARCHAR  |
|   currency: VARCHAR(3)                           currency: VARCHAR(3)      resource_id: VARCHAR    |
|   method: VARCHAR(32) (CASH, CARD, PAYME, CLICK) description: TEXT         diff: JSONB             |
|   status: VARCHAR(32)                            paid_at: DATE             ip_address: VARCHAR(45) |
|   idempotency_key: VARCHAR(128) UNIQUE           approved_by: UUID         created_at: TIMESTAMPTZ |
+----------------------------------------------------------------------------------------------------+
```

### 4.4 Liquibase Migration Structure
Migrations use master XML changelogs invoking structured SQL or XML sub-changesets:
```
src/main/resources/db/changelog/
├── db.changelog-master.xml
├── v1.0.0/
│   ├── 01-schema-identity.xml
│   ├── 02-schema-organization.xml
│   ├── 03-schema-academic.xml
│   ├── 04-schema-attendance.xml
│   ├── 05-schema-financial.xml
│   └── 06-schema-indexes-and-rls.sql
```

---

## 5. API Strategy & Standards

### 5.1 URL Conventions & Tenant Resolution
Tenant identification is passed via header: `X-Organization-Id: <UUID>` and validated against the authenticated user's JWT authorization claims.

```
POST   /api/v1/auth/login
POST   /api/v1/auth/refresh
GET    /api/v1/organizations/me
GET    /api/v1/branches
GET    /api/v1/students?branchId={id}&status=ACTIVE&search=Ali&page=0&size=20&sort=lastName,asc
POST   /api/v1/students
GET    /api/v1/students/{id}
POST   /api/v1/groups/{id}/enrollments
POST   /api/v1/lessons/{id}/attendance
POST   /api/v1/payments (Headers: Idempotency-Key)
```

### 5.2 Uniform Response Envelope & Pagination
```json
{
  "success": true,
  "data": {
    "items": [ ... ],
    "page": 0,
    "size": 20,
    "totalElements": 142,
    "totalPages": 8,
    "hasNext": true,
    "hasPrevious": false
  },
  "timestamp": "2026-10-28T16:55:00Z"
}
```

### 5.3 Error Handling: RFC 9457 Problem Details
All errors return `application/problem+json`:
```json
{
  "type": "https://api.eduflow.uz/errors/schedule-conflict",
  "title": "Schedule Conflict Encountered",
  "status": 409,
  "detail": "Teacher 'Aziz Rahimov' is already scheduled in Group 'SAT-02' between 14:00 and 15:30 on Mondays.",
  "instance": "/api/v1/groups/7f18b5b9-e137-47b2-8509-646ef5de3201/schedule",
  "errorCode": "SCHEDULE_TEACHER_DOUBLE_BOOKED",
  "timestamp": "2026-10-28T16:55:00Z",
  "invalidParams": [
    {
      "field": "startTime",
      "reason": "Overlaps with existing lesson"
    }
  ]
}
```

---

## 6. Authentication Strategy

### 6.1 Token Lifecycle & Storage
- **Access Token (JWT)**:
  - Expiration: **15 minutes**
  - Signer: RS256 or HMAC-SHA512 with high-entropy 512-bit secret.
  - Claims: `sub` (userId), `orgId` (active tenant), `branchId` (optional), `roles` (array), `perms` (compressed bitmask or string array), `type` ("access").
- **Refresh Token**:
  - Expiration: **14 days**
  - Stored in **Redis** with key pattern: `auth:refresh:{tokenFamilyId}:{userId}`.
  - **Automatic Breach Detection (Token Rotation)**: If a previously rotated refresh token is reused, the entire token family is immediately revoked, forcing re-authentication across all active user sessions.
- **Password Security**: **Argon2id** (`m=65536, t=3, p=1`) or **BCrypt** (cost factor 12).
- **Brute-Force Protection**: Redis-backed sliding window rate limiter (Bucket4j): max 5 failed attempts per phone/email per 15 minutes before temporary lockout.

```mermaid
sequenceDiagram
    autonumber
    actor Client as React Client
    participant Gateway as Security Filter
    participant AuthMod as Identity Module
    participant Redis as Redis Token Store
    participant DB as Postgres DB

    Client->>Gateway: POST /api/v1/auth/login {phone, password}
    Gateway->>AuthMod: Authenticate
    AuthMod->>DB: Fetch user by phone & verify Argon2id hash
    AuthMod->>Redis: Generate RefreshToken & save with TokenFamilyId (TTL 14d)
    AuthMod-->>Client: Return {accessToken (15m), refreshToken (14d), userProfile}
    
    Note over Client, Gateway: Subsequent API Requests with Header "Authorization: Bearer <token>"
    Client->>Gateway: GET /api/v1/students (Bearer token)
    Gateway->>Gateway: Validate signature & expiry, extract OrgContext
    Gateway-->>Client: 200 OK Response
```

---

## 7. Authorization Strategy

### 7.1 Multi-Tenant RBAC + PBAC (Permission-Based Access Control)
Roles are collections of fine-grained permissions. Controllers NEVER check `@PreAuthorize("hasRole('ADMIN')")`; they enforce granular permissions:

| System Role | Scope | Key Permissions |
| :--- | :--- | :--- |
| **SUPER_ADMIN** | Global Platform | `platform:org:create`, `platform:metrics:view` |
| **ORG_OWNER** | Organization | `org:manage`, `branch:create`, `finance:full`, `user:manage` |
| **BRANCH_MANAGER**| Assigned Branch | `student:write`, `teacher:assign`, `group:manage`, `expense:create` |
| **REGISTRAR** | Assigned Branch | `student:create`, `student:enroll`, `payment:receive` |
| **TEACHER** | Assigned Groups | `lesson:start`, `attendance:mark`, `homework:grade` |
| **STUDENT** | Self Profile | `my:schedule:view`, `my:grades:view`, `my:balance:view` |
| **PARENT** | Linked Children | `child:attendance:view`, `child:grades:view`, `child:payment:pay` |

### 7.2 Tenant Context & Enforcement Pipeline
1. `TenantContextFilter` extracts the active Organization ID from JWT and `X-Organization-Id`.
2. Verifies user membership in the requested organization.
3. Sets `TenantContextHolder.setContext(new TenantContext(orgId, branchId, userId, permissions))`.
4. Executes database query under tenant-scoped JPA filter or PostgreSQL session variable:
   ```sql
   SET LOCAL app.current_organization_id = 'c4b8b6e2-...';
   ```

---

## 8. Frontend Architecture (Feature-Sliced Design)

### 8.1 Directory Structure
```
frontend/src/
├── app/                        # Application initialization, providers, routing, root styles
│   ├── providers/              # QueryClientProvider, ThemeProvider, I18nProvider, AuthProvider
│   ├── routes/                 # React Router v6 definitions with permission guards
│   └── styles/                 # Tailwind CSS globals, fonts, CSS variables
├── pages/                      # Page components (thin composition layer)
│   ├── dashboard/              # Dashboard overview page
│   ├── students/               # Student list, detail, and enrollment pages
│   ├── schedule/               # Interactive calendar and timetable page
│   ├── attendance/             # Daily attendance marking matrix
│   └── finance/                # Payments, invoices, and expense ledger
├── widgets/                    # Composite business blocks reused across pages
│   ├── sidebar/                # Collapsible tenant-aware navigation
│   ├── header/                 # Branch selector, search command palette, user dropdown
│   ├── attendance-sheet/       # Interactive table for grading/attendance
│   └── revenue-chart/          # Financial chart widget with date range selector
├── features/                   # User interactions that yield business value
│   ├── auth-by-phone/          # Login form, OTP verification, password reset
│   ├── enroll-student/         # Modal wizard to enroll student into a group
│   ├── mark-attendance/        # Instant toggle buttons (Present/Late/Absent)
│   ├── collect-payment/        # Payment drawer with receipt printer
│   └── switch-branch/          # Header dropdown to change active branch
├── entities/                   # Domain entities with types, API hooks, and UI cards
│   ├── student/                # types, api/useStudent, ui/StudentAvatar, ui/StudentBadge
│   ├── teacher/                # types, api/useTeacher, ui/TeacherCard
│   ├── course/                 # types, api/useCourse, ui/CourseSelect
│   ├── group/                  # types, api/useGroup, ui/GroupStatusBadge
│   └── payment/                # types, api/usePayment, ui/PaymentStatusChip
└── shared/                     # Reusable utilities, design system primitives, api client
    ├── api/                    # Axios instance with interceptors, token refresh queue
    ├── ui/                     # shadcn/ui components (Button, Dialog, Table, Input, etc.)
    ├── lib/                    # Utilities (cn, date-fns formatters, number/currency formatters)
    ├── hooks/                  # useDebounce, useMediaQuery, useKeyboardShortcut
    └── config/                 # i18n instance, env constants
```

### 8.2 State Separation Principle
- **Server State**: Managed strictly through **TanStack Query (v5)**.
  - Standardized Query Keys: `['students', 'list', { branchId, page, search }]`
  - Optimistic mutations for high-frequency actions (e.g. marking attendance).
- **Client UI State**: Managed via lightweight **Zustand** stores:
  - `useAuthStore`: current user, active tokens, assigned roles.
  - `useTenantStore`: active organization, active branch selection.
  - `useThemeStore`: dark / light / system preference.
  - `useSidebarStore`: expanded / collapsed state.

---

## 9. Design System & UX Standards

### 9.1 Visual Philosophy: Calm, Premium Enterprise SaaS
- **Neutral Foundation**: Tailwind `zinc` palette with subtle 1px borders (`border-zinc-200` light, `border-zinc-800` dark).
- **Accent Color**: Deep Indigo (`#4338CA` / `indigo-700`) or Emerald (`#059669` / `emerald-600`) for high-trust actions.
- **Typography**: `Inter` or `Plus Jakarta Sans` with high legibility at 12px–14px data-dense tables.
- **Zero Gimmicks**: No neon gradients, no exaggerated glassmorphism, no playful oversized rounded corners. Clean, compact, highly legible.

### 9.2 Complete Theme Token Mapping
| Token | Light Mode | Dark Mode | Usage |
| :--- | :--- | :--- | :--- |
| `--background` | `0 0% 100%` (`#FFFFFF`) | `240 10% 4%` (`#09090B`) | Main app canvas |
| `--card` | `0 0% 100%` (`#FFFFFF`) | `240 10% 6%` (`#111113`) | Surface container |
| `--popover` | `0 0% 100%` (`#FFFFFF`) | `240 10% 6%` (`#111113`) | Dropdowns, menus |
| `--primary` | `238 84% 59%` (`#4F46E5`) | `238 84% 67%` (`#6366F1`) | Primary CTAs, active items |
| `--muted` | `240 5% 96%` (`#F4F4F5`) | `240 4% 16%` (`#27272A`) | Inactive tabs, table headers |
| `--border` | `240 6% 90%` (`#E4E4E7`) | `240 4% 18%` (`#27272A`) | Data borders, dividers |
| `--success` | `158 64% 45%` (`#10B981`) | `158 64% 45%` (`#10B981`) | Paid, Present, Passed |
| `--danger` | `0 84% 60%` (`#EF4444`) | `0 84% 60%` (`#EF4444`) | Absent, Overdue, Cancelled |

### 9.3 UX States Checklist
1. **Loading State**: Customized skeleton placeholders matching table row heights.
2. **Empty State**: Meaningful vector illustration or icon, concise title, and an immediate primary CTA button (e.g., *"No students enrolled yet. Add your first student"*).
3. **Error State**: Non-destructive alert banner with a clear retry trigger.
4. **Form Validation**: Real-time feedback via React Hook Form + Zod with translated error strings.

### 9.4 Trilingual Localization Matrix (i18next)
All UI keys are segregated by feature namespace:
```json
// locales/uz/attendance.json
{
  "title": "Davomat jurnali",
  "markAllPresent": "Barchasini keldi deb belgilash",
  "status": {
    "present": "Keldi",
    "absent": "Kelmadi",
    "late": "Kechikdi",
    "excused": "Sababli"
  }
}
```

---

## 10. Multi-Phase Implementation Roadmap

```mermaid
gantt
    title EduFlow Engineering Roadmap
    dateFormat  YYYY-MM-DD
    section Phase 0: Foundations
    Architecture & Workspace Scaffolding        :p0_1, 2026-11-01, 3d
    Base Modules, Security, Liquibase Setup    :p0_2, after p0_1, 4d
    section Phase 1: Core Multi-Tenancy & Auth
    Organization, Branch & User Management     :p1_1, after p0_2, 5d
    JWT Auth, Refresh Rotation, Role Engine    :p1_2, after p1_1, 5d
    Frontend Shell, Layout, Auth Flow, i18n    :p1_3, after p1_2, 5d
    section Phase 2: Academic Core
    Courses, Levels & Syllabi                  :p2_1, after p1_3, 4d
    Teachers & Staff Directory                 :p2_2, after p2_1, 4d
    Groups, TimeSlots & Student Enrollment     :p2_3, after p2_2, 6d
    section Phase 3: Operations & Journal
    Lesson Generation & Timetable Calendar     :p3_1, after p2_3, 5d
    Attendance Marking Matrix                  :p3_2, after p3_1, 5d
    Homework, Assignments & Grading Journal    :p3_3, after p3_2, 5d
    section Phase 4: Financials & Billing
    Student Balances & Pricing Plans           :p4_1, after p3_3, 5d
    Payments, Receipts & Invoicing             :p4_2, after p4_1, 5d
    Branch Operating Expense Register          :p4_3, after p4_2, 4d
    section Phase 5: Notifications & Analytics
    SMS Gateway (Uzbekistan) & Telegram Bot    :p5_1, after p4_3, 5d
    Executive Dashboard & Revenue Reporting    :p5_2, after p5_1, 5d
    section Phase 6: Production Hardening
    Audit Logs, E2E Testing & Performance Tuning:p6_1, after p5_2, 6d
```

### Milestone Deliverables Summary
- **Phase 0 (Target Next)**: 
  - Gradle multi-project or clean module setup (`backend/`), Vite React TypeScript Tailwind app (`frontend/`).
  - Liquibase master changelog and initial schema migrations.
  - Base entity models, error handlers, and ArchUnit architecture verification tests.
- **Phase 1**:
  - Identity, Organization, Branch domain models and APIs.
  - Spring Security JWT + Redis refresh token rotation.
  - Frontend login, tenant/branch selector, responsive shell, dark/light theme, i18n switcher.
- **Phase 2**:
  - Course, Teacher, Student, and Group domain models with strict invariants.
  - Group capacity enforcement and student enrollment workflows.
- **Phase 3**:
  - Timetable, Lesson generator, High-speed Attendance Matrix with keyboard shortcuts.
- **Phase 4**:
  - Billing, Cashier desk, Payme/Click integration ready, Expense logging.
- **Phase 5**:
  - SMS notification triggers on attendance/payment, Executive charts and dashboards.
- **Phase 6**:
  - Audit logging, Testcontainers integration testing, Docker Compose for production deployment.
