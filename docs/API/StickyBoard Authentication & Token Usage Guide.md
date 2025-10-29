# StickyBoard Authentication & Token Usage Guide

### **Overview**

StickyBoard uses a **two-tier token system** for authentication and session management.
 This design provides strong security, offline continuity, and seamless user experience across all clients (web, desktop, and mobile).

| Token Type               | Typical Lifetime | Purpose                                        |
| ------------------------ | ---------------- | ---------------------------------------------- |
| **Access Token (JWT)**   | ~10–15 minutes   | Grants access to protected API routes          |
| **Refresh Token (UUID)** | ~7 days          | Renews access tokens silently without re-login |

Access tokens are **short-lived** to minimize exposure.
 Refresh tokens are **securely stored and rotated** to maintain long sessions.

------

## 1. Login and Initial Session Setup

**Endpoint:** `POST /api/auth/login`

**Response:**

```
{
  "token": "<ACCESS_TOKEN>",
  "refreshToken": "<REFRESH_TOKEN>",
  "user": { "id": "GUID", "email": "user@domain.com", "displayName": "User" }
}
```

**Client behavior:**

- Save `refreshToken` securely (e.g., keychain, secure storage, encrypted DB).

- Store `token` **in memory only** (not on disk) and attach it to all API requests as:

  ```
  Authorization: Bearer <token>
  ```

- Initialize a timer or event system to track token expiration (see below).

------

## 2. Access Token Lifespan and Auto-Refresh Strategy

Because access tokens expire quickly (≈15 min), the client must **refresh them proactively** to maintain a seamless session.

### **Recommended Strategy**

There are two complementary triggers:

#### a) **Activity-based Refresh**

Every time the user performs an action that calls a protected endpoint (e.g., saving a note, editing a board):

1. The client checks how old the current access token is.

2. If it’s near expiration (e.g., within 2 minutes of expiry), call:

   ```
   POST /api/auth/refresh
   {
     "refreshToken": "<stored_refresh_token>"
   }
   ```

3. Replace both tokens with the ones returned in the response.

This keeps sessions active **as long as the user is active** — like a “sliding session window”.

#### b) **Error-based Refresh**

If any API call returns `401 Unauthorized`, it means:

- The access token has expired **and**
- The refresh token is still valid.

Then:

1. Attempt `/api/auth/refresh` immediately.
2. If successful, retry the failed request with the new access token.
3. If it fails again → force re-login.

This ensures resilience even if timing or network issues occur.

------

## 3. Refresh Token Rotation

**Endpoint:** `POST /api/auth/refresh`

**Request:**

```
{ "refreshToken": "<stored_refresh_token>" }
```

**Response:**

```
{
  "token": "<new_access_token>",
  "refreshToken": "<new_refresh_token>",
  "user": { ... }
}
```

**Important:**

- Each refresh token can be used **only once**.
- Always **replace** your stored refresh token with the new one.
- If you lose or reuse an old refresh token, it becomes invalid.

------

## 4. Logout and Revocation

**Endpoint:** `POST /api/auth/logout`

- Requires Authorization header (`Bearer <access_token>`).
- Revokes **all refresh tokens** linked to the user.
- Effect: the user is logged out on all devices; all sessions become invalid for refresh.

**Client behavior:**

- Delete both `accessToken` and `refreshToken`.
- Redirect user to the login screen.

------

## 5. Token Renewal and User Activity Model

To balance security and usability, StickyBoard clients should integrate token refreshing with **activity monitoring**:

### a) **Activity Tracking**

Every meaningful user event (keyboard, mouse, scroll, tap, navigation) updates a local “lastActive” timestamp.

### b) **Background Refresh Logic**

Every few minutes (e.g., 2–5 min), the client checks:

```
if (Now - lastActive < 10 minutes)
    and (AccessToken expires in < 2 minutes):
        call /api/auth/refresh
```

This way:

- Active users stay authenticated indefinitely (refresh tokens rotate transparently).
- Inactive users naturally expire after ~7 days (when the refresh token expires).

### c) **Offline Mode**

When offline, access tokens cannot be renewed.
 Clients can continue using cached data locally and retry `/auth/refresh` when connectivity returns.

------

## 6. Token Storage Recommendations

| Environment                | Access Token                                       | Refresh Token                             |
| -------------------------- | -------------------------------------------------- | ----------------------------------------- |
| **Web**                    | In-memory variable (never localStorage)            | HttpOnly cookie (if using secure session) |
| **Mobile / Desktop**       | Secure storage (Keychain, Android Keystore, DPAPI) | Same secure store                         |
| **CLI / Headless clients** | Encrypted local file or key vault                  | Same, restricted access                   |

**Never log, serialize, or expose tokens** in error traces or analytics.

------

## 7. Example Client Workflow (Pseudocode)

```
async function authorizedFetch(url, options) {
  const now = Date.now();

  // Refresh if token close to expiry
  if (accessTokenExp - now < 120000) {
    const refreshed = await refreshSession();
    if (!refreshed) {
      logout();
      return;
    }
  }

  // Execute the request
  const res = await fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      Authorization: `Bearer ${accessToken}`,
    },
  });

  // Retry on 401
  if (res.status === 401) {
    const refreshed = await refreshSession();
    if (refreshed) {
      return authorizedFetch(url, options);
    } else {
      logout();
    }
  }

  return res;
}

async function refreshSession() {
  const res = await fetch("/api/auth/refresh", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ refreshToken }),
  });
  if (!res.ok) return false;

  const data = await res.json();
  accessToken = data.token;
  refreshToken = data.refreshToken;
  accessTokenExp = decodeJwt(data.token).exp * 1000;
  return true;
}
```

------

## 8. Token Lifecycle Summary

```
[Login/Register] 
     ↓
[Access Token (15m)] — used for API calls
     ↓ (expires)
[Refresh Token (7d)] — renews session if user active
     ↓ (revoked or expired)
[Logout or Re-login required]
```

------

## Key Takeaways

- **Access tokens** are short-lived and never reused beyond their lifespan.
- **Refresh tokens** are securely stored and rotated on each use.
- **Client logic** must proactively refresh tokens based on:
  - Time-to-expiry
  - User activity
  - 401 Unauthorized responses
- **Logout** revokes all active sessions.
- The design ensures users stay signed in as long as they’re active — and automatically signed out after inactivity.