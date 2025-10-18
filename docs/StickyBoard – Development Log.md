# StickyBoard – Development Log

A chronological record of tasks, milestones, and key decisions for the StickyBoard project.

------

## Project Timeline

### October 17, 2025 (Friday)

**Milestone:** Project Direction and Deployment Planning

- Drafted full StickyBoard Production Deployment Guide.
- Defined complete Docker + Apache reverse proxy stack.
- Decided to host under the subdomain **stickyboard.aedev.pro**.
- Established architecture for unified deployment (Docker + Apache + HTTPS).
- Set the foundation for future frontend and backend integration.

**Result:**
 First full version of the deployment plan completed. Environment architecture validated conceptually.

------

### October 18, 2025 (Saturday)

**Milestone 1:** Server Setup and Documentation Finalization

- Created and verified DNS entry for `stickyboard.aedev.pro`.

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

- Completed full deployment & CI/CD documentation, including GitHub Actions workflow.

- Validated Apache proxy configuration for `/api/**` route.

**Result:**
 Deployment environment fully operational. Repository now acts as a single self-contained project. Ready to initialize the .NET 9 API.

------

**Milestone 2:** Multi-Repository Architecture and Automation

- Established the **StickyBoard multi-platform ecosystem** structure:

  - `stickyboard-api` – Backend (.NET 9 + PostgreSQL)
  - `stickyboard-ios` – Native SwiftUI client
  - `stickyboard-android` – Kotlin Jetpack Compose client
  - `stickyboard-macos` – SwiftUI desktop version
  - `stickyboard-windows` – .NET MAUI / WPF desktop version
  - All linked within the umbrella repository `stickyboard`.

- Initialized each platform repository with an initial commit and `main` branch.

- Added all submodules to the umbrella repository and validated the structure.

- Configured `.gitmodules` for automatic path and URL mapping.

- Implemented **GitHub Actions automation** across all submodules:

  - Each submodule automatically updates the umbrella repo when new commits are pushed to `main`.

  - Confirmed successful synchronization for all five submodules.

  - Verified automated commit sequence:

    ```
    Auto-sync stickyboard-<platform> to latest commit
    ```

- Confirmed umbrella repo integrity with correct commit pointers for all submodules.

**Result:**
 Submodule automation and synchronization are fully functional. Umbrella repository now acts as the centralized reference for all StickyBoard components. All commits and CI/CD pipelines are validated.

------

## Upcoming Tasks

| Target Date         | Task                      | Description                                                  |
| ------------------- | ------------------------- | ------------------------------------------------------------ |
| October 19–20, 2025 | Initialize .NET 9 Web API | Scaffold the StickyBoard backend with a health endpoint and PostgreSQL link. |
| October 21, 2025    | Database integration      | Configure Entity Framework Core, add initial migrations, and seed baseline data. |
| October 22–23, 2025 | Authentication system     | Implement JWT-based authentication (users table, register/login endpoints). |
| October 24–26, 2025 | Core API features         | Develop main modules: boards, notes, clusters, notifications, and sync. |
| Following week      | Frontend integration      | Begin connecting mobile/desktop clients to the API endpoints. |
| Later phase         | Automation refinement     | Add umbrella-side daily sync workflow and tag propagation system. |

------

## Notes

- **Current Environment:** Ubuntu + Apache 2 + Docker + PostgreSQL 17 + .NET 9
- **Automation:** Full submodule auto-sync via GitHub Actions
- **Deployment URL:** [https://stickyboard.aedev.pro](https://stickyboard.aedev.pro/)
- **Security:** SSL (Let’s Encrypt) + Apache reverse proxy verified
- **Repositories:** All five platform submodules linked under umbrella `stickyboard`
- **Branching Policy:** `main` only (feature branches to be introduced post-API setup)

------

## Next Steps

1. Scaffold backend API (`dotnet new webapi`) inside `/api/`.
2. Implement `/api/health` test endpoint and verify HTTPS response.
3. Create and test API Dockerfile; integrate into unified stack.
4. Begin database schema and authentication model design.
5. Add daily umbrella sync workflow and optional tag propagation CI.

------

**Maintained by:** Alexandre Emond
 **Role:** StickyBoard Project Lead
 **Last Updated:** **October 18, 2025**