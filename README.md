# StickyBoard – Project Overview (2025-10-24)

StickyBoard is a **cross-platform collaborative workspace** designed for creativity, productivity, and intelligent organization. It unifies personal notes, project boards, and team collaboration into a single local-first ecosystem with seamless offline and cloud sync.

This repository acts as the **documentation and coordination hub** for the entire StickyBoard ecosystem, providing architecture references and linking platform-specific submodules.

| Repository                                                   | Platform | Description                                     |
| ------------------------------------------------------------ | -------- | ----------------------------------------------- |
| [stickyboard-api](https://github.com/a3emond/stickyboard-api) | Backend  | ASP.NET Core REST API + PostgreSQL (Dockerized) |
| [stickyboard-ios](https://github.com/a3emond/stickyboard-ios) | iOS      | Native SwiftUI app for iPhone and iPad          |
| [stickyboard-android](https://github.com/a3emond/stickyboard-android) | Android  | Native Kotlin app                               |
| [stickyboard-macos](https://github.com/a3emond/stickyboard-macos) | macOS    | Desktop version built in SwiftUI                |
| [stickyboard-windows](https://github.com/a3emond/stickyboard-windows) | Windows  | Desktop version built in .NET MAUI              |

------
# StickyBoard — Project Presentation

## Overview

StickyBoard is a large-scale, multi-platform collaborative workspace system designed and developed as a complete end-to-end software ecosystem. The project was conceived to explore and implement modern software engineering practices across mobile development, backend engineering, database architecture, synchronization, and system deployment. Unlike a typical academic assignment, StickyBoard was built as a real-world application with production-style architecture, multi-repository organization, and full infrastructure ownership including backend hosting and deployment.

The system enables users to create workspaces, boards, and structured visual content using cards, sections, and views. It supports collaborative editing, messaging, file attachments, and secure invitation flows. A major design objective is to provide a local-first user experience: each client device maintains a local database and can operate fully offline, synchronizing changes with the server when connectivity is restored. This ensures responsiveness, reliability, and consistency across devices. StickyBoard is implemented as a native multi-platform system, with dedicated applications for iOS (SwiftUI), Android (Kotlin), macOS (SwiftUI), and Windows (.NET MAUI), all sharing a unified backend and synchronization model.

This project required independent work across the full software lifecycle, including system design, architecture modeling, backend implementation, mobile client development, database schema design, synchronization logic, authentication systems, file storage management, realtime infrastructure, and self-hosted deployment.

---

## Project Scope and Technical Scale

StickyBoard is structured as a multi-repository ecosystem, where each platform and subsystem is developed independently but follows a shared architecture and domain model.

### Main Components

| Component          | Technology                        | Responsibility                                               |
| ------------------ | --------------------------------- | ------------------------------------------------------------ |
| Backend API        | ASP.NET Core (.NET 9), PostgreSQL | Core domain logic, authentication, sync, collaboration       |
| iOS Client         | SwiftUI                           | Native mobile interface, local database, sync client         |
| Android Client     | Kotlin                            | Native mobile interface and sync client                      |
| Windows Client     | .NET MAUI                         | Cross-platform desktop client                                |
| macOS Client       | SwiftUI                           | Native desktop client                                        |
| Database           | PostgreSQL                        | Persistent storage, domain constraints, sync source of truth |
| Realtime Layer     | Event outbox + Firebase           | Multi-device synchronization and notifications               |
| File Storage       | Self-hosted CDN + signed URLs     | Secure attachment storage and delivery                       |
| Background Workers | .NET Services                     | Event processing, notifications, and maintenance tasks       |

Each component was designed and implemented to operate independently while maintaining strict consistency and interoperability.

---

## Architecture and Design Principles

The project follows modern layered architecture and domain-driven design principles, ensuring separation of concerns, maintainability, and scalability.

### Core Architectural Layers

* Presentation Layer

  * Native UI applications on each platform
  * Platform-specific integration and local persistence

* Service Layer

  * Business logic and domain orchestration
  * Validation, permissions, and workflow coordination

* Repository Layer

  * Direct database access using parameterized SQL
  * Custom repository pattern built on Npgsql

* Database Layer

  * PostgreSQL schema with strict constraints and triggers
  * Versioning, soft deletes, and audit tracking

* Synchronization Layer

  * Event outbox pattern for deterministic realtime sync
  * Cursor-based incremental update propagation

* Infrastructure Layer

  * Authentication, file storage, background workers, and deployment

This layered structure ensures that each responsibility is isolated, enabling clean architecture and long-term extensibility.

---

## Advanced Backend and Database Engineering

The backend was implemented using ASP.NET Core with a strong focus on architectural correctness and database-level integrity.

### Key Backend Features

* REST API using DTO-based contracts
* Repository pattern without ORM to maintain full SQL control
* JWT authentication with refresh token rotation
* Role-based access control for workspaces and boards
* Secure invitation system using cryptographic token hashing
* Soft-delete system preserving audit history
* Optimistic concurrency control using versioning
* Background worker queue for asynchronous processing

### Database Architecture

The PostgreSQL database was designed as the authoritative source of truth and enforces many domain rules directly.

Key mechanisms include:

* Foreign key constraints ensuring relational integrity
* Database triggers maintaining versioning and timestamps
* Event outbox pattern enabling reliable realtime synchronization
* Soft deletion preserving historical data
* Optimized indexing for performance and sync queries

This approach ensures consistency, reliability, and safe multi-device synchronization.

---

## Multi-Platform Native Client Development

StickyBoard clients were developed using native frameworks for each platform to leverage operating system capabilities fully.

### Platform Technologies

| Platform | Technology |
| -------- | ---------- |
| iOS      | SwiftUI    |
| Android  | Kotlin     |
| macOS    | SwiftUI    |
| Windows  | .NET MAUI  |

Each client implements:

* Local database storage for offline operation
* Synchronization engine with conflict detection
* Secure authentication handling
* Platform-native UI and interaction models

This demonstrates cross-platform system design while maintaining native performance and integration.

---

## Synchronization and Realtime Architecture

One of the most technically advanced aspects of StickyBoard is its synchronization system.

### Sync Mechanisms

* Event Outbox Pattern

  * Database triggers record all changes into a sync event log
  * Ensures reliable and ordered propagation of updates

* Cursor-Based Delta Sync

  * Clients fetch only changes since their last sync
  * Minimizes network usage and improves efficiency

* Offline-First Model

  * Clients operate independently with local databases
  * Synchronization occurs automatically when connectivity resumes

* Realtime Notifications

  * Background workers propagate events to connected clients

This architecture ensures data consistency across multiple devices while supporting offline usage.

---

## File Storage and Security Architecture

StickyBoard implements secure file handling using a dedicated CDN and signed URL access control.

### File System Features

* Files stored outside the API for performance and scalability
* Secure access using time-limited signed URLs
* Cryptographic token validation using HMAC
* Support for attachment variants such as previews and thumbnails
* Centralized attachment management service

This design ensures security, scalability, and efficient file delivery.

---

## Deployment, Hosting, and Infrastructure

The StickyBoard backend is fully self-hosted and deployed independently.

### Infrastructure Responsibilities

* Backend deployment on a personal server
* HTTPS configuration and reverse proxy setup
* PostgreSQL database installation and maintenance
* CDN setup and secure file serving
* Background worker process management
* Version control and multi-repository coordination

This demonstrates full-stack ownership, including infrastructure and deployment management.

---

## Software Engineering Concepts and Design Patterns Demonstrated

StickyBoard incorporates multiple advanced architectural and design concepts covered in software engineering and design pattern courses.

### Concepts Implemented

* Layered Architecture
* Repository Pattern
* Service Layer Pattern
* Event Outbox Pattern
* Domain-driven design principles
* Separation of concerns
* Optimistic concurrency control
* Secure token-based authentication
* Transactional database logic using stored functions
* Multi-client synchronization architecture

These patterns are commonly used in professional software systems and demonstrate strong architectural understanding.

---

## Educational Value and Learning Outcomes

This project provided hands-on experience with real-world software engineering challenges, including:

* Designing and implementing a complete backend system
* Developing native applications across multiple platforms
* Implementing secure authentication systems
* Designing relational database schemas with integrity constraints
* Implementing synchronization and realtime systems
* Managing file storage securely
* Deploying and hosting production-style infrastructure
* Applying software architecture and design pattern principles

StickyBoard represents a comprehensive application of both mobile development and software architecture concepts in a unified, real-world system.

---

## Conclusion

StickyBoard is a large-scale software engineering project that integrates mobile development, backend architecture, database engineering, synchronization systems, and infrastructure management into a single cohesive system. It demonstrates the ability to design, implement, deploy, and maintain a complex multi-platform application using modern software engineering practices and architectural principles.

------
## License

All rights reserved — © Alexandre Emond, 2025. Unauthorized reproduction or redistribution of this project’s source code, in whole or in part, is strictly prohibited without express written permission.
