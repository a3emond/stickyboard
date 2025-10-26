# **StickyBoard – Development Log**

**Last Updated:** ****October 24, 2025 (Friday)****



A chronological record of tasks, milestones, and key decisions for the StickyBoard project.



------





## **Project Timeline**

### **October 17, 2025 (Friday)**

**Milestone:** Project Direction and Deployment Planning



- Drafted full StickyBoard Production Deployment Guide.
- Defined complete Docker + Apache reverse proxy stack.
- Decided to host under the subdomain **stickyboard.aedev.pro**.
- Established architecture for unified deployment (Docker + Apache + HTTPS).
- Set the foundation for future frontend and backend integration.





**Result:**

First full version of the deployment plan completed. Environment architecture validated conceptually.



------





### **October 18, 2025 (Saturday)**





**Milestone 1:** Server Setup and Documentation Finalization



- Created and verified DNS entry for stickyboard.aedev.pro.
- Configured Apache VirtualHost and Let’s Encrypt SSL certificates.
- Verified live HTTPS access with placeholder landing page.
- Defined unified project structure:



```
/var/www/html/stickyboard.aedev.pro/
├── frontend/
├── api/
├── worker/
├── logs/
├── docker-compose.yml
├── .env
└── .github/workflows/deploy.yml
```



- 
- Completed full deployment & CI/CD documentation, including GitHub Actions workflow.
- Validated Apache proxy configuration for /api/** route.





**Result:**

Deployment environment fully operational. Repository now acts as a single self-contained project. Ready to initialize the .NET 9 API.



------



**Milestone 2:** Multi-Repository Architecture and Automation



- Established the **StickyBoard multi-platform ecosystem** structure:

  

  - stickyboard-api – Backend (.NET 9 + PostgreSQL)
  - stickyboard-ios – Native SwiftUI client
  - stickyboard-android – Kotlin Jetpack Compose client
  - stickyboard-macos – SwiftUI desktop version
  - stickyboard-windows – .NET MAUI / WPF desktop version
  - All linked within the umbrella repository stickyboard.

  

- Initialized each platform repository with an initial commit and main branch.

- Added all submodules to the umbrella repository and validated the structure.

- Configured .gitmodules for automatic path and URL mapping.

- Implemented **GitHub Actions automation** across all submodules:

  

  - Each submodule automatically updates the umbrella repo when new commits are pushed to main.
  - Confirmed successful synchronization for all five submodules.
  - Verified automated commit sequence:

  



```
Auto-sync stickyboard-<platform> to latest commit
```



- 
- Confirmed umbrella repo integrity with correct commit pointers for all submodules.





**Result:**

Submodule automation and synchronization are fully functional. Umbrella repository now acts as the centralized reference for all StickyBoard components. All commits and CI/CD pipelines are validated.



------





### **October 19, 2025 (Sunday)**





**Milestone:** Backend API Initialization and Container Integration



- Scaffolded initial .NET 9 Web API project inside /api.

- Implemented a lightweight Program.cs startup with async PostgreSQL connection test.

- Replaced the default WeatherForecast template with a minimal clean setup.

- Added environment-variable-based connection logic (DATABASE_URL, POSTGRES_*).

- Unified .env naming across Docker and IDE for seamless dev/prod parity.

- Created initial launchSettings.json to mirror .env variables (for Rider/VS/VS Code).

- Built and validated **Dockerfile** for the API container:

  

  - Multi-stage build (SDK → Runtime).
  - Publishes app to /app/publish and runs via dotnet StickyBoard.Api.dll.

  

- Fixed compose mount collision by removing ./api:/app bind mount.

- Verified successful startup of stickyboard_api and stickyboard_db containers.

- Disabled unused worker service temporarily to stabilize stack.





**Result:**

Backend environment confirmed operational. Containers build and run cleanly; PostgreSQL connectivity confirmed. Ready for first endpoint and schema work.



------





### **October 20, 2025 (Monday)**





**Milestone:** Core Architecture Refinement & DTO Layer Definition



- Fully **revised the PostgreSQL schema** after reevaluating the system architecture.
- **Sanitized and consolidated database structure** to better support modular design and scalability.
- **Introduced worker/runner integration** for background job processing — offloading heavy or asynchronous tasks from the main API service.
- Refactored all related **technical documentation**, including API and Worker Plans, for alignment with the new structure.
- Defined a complete **DTO (Data Transfer Object) layer** covering all controller interactions across authentication, boards, cards, sync, and admin endpoints.
- Established clear **architectural boundaries** between Entities (persistence models) and DTOs (API-facing contracts) for clean separation of concerns.
- Prepared the foundation to implement a **Generic Base Repository** pattern to standardize database access logic across all models.





**Result:**

StickyBoard’s backend architecture is now fully modular, scalable, and ready for the repository/service layer build-out.

The API layer is standardized, worker-ready, and DTO-complete — setting the stage for clean controller implementation and future client integrations.



------

### **October 21, 2025 (Tuesday)**

**Milestone:** Model Refactor and Repository Layer Implementation

- **Refactored all models** to align precisely with the finalized PostgreSQL schema.
  - Ensured column-level parity with the database, including JSONB fields and ENUM types.
  - Rebuilt entity classes with `[Table]` and `[Column]` attributes to maintain a consistent mapping layer without using Entity Framework.
  - Introduced `IEntity` and `IEntityUpdatable` interfaces to enforce structural consistency and enable generic repository constraints.
  - Removed unnecessary navigation properties to keep models lightweight and persistence-focused, in line with the **raw SQL / DataReader** approach.
- **Redefined architectural intent of the API**:
  - The backend is not a generic CRUD service but a **synchronization and orchestration layer** for StickyBoard’s local-first ecosystem.
  - Reconfirmed that join tables (e.g., `card_tags`, `cluster_members`) are **not domain entities** but logical relationships managed by higher-level services or worker processes.
  - Clarified that derived tables (clusters, indexes, etc.) are **computed artifacts**, rebuilt by the worker system rather than directly exposed as resources.
- **Implemented the complete Repository Layer**:
  - Built a **generic `RepositoryBase<T>`** using `Npgsql` and `async` patterns for all CRUD operations.
  - Added a **reflection-based `MappingHelper`** to auto-map database columns to model properties via `[Column]` attributes, eliminating manual mapping repetition.
  - Created specialized repositories for all entities (Users, Boards, Sections, Tabs, Cards, Tags, Links, Clusters, Rules, Activities, Files, Operations, WorkerJobs).
  - Preserved immutability where applicable (e.g., `Activity`, `Operation` updates disabled).
  - Ensured clean, database-agnostic transaction handling suitable for future multi-database or offline sync extensions.
- **Design Decision Summary:**
  - Abandoned Entity Framework to maintain full **control over SQL performance and query optimization**, aligning with StickyBoard’s **offline-first** and **data-replicable** nature.
  - Consolidated the model layer under a strict contract system (`IEntity` / `IEntityUpdatable`) to guarantee type safety and uniform CRUD capabilities.
  - Delegated link and derived data management (e.g., tags, clusters) to higher logic layers, simplifying persistence and improving maintainability.
  - This structure now supports **deterministic data flow**, essential for CRDT-based sync, background worker tasks, and local delta propagation.

**Result:**
 All backend persistence layers are now fully operational, consistent with the PostgreSQL schema, and decoupled from ORM dependencies.
 The API core is now data-complete, worker-compatible, and ready for the **service layer** and **controller integration** phases.

------

### **October 22, 2025 (Wednesday)**

**Milestone:** Authentication System Implementation & Infrastructure Stability Fixes

- **Implemented full authentication layer** with JWT support.
  - Developed `/api/Auth/register` and `/api/Auth/login` endpoints.
  - Integrated secure password hashing using `BCrypt` and proper JWT token issuance via `JwtTokenService`.
  - Created corresponding DTOs for registration, login, and response payloads, ensuring consistent request validation and return structure.
- **Introduced dependency-injected service architecture** for authentication.
  - Added `IAuthService` interface and `AuthService` implementation with clear separation of concerns.
  - Injected `UserRepository` and `AuthUserRepository` via constructor-based DI for cleaner lifecycle management.
  - Defined centralized `JwtOptions` configuration section for environment-bound secret and token claims.
- **Resolved critical dependency injection (DI) issues** caused by incorrect repository constructors and outdated Npgsql connection handling.
  - Migrated from direct `NpgsqlConnection` usage to the new `NpgsqlDataSource` pattern introduced in **Npgsql 7.x**.
  - Updated `RepositoryBase<T>` to use a data-source-backed connection pool with enum-aware mapping.
  - Registered all PostgreSQL `ENUM` types in `Program.cs` to maintain strict type fidelity across database and C# models.
- **Refined Swagger and local development setup:**
  - Added cache-busting for `swagger.json` to prevent stale documentation.
  - Enabled JWT authorization headers directly in Swagger UI for easy API testing.
  - Standardized port configuration and isolated development launch profiles (`http` and `https`) for Rider/VS debugging.
- **Validated authentication workflow end-to-end**:
  - Confirmed working registration, user persistence, password hashing, and token generation.
  - Resolved duplicate-key and enum mapping exceptions during initial database writes.
  - Verified integration with the live PostgreSQL 17 instance inside Docker Compose.

**Result:**
 Authentication system is now fully operational, secure, and integrated into the StickyBoard API.
 The backend now uses a modern, dependency-injected architecture with type-safe database access and fully functional Swagger documentation.

------

### **October 24, 2025 (Friday)**

**Milestone 1:** Cancellation Token Integration, Async Pipeline Standardization, and Architecture Stabilization

- Completed the design and planning phase for full **`CancellationToken` propagation** across all asynchronous layers of the StickyBoard API.

- Updated the **repository and service architecture** to adopt a unified async signature convention — every core I/O operation (`CreateAsync`, `UpdateAsync`, `DeleteAsync`, `GetAllAsync`, `GetByIdAsync`) now receives and forwards a `CancellationToken ct`.

- Refactored the **`RepositoryBase<T>`** class to propagate tokens from the controller layer down to database operations.

- Verified **Npgsql** async operations (`OpenConnectionAsync`, `ExecuteReaderAsync`, `ExecuteScalarAsync`, etc.) are token-aware, ensuring proper cooperative cancellation for PostgreSQL queries.

- Ensured **ASP.NET Core’s** built-in request lifetime token (`HttpContext.RequestAborted`) is passed automatically into controllers, cascading through services and repositories without manual interruption handling.

- Standardized the **async pipeline** to guarantee that all pending operations terminate cleanly if a client disconnects mid-request.

- Discussed and documented the new propagation flow in the architecture plan:

  ```
  Controller → Service → Repository → Database (Npgsql)
  ```

- Confirmed that cancellation-aware patterns reduce risk of orphaned database sessions, improve scalability, and prepare the API for high concurrency under load.

**Result:**
 The StickyBoard backend architecture is now **cancellation-ready and fully async-cooperative**.
 This refactor marks the stabilization of the core pipeline, setting the stage for further service expansion and worker integration built on a consistent, cancellation-safe data layer.

**Milestone 2:** JWT + Refresh Token System, Seamless Session Renewal, and Secure Revocation Workflow

- Implemented a **hybrid authentication model** supporting short-lived access tokens and renewable refresh tokens.
- Extended the **`AuthService`**, **`AuthController`**, and **`RefreshTokenRepository`** to support:
  - Refresh token issuance and rotation on login/register.
  - Token renewal via `/api/auth/refresh`.
  - Full user logout and token revocation via `/api/auth/logout`.
- Refactored DTOs (`AuthResponseDto`, `RefreshRequestDto`) to include both tokens and unified user identity data.
- Updated `Program.cs` to ensure **`JwtOptions`** are properly injected, **token lifetimes are validated**, and **refresh token persistence** is available through dependency injection.
- Integrated refresh-token persistence with PostgreSQL using hashed identifiers and one-time rotation semantics.
- Verified that `JwtTokenService` generates consistent tokens respecting expiration, audience, and issuer claims defined in `appsettings.json`.
- Confirmed that the refresh token system supports both **offline-first native clients** (secure local storage) and **web-based admin sessions** (short-lived JWTs with sliding refresh).

**Result:**
 The StickyBoard authentication pipeline now provides **secure, persistent sessions with stateless JWT validation** and **revocable, database-tracked refresh tokens**, enabling both web and native clients to maintain long-lived authenticated states without compromising security.

------

**Milestone 3:** Collaboration Layer Foundation and Model Integration

- Introduced the **Collaboration Layer** into the StickyBoard API, expanding beyond individual user boards to full multi-user, organization-aware collaboration.
- Added all new **core models**:
  - `Organization`, `OrganizationMember`, `UserRelation`, `Message`, and `Invite` (plus planned `Notification` view).
- Updated existing models (`Board`, `Permission`, `User`, `Tab`, `Card`, `RefreshToken`) to support collaborative ownership, shared visibility, and scoped access.
- Extended database schema (PostgreSQL 17) accordingly — see `001_schema.sql`.
- Implemented **new repository classes** for each new model under `Repositories/Organizations`, `Repositories/Messaging`, and `Repositories/Social`.
- Established **service layer contracts** for upcoming `OrganizationService`, `BoardSharingService`, `FriendService`, `MessageService`, and `InviteService`.
- Updated dependency injection structure and base repository to ensure cancellation tokens propagate correctly through new entities.
- Revised documentation and schema diagrams to reflect the multi-tier collaboration system, matching the vision detailed in the architecture overview and README.

**Result:**
 StickyBoard now includes a fully defined **Collaboration Domain**. The API supports teams, friendships, and messaging within one unified model, setting the groundwork for organization management, invitations, and shared boards.

------

### **October 25, 2025 (Saturday)**

**Milestone:** Collaboration Layer Completion and Functional Expansion

- Completed the **Collaboration Layer** implementation across repositories, services, and controller planning (Organizations, Friends, Messages, Invites).
- Finalized **board-sharing and membership logic** following the structure described in *1. Files to Add / Update.md*.
- Verified relational integrity and tested multi-role propagation:
  - Organization roles automatically map to board roles (Owner → Owner, Admin → Editor, etc.).
  - User invitations generate both permission entries and in-app messages.
- Added schema-verified social layer (`user_relations`) and messaging system (`messages`, `invites`) supporting direct, system, and org-invite message types.
- Prepared foundation for `Notification` projection table or view for future worker/SSE integration.
- Updated `RepositoryBase` and all derived repositories to enforce consistent async token propagation and exception safety.
- Reviewed and aligned the architecture plan with the unified **Collaboration + Sync + Auth** flow now documented in `Architecture Overview.md`.

**Result:**
 The collaboration system is now feature-complete at the data and repository levels.
 The API is ready for controller implementation and integration testing of organization creation, member management, invites, and friend workflows.

------

## **Timeline**

| **Target Date**         | **Task**              | **Description**                                              |
| ----------------------- | --------------------- | ------------------------------------------------------------ |
| **October 20–21, 2025** | Database integration  | Implement connection helper and seed logic using async Npgsql DataReader pattern. |
| **October 22–23, 2025** | Authentication system | Implement JWT auth (users table, register/login endpoints).  |
| **October 24–26, 2025** | Core API features     | Develop main modules: boards, notes, clusters, notifications, and sync. |
| **Following week**      | Frontend integration  | Begin connecting mobile/desktop clients to API endpoints.    |
| **Later phase**         | Automation refinement | Add umbrella-side daily sync workflow and tag propagation system. |



------





## **Notes**





- **Current Environment:** Ubuntu + Apache 2 + Docker + PostgreSQL 17 + .NET 9
- **Automation:** Full submodule auto-sync via GitHub Actions
- **Deployment URL:** https://stickyboard.aedev.pro
- **Security:** SSL (Let’s Encrypt) + Apache reverse proxy verified
- **Repositories:** All five platform submodules linked under umbrella stickyboard
- **Branching Policy:** main only (feature branches to be introduced post-API setup)





------



**Maintained by:** Alexandre Emond

**Role:** StickyBoard Project Lead

**Last Updated:** **October 20, 2025**