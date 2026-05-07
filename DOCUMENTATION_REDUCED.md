# CallCenterApp — System Technical Documentation

> **Version:** 1.0 · **Framework:** ASP.NET Core MVC (.NET 10) · **Database:** SQL Server  
> **Repository:** https://github.com/KsAmJ/CallCenterApp

---

## Table of Contents

1. [Project Overview](#1-project-overview)
2. [Technology Stack](#2-technology-stack)
3. [Architecture Overview](#3-architecture-overview)
4. [Security & Authentication Model](#4-security--authentication-model)
5. [Process Flow Diagrams](#5-process-flow-diagrams)
   - 5.1 [Application Startup Flow](#51-application-startup-flow)
   - 5.2 [Authentication & Role Routing Flow](#52-authentication--role-routing-flow)
   - 5.3 [AMI Connection & Event Lifecycle](#53-ami-connection--event-lifecycle)
   - 5.4 [Queue Log Import Flow](#54-queue-log-import-flow)
   - 5.5 [Scheduled Report Flow](#55-scheduled-report-flow)
   - 5.6 [Agent Status Update Flow](#56-agent-status-update-flow)
   - 5.7 [Call Registration Flow](#57-call-registration-flow)
6. [Request Lifecycle Flowchart](#6-request-lifecycle-flowchart)

---

## 1. Project Overview

**CallCenterApp** is a full-featured call center management platform built on ASP.NET Core MVC (.NET 10). It provides three distinct role-based portals — **Admin**, **Supervisor**, and **Agent** — backed by a SQL Server database and a real-time integration layer with an **Asterisk PBX** server via the Asterisk Manager Interface (AMI) protocol.

### Key Capabilities

| Capability | Description |
|---|---|
| Role-Based Portals | Separate UI experience per role (Admin / Supervisor / Agent) |
| PBX Integration | Real-time AMI TCP connection to Asterisk for live queue and agent control |
| Call Logging | Manual and AMI-driven call record management |
| Queue Management | Create, configure, and monitor Asterisk call queues |
| IVR Management | Define IVR menus with JSON-configured options |
| Routing Rules | Priority-based inbound call routing configuration |
| Subscription & Billing | Per-company subscription plans with billing metadata |
| Scheduled Reporting | Automated report generation (Daily / Weekly / Monthly) |
| Queue Log Import | Continuous file-based Asterisk `queue_log` ingestion |
| Audit Trail | Full user-action audit log with IP tracking |
| Export | Excel (ClosedXML) and PDF (QuestPDF) report exports |

---

## 2. Technology Stack

| Layer | Technology |
|---|---|
| Runtime | .NET 10 |
| Web Framework | ASP.NET Core MVC |
| Authentication | ASP.NET Core Identity |
| ORM | Entity Framework Core 10 |
| Database | Microsoft SQL Server |
| UI Styling | Bootstrap 5 |
| Excel Export | ClosedXML |
| PDF Export | QuestPDF (Community License) |
| PBX Integration | Asterisk Manager Interface (AMI) over raw TCP |
| Background Jobs | `IHostedService` / `BackgroundService` |
| Async Channels | `System.Threading.Channels` |

---

## 3. Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                          Browser / Client                        │
└────────────────────────────┬─────────────────────────────────────┘
                             │  HTTPS
┌────────────────────────────▼─────────────────────────────────────┐
│                    ASP.NET Core MVC (.NET 10)                    │
│                                                                  │
│  ┌──────────────┐  ┌─────────────────┐  ┌────────────────────┐  │
│  │  Controllers │  │    ViewModels   │  │      Views         │  │
│  │  (12 total)  │  │   (10 total)    │  │   (Razor .cshtml)  │  │
│  └──────┬───────┘  └────────┬────────┘  └────────────────────┘  │
│         │                   │                                     │
│  ┌──────▼───────────────────▼──────────────────────────────────┐ │
│  │                     Service Layer                           │ │
│  │  ┌──────────────┐  ┌───────────────┐  ┌─────────────────┐  │ │
│  │  │  AmiService  │  │ QueueLogSvc   │  │  AuditLogSvc    │  │ │
│  │  │ (IHostedSvc) │  │  (Scoped)     │  │   (Scoped)      │  │ │
│  │  └──────┬───────┘  └───────┬───────┘  └────────┬────────┘  │ │
│  │         │                  │                    │           │ │
│  │  ┌──────▼──────────────────▼────────────────────▼────────┐  │ │
│  │  │             ReportSchedulerService (BackgroundSvc)    │  │ │
│  │  └───────────────────────────────────────────────────────┘  │ │
│  └──────────────────────────────┬──────────────────────────────┘ │
│                                 │                                 │
│  ┌──────────────────────────────▼──────────────────────────────┐ │
│  │             Data Layer  (Entity Framework Core)             │ │
│  │                    CallCenterDbContext                       │ │
│  └──────────────────────────────┬──────────────────────────────┘ │
└─────────────────────────────────┼────────────────────────────────┘
                                  │  ADO.NET / TDS
┌─────────────────────────────────▼────────────────────────────────┐
│                     SQL Server  (CallCenterDb)                   │
│         Tables · Views · Stored Procedures · Indexes             │
└──────────────────────────────────────────────────────────────────┘

                    ┌─────────────────────────┐
                    │      Asterisk PBX        │
                    │  manager.conf (TCP 5038) │
                    └────────────┬────────────┘
                                 │  AMI Protocol (TCP)
                    ┌────────────▼────────────┐
                    │   AmiService (Singleton) │
                    │   - Persistent TCP conn  │
                    │   - Async reader loop    │
                    │   - Channel<AmiEvent>    │
                    └─────────────────────────┘
```

---

## 4. Security & Authentication Model

### Identity Setup

- Powered by **ASP.NET Core Identity** with a custom `ApplicationUser` class extending `IdentityUser`.
- Password policy: minimum 8 characters, requires digit, uppercase, and non-alphanumeric character.
- Unique email enforced per user.
- Default admin user seeded at startup: `admin@callcenter.local` / `Admin#12345`.

### Roles

| Role | Constant | Default Portal |
|---|---|---|
| `Admin` | `Roles.Admin` | `/AdminPanel` |
| `Supervisor` | `Roles.Supervisor` | `/SupervisorPanel` |
| `Agent` | `Roles.Agent` | `/AgentPanel` |

### Global Authorization Policy

All routes require authentication by default via a global `AuthorizeFilter`. Public routes (`/Account/Login`, `/Account/AccessDenied`) are decorated with `[AllowAnonymous]`.

### Controller-Level Authorization

| Controller | Allowed Roles |
|---|---|
| `AccountController` | All (login is `[AllowAnonymous]`), user management = Admin only |
| `AdminPanelController` | Admin |
| `SupervisorPanelController` | Admin, Supervisor |
| `AgentPanelController` | Agent |
| `AmiMonitorController` | Admin, Supervisor |
| `ReportsController` | Admin, Supervisor |
| `DashboardController` | Admin, Supervisor, Agent |
| `AuditLogsController` | Admin |
| `CallsController` | Admin, Supervisor |
| `CompaniesController` | Admin, Supervisor |
| `CustomersController` | Admin, Supervisor |

### CSRF Protection

All `[HttpPost]` actions are decorated with `[ValidateAntiForgeryToken]`. All forms include `@Html.AntiForgeryToken()`.

### Audit Trail

Every significant user action (login, logout, role change, data export) is recorded in the `AuditLogs` table via `IAuditLogService`, capturing: `Action`, `UserId`, `UserEmail`, `EntityName`, `EntityId`, `IpAddress`, `Details`, `CreatedAtUtc`.

---

## 5. Process Flow Diagrams

### 5.1 Application Startup Flow

```mermaid
flowchart TD
    A([Application Start]) --> B[Build WebApplication]
    B --> C[Register Services\nDI Container]
    C --> D{AmiHost configured?}
    D -- Yes --> E[AmiService.StartAsync\nConnect to Asterisk AMI]
    D -- No --> F[AmiService starts\nin disconnected mode]
    E --> G[ReportSchedulerService\nstarts BackgroundService]
    F --> G
    G --> H[IdentitySeeder.SeedAsync\nCreate roles + admin user]
    H --> I[Configure HTTP Pipeline\nAuth · HTTPS · Routing]
    I --> J([Application Ready\nListening for requests])
```

---

### 5.2 Authentication & Role Routing Flow

```mermaid
flowchart TD
    A([User visits any URL]) --> B{Authenticated?}
    B -- No --> C[Redirect to /Account/Login]
    C --> D[User enters Email + Password]
    D --> E[POST /Account/Login]
    E --> F[SignInManager.PasswordSignInAsync]
    F --> G{Result?}
    G -- Failed --> H[Log LOGIN_FAILED\nShow error message]
    H --> D
    G -- Succeeded --> I[Log LOGIN_SUCCESS]
    I --> J{ReturnUrl present\nand local?}
    J -- Yes --> K[Redirect to ReturnUrl]
    J -- No --> L[GetRolesAsync for user]
    L --> M{Role?}
    M -- Admin --> N["/AdminPanel/Index"]
    M -- Supervisor --> O["/SupervisorPanel/Index"]
    M -- Agent --> P["/AgentPanel/Index"]
    M -- None --> Q["/Dashboard/Index"]
    B -- Yes --> R{Access authorized\nfor route?}
    R -- Yes --> S[Process Request\nReturn View]
    R -- No --> T["/Account/AccessDenied"]
```

---

### 5.3 AMI Connection & Event Lifecycle

```mermaid
sequenceDiagram
    participant App as ASP.NET App
    participant AMI as AmiService
    participant TCP as TcpClient
    participant AST as Asterisk Server

    App->>AMI: StartAsync (IHostedService)
    AMI->>TCP: ConnectAsync(host:5038)
    TCP->>AST: TCP SYN
    AST-->>TCP: TCP ACK
    AST-->>AMI: "Asterisk Call Manager/2.x\r\n"
    AMI->>AMI: Start ReaderLoopAsync (background Task)
    AMI->>AST: Action: Login\r\nUsername: admin\r\nActionID: abc123\r\n\r\n
    AST-->>AMI: Response: Success\r\nActionID: abc123\r\n\r\n
    AMI->>AMI: IsConnected = true

    loop Every AMI Event
        AST-->>AMI: Event: AgentCalled\r\nQueue: sales\r\n...\r\n\r\n
        AMI->>AMI: Parse block
        AMI->>AMI: Write to Channel<AmiEvent>
    end

    Note over App,AMI: Controller requests queue status
    App->>AMI: GetQueueStatusAsync()
    AMI->>AST: Action: QueueStatus\r\nActionID: xyz789\r\n\r\n
    AST-->>AMI: Response: Success\r\nActionID: xyz789\r\n\r\n
    AST-->>AMI: Event: QueueParams\r\nQueue: sales\r\n...\r\n\r\n
    AST-->>AMI: Event: QueueMember\r\nQueue: sales\r\n...\r\n\r\n
    AST-->>AMI: Event: QueueStatusComplete\r\nActionID: xyz789\r\n\r\n
    AMI->>AMI: BulkPending TCS completed
    AMI-->>App: List<AmiQueueStatus>
```

---

### 5.4 Queue Log Import Flow

```mermaid
flowchart TD
    A([Timer fires\nevery 5 minutes]) --> B{QueueLogPath\nconfigured?}
    B -- No --> Z([Skip])
    B -- Yes --> C[Open queue_log file\nFileShare.ReadWrite]
    C --> D{File shrank?\nlog rotation?}
    D -- Yes --> E[Reset offset to 0]
    D -- No --> F[Seek to saved offset]
    E --> F
    F --> G[Read up to\nQueueLogBatchSize lines]
    G --> H["ParseLine each entry\ntimestamp | callid | queue | agent | event | d1 | d2 | d3"]
    H --> I{Entries found?}
    I -- No --> Z
    I -- Yes --> J[Load existing keys\nfrom DB for dedup]
    J --> K[Filter: only new entries\nnot already in DB]
    K --> L{New entries > 0?}
    L -- No --> M[Update offset]
    L -- Yes --> N[db.QueueLogEntries.AddRange\nSaveChangesAsync]
    N --> M
    M --> Z
```

---

### 5.5 Scheduled Report Flow

```mermaid
flowchart TD
    A([Timer fires\nevery 1 minute]) --> B[Load active\nScheduledReports from DB]
    B --> C{For each report:\nIsDue?}
    C --> D{CurrentHour ==\nRunHour?}
    D -- No --> E([Skip this report])
    D -- Yes --> F{LastRunAt within\nlast 50 minutes?}
    F -- Yes --> E
    F -- No --> G{Frequency?}
    G -- Daily --> H[Due → Run]
    G -- Weekly --> I{Today ==\nMonday?}
    I -- No --> E
    I -- Yes --> H
    G -- Monthly --> J{Today ==\n1st?}
    J -- No --> E
    J -- Yes --> H
    H --> K[Generate report\nby ReportType]
    K --> L[Send to Recipients\nvia email]
    L --> M[Update LastRunAt = now\nSaveChangesAsync]
    M --> C
```

---

### 5.6 Agent Status Update Flow

```mermaid
flowchart TD
    A([Agent on /AgentPanel/Index]) --> B[Selects new Status\noptionally Queue + Break Reason]
    B --> C[POST /AgentPanel/UpdateStatus]
    C --> D[Get AgentId from\nClaims Principal]
    D --> E{AgentQueueStatus\nexists for agent?}
    E -- No --> F[Create new\nAgentQueueStatus record]
    E -- Yes --> G[Load existing record]
    F --> H[Set Status, QueueId, UpdatedAt]
    G --> H
    H --> I{Status == OnBreak?}
    I -- Yes --> J[Set BreakReason]
    I -- No --> K[Clear BreakReason = null]
    J --> L[SaveChangesAsync]
    K --> L
    L --> M[TempData success message]
    M --> N([Redirect to AgentPanel/Index])
```

---

### 5.7 Call Registration Flow

```mermaid
flowchart TD
    A([Agent/Supervisor\nclicks Register Call]) --> B[GET /Calls/Create]
    B --> C[Fill form:\nCustomer, AgentName,\nStartTime, Duration,\nRecordingUrl, Status, Notes]
    C --> D[POST /Calls/Create]
    D --> E{ModelState\nvalid?}
    E -- No --> C
    E -- Yes --> F[Create VoiceCall entity\nStartTime = UTC]
    F --> G[db.VoiceCalls.Add\nSaveChangesAsync]
    G --> H[Redirect to Calls/Index\nor Dashboard]
```

---

## 6. Request Lifecycle Flowchart

```mermaid
flowchart LR
    Browser -->|HTTPS Request| MW1[UseHttpsRedirection]
    MW1 --> MW2[UseRouting]
    MW2 --> MW3[UseAuthentication\nRead Identity Cookie]
    MW3 --> MW4[UseAuthorization\nCheck Role + Policy]
    MW4 --> MW5{Authorized?}
    MW5 -- No --> RD["Redirect /Account/AccessDenied"]
    MW5 --Yes--> CTR[Controller Action]
    CTR --> SVC[Service / DbContext]
    SVC --> DB[(SQL Server)]
    DB --> SVC
    SVC --> CTR
    CTR --> VIEW[Razor View\nRender HTML]
    VIEW --> Browser
```

---

