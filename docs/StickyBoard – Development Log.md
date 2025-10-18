# StickyBoard – Development Log

A chronological record of tasks, milestones, and key decisions for the StickyBoard project.

------

## Project Timeline

### October 17, 2025 (Friday)

**Milestone:** Project Direction and Deployment Planning

- Drafted full StickyBoard Production Deployment Guide.
- Defined complete Docker + Apache reverse proxy stack.
- Decided to host under the subdomain stickyboard.aedev.pro.
- Established the architecture for a unified deployment using Docker, Apache, and HTTPS.
- Set the foundation for future frontend + backend integration.

Result: First full version of the deployment plan completed. Environment architecture validated conceptually.

------

### October 18, 2025 (Saturday)

**Milestone:** Server Setup and Documentation Finalization

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

- Completed full deployment & CI/CD documentation, including GitHub Actions workflow.

- Validated Apache proxy configuration for /api/** route.

Result: Deployment environment fully operational. Repo now acts as a single self-contained project. Ready to initialize the .NET 9 API.

------

## Upcoming Tasks

| Target Date         | Task                      | Description                                                  |
| ------------------- | ------------------------- | ------------------------------------------------------------ |
| October 19–20, 2025 | Initialize .NET 9 Web API | Scaffold the StickyBoard backend with a health endpoint and PostgreSQL integration. |
| October 21, 2025    | Database integration      | Configure Entity Framework Core and implement initial migrations. |
| October 22–23, 2025 | Authentication system     | Add JWT-based authentication (users table, login/register endpoints). |
| October 24–26, 2025 | Core feature development  | Implement main API modules (boards, posts, notifications, etc.). |
| Following week      | Frontend integration      | Build frontend skeleton and connect to /api endpoints.       |

------

## Notes

- Current environment: Ubuntu + Apache 2 + Docker + PostgreSQL 17 + .NET 9
- Deployment URL: [https://stickyboard.aedev.pro](https://stickyboard.aedev.pro/)
- All SSL and proxying verified via Apache + Let’s Encrypt.
- CI/CD is handled automatically via GitHub Actions on push to main.

------

## Next Steps

1. Scaffold backend API (dotnet new webapi) inside /api/.
2. Add a simple /api/health endpoint and test via HTTPS.
3. Create initial Dockerfile for the API service and rebuild the stack.
4. Begin database schema and authentication design.

------

Maintained manually by Alexandre Emond – StickyBoard project lead.
 (Last updated: October 18, 2025)