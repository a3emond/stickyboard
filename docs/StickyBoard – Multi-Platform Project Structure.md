# StickyBoard – Multi-Platform Project Structure

This repository serves as the **umbrella project** for StickyBoard, a multi-platform application ecosystem consisting of:

- A centralized backend API (ASP.NET Core / PostgreSQL)
- Native clients for iOS, Android, macOS, and Windows

Each platform lives in its own independent GitHub repository, tracked here as a **Git submodule** for coordinated versioning and documentation.

------

## Repository Overview

| Repository                                                   | Platform | Description                                   |
| ------------------------------------------------------------ | -------- | --------------------------------------------- |
| [stickyboard-api](https://github.com/a3emond/stickyboard-api) | Backend  | .NET 9 REST API (Docker + PostgreSQL)         |
| [stickyboard-ios](https://github.com/a3emond/stickyboard-ios) | iOS      | Native iOS app built in SwiftUI               |
| [stickyboard-android](https://github.com/a3emond/stickyboard-android) | Android  | Native Android app (Kotlin + Jetpack Compose) |
| [stickyboard-macos](https://github.com/a3emond/stickyboard-macos) | macOS    | Desktop version built with SwiftUI            |
| [stickyboard-windows](https://github.com/a3emond/stickyboard-windows) | Windows  | Desktop version built in .NET MAUI or WPF     |

All these repositories are tracked as submodules inside this parent repository:
 [stickyboard](https://github.com/a3emond/stickyboard)

------

## Folder Structure

```
stickyboard/
├── stickyboard-api/          # Backend API (Dockerized .NET 9 service)
├── stickyboard-ios/          # Native iOS app
├── stickyboard-android/      # Native Android app
├── stickyboard-macos/        # macOS desktop client
├── stickyboard-windows/      # Windows desktop client
├── docs/                     # Shared project documentation
├── .gitmodules               # Git submodule configuration
└── README.md                 # You are here
```

Each subfolder (except `docs/`) is a linked repository, not a simple directory.

------

## Setup Procedure

### 1. Create Sub-Repositories on GitHub

Create **empty public or private repos** under your account for each platform:

-  stickyboard-api
-  stickyboard-ios
-  stickyboard-android
-  stickyboard-macos
-  stickyboard-windows

No need to add README or license files yet (Git will handle linking automatically).

------

### 2. Clone the Umbrella Repo

```bash
git clone git@github.com:a3emond/stickyboard.git
cd stickyboard
```

------

### 3. Add Each Submodule

Add the submodules in their respective directories:

```bash
git submodule add https://github.com/a3emond/stickyboard-api.git stickyboard-api
git submodule add https://github.com/a3emond/stickyboard-ios.git stickyboard-ios
git submodule add https://github.com/a3emond/stickyboard-android.git stickyboard-android
git submodule add https://github.com/a3emond/stickyboard-macos.git stickyboard-macos
git submodule add https://github.com/a3emond/stickyboard-windows.git stickyboard-windows
```

Then commit and push:

```bash
git add .gitmodules stickyboard-*
git commit -m "Add StickyBoard platform submodules"
git push origin main
```

------

### 4. Verify Structure

Run:

```bash
git submodule status
```

You should see output similar to:

```
 1234abcd1234abcd1234abcd1234abcd1234abcd stickyboard-api (main)
 5678efgh5678efgh5678efgh5678efgh5678efgh stickyboard-ios (main)
 9abcijkl9abcijkl9abcijkl9abcijkl9abcijkl stickyboard-android (main)
 def0mnopdef0mnopdef0mnopdef0mnopdef0mnop stickyboard-macos (main)
 1111qrst1111qrst1111qrst1111qrst1111qrst stickyboard-windows (main)
```

------

### 5. Cloning and Updating

To clone everything (umbrella + submodules):

```bash
git clone --recurse-submodules https://github.com/a3emond/stickyboard.git
```

To pull updates from all submodules:

```bash
git pull --recurse-submodules
git submodule update --remote --merge
```

------

### 6. Working with Submodules

#### Pull latest changes from one submodule:

```bash
cd stickyboard-api
git pull origin main
cd ..
git add stickyboard-api
git commit -m "Update API submodule to latest commit"
git push
```

#### Add a new platform in the future:

```bash
git submodule add https://github.com/a3emond/stickyboard-web.git stickyboard-web
git commit -m "Add StickyBoard Web client"
git push
```

#### Remove a submodule:

```bash
git submodule deinit -f stickyboard-ios
git rm -f stickyboard-ios
rm -rf .git/modules/stickyboard-ios
git commit -m "Remove iOS submodule"
git push
```

------

## Best Practices

- Each subrepo maintains its own CI/CD pipeline.
- The umbrella repo contains project-wide documentation and coordination tools.
- Always use `--recurse-submodules` when cloning or pulling to ensure all linked repos are up to date.
- Keep `.gitmodules` synchronized with actual folder names.
- Commit submodule pointer updates whenever one of the linked repos is updated.

------

## Example Workflow

1. Work on your API in `stickyboard-api/` and push changes there.

2. Update umbrella repo to point to latest API commit:

   ```bash
   git add stickyboard-api
   git commit -m "Update API version reference"
   git push
   ```

3. Continue work on a specific client (e.g., `stickyboard-android/`) independently.

4. The umbrella repo can serve as a release anchor — every tag here can correspond to a synchronized ecosystem release.

------

## Summary

| Aspect           | Responsibility                                               |
| ---------------- | ------------------------------------------------------------ |
| Individual Repos | Platform-specific development, builds, and CI/CD             |
| Umbrella Repo    | Documentation, coordination, version tracking, shared assets |
| Submodules       | Bind each repo to a fixed version for consistency            |
| Tags/Releases    | Create synchronized project milestones                       |

------

Maintained by: Alexandre Emond
 Primary Repo: https:/