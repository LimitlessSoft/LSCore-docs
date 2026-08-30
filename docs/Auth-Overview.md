---
layout: default
title: Auth Overview
nav_order: 4
has_children: true
---

# LSCore Auth System Overview

The LSCore Auth system provides a modular, middleware-based authentication and authorization framework for ASP.NET Core applications. It is designed around clean separation of concerns, with each auth mechanism packaged independently so you only include what you need.

## What the Auth System Provides

- **Authentication** -- verifying the identity of a caller (Key auth, Username/Password with JWT)
- **Authorization** -- verifying that an authenticated caller has the right to perform an action (Role-based, Permission-based)
- A shared **auth context** (`LSCoreAuthContextEntity<TEntityIdentifier>`) that flows through the request, giving downstream code typed access to the authenticated identity
- Consistent error handling via `LSCoreUnauthenticatedException` (401) and `LSCoreForbiddenException` (403)

## The Four Auth Mechanisms

### 1. Key Authentication

API-key-based authentication. Callers pass a key in the `X-LS-Key` HTTP header. The server validates it through a provider you implement. Key-authenticated requests bypass role and permission checks entirely.

**Package:** `LSCore.Auth.Key.DependencyInjection`

### 2. Username/Password Authentication (UserPass)

Classic credential-based authentication backed by JWT tokens. Callers authenticate with a username and password; the system returns an access token and a refresh token. Subsequent requests carry the JWT in the standard `Authorization: Bearer <token>` header.

**Package:** `LSCore.Auth.UserPass.DependencyInjection`

### 3. Role-Based Authorization (Role)

Adds role checks on top of an existing authentication mechanism. You define roles as an enum, assign a role to each user entity, and decorate endpoints with `LSCoreAuthRoleAttribute<TRole>`. The middleware ensures the caller has one of the required roles.

**Package:** `LSCore.Auth.Role.DependencyInjection`

### 4. Permission-Based Authorization (Permission)

Adds granular permission checks on top of an existing authentication mechanism. You define permissions as an enum, assign a collection of permissions to each user entity, and decorate endpoints with `LSCoreAuthPermissionAttribute<TPermission>`. The middleware ensures the caller has the required permissions (all or any, configurable per endpoint).

**Package:** `LSCore.Auth.Permission.DependencyInjection`

## How They Combine

The four mechanisms are independent NuGet packages that compose through the ASP.NET Core middleware pipeline. Common combinations:

- **Key only** -- Machine-to-machine APIs where a shared secret is sufficient.
- **UserPass only** -- User-facing APIs where you need login/logout but no fine-grained authorization.
- **UserPass + Role** -- User-facing APIs where different user types have different access levels (e.g., Admin vs. Member).
- **UserPass + Permission** -- User-facing APIs where access is controlled by a specific set of granular permissions.
- **UserPass + Role + Permission** -- Full authorization stack with both coarse (role) and fine-grained (permission) checks.
- **Key + UserPass + Role + Permission** -- All four. Key-authenticated callers bypass role and permission checks, allowing service accounts full access alongside regular user authorization.

## Architecture Diagram

```
                          Incoming HTTP Request
                                  |
                                  v
                   +------------------------------+
                   |   LSCoreAuthKeyMiddleware     |  <-- Checks X-LS-Key header
                   |   (if Key auth registered)    |      Sets ClaimsPrincipal
                   +------------------------------+
                                  |
                                  v
                   +------------------------------+
                   |   ASP.NET JWT Bearer Auth     |  <-- Validates Authorization
                   |   (if UserPass registered)    |      header / Bearer token
                   +------------------------------+
                                  |
                                  v
                   +------------------------------+
                   | LSCoreAuthUserPassMiddleware  |  <-- Populates auth context
                   |   (if UserPass registered)    |      (LSCoreAuthContextEntity)
                   +------------------------------+
                                  |
                                  v
                   +------------------------------+
                   |  LSCoreAuthRoleMiddleware     |  <-- Checks role attribute
                   |   (if Role registered)        |      on endpoint
                   +------------------------------+
                                  |
                                  v
                   +------------------------------+
                   | LSCoreAuthPermissionMiddleware|  <-- Checks permission
                   |   (if Permission registered)  |      attribute on endpoint
                   +------------------------------+
                                  |
                                  v
                          Endpoint Handler
```

## Quick Comparison

| Mechanism  | Purpose          | Identity Source          | Granularity    | Requires UserPass? |
|------------|------------------|--------------------------|----------------|---------------------|
| Key        | Authentication   | `X-LS-Key` header        | All-or-nothing | No                  |
| UserPass   | Authentication   | Username + Password (JWT)| Per-user       | N/A (is UserPass)   |
| Role       | Authorization    | User entity role field   | Per-role       | Yes                 |
| Permission | Authorization    | User entity permissions  | Per-permission | Yes                 |

## Package Dependency Graph

```
LSCore.Auth.Contracts                        (shared types: ILSCoreAuthEntity, LSCoreAuthAttribute, etc.)
    |
    +-- LSCore.Auth.Key.Contracts            (ILSCoreAuthKeyProvider, headers, configuration)
    |       |
    |       +-- LSCore.Auth.Key.DependencyInjection   (middleware, DI extensions)
    |
    +-- LSCore.Auth.UserPass.Contracts       (ILSCoreAuthUserPassEntity, configuration, manager interface)
    |       |
    |       +-- LSCore.Auth.UserPass.Domain           (manager implementation, password helpers)
    |       +-- LSCore.Auth.UserPass.DependencyInjection   (middleware, DI extensions)
    |
    +-- LSCore.Auth.Role.Contracts           (ILSCoreAuthRoleEntity, attribute, manager interface)
    |       |
    |       +-- LSCore.Auth.Role.Domain               (manager implementation)
    |       +-- LSCore.Auth.Role.DependencyInjection   (middleware, DI extensions)
    |
    +-- LSCore.Auth.Permission.Contracts     (ILSCoreAuthPermissionEntity, attribute, manager interface)
            |
            +-- LSCore.Auth.Permission.Domain          (manager implementation)
            +-- LSCore.Auth.Permission.DependencyInjection  (middleware, DI extensions)
```

## NuGet Packages (v10.0.0)

All packages target **.NET 10.0**.

| Package | Description |
|---------|-------------|
| `LSCore.Auth.Contracts` | Shared contracts: `ILSCoreAuthEntity`, `LSCoreAuthAttribute`, `LSCoreAuthContextEntity`, `LSCoreJwt`, `LSCoreAuthEntityType` |
| `LSCore.Auth.Key.Contracts` | Key auth contracts: `ILSCoreAuthKeyProvider`, `LSCoreAuthKeyConfiguration`, `LSCoreAuthKeyHeaders` |
| `LSCore.Auth.Key.DependencyInjection` | Key auth middleware and DI registration |
| `LSCore.Auth.UserPass.Contracts` | UserPass auth contracts: `ILSCoreAuthUserPassEntity`, `LSCoreAuthUserPassConfiguration`, manager interface |
| `LSCore.Auth.UserPass.Domain` | UserPass manager implementation, BCrypt password helpers |
| `LSCore.Auth.UserPass.DependencyInjection` | UserPass middleware and DI registration (includes JWT Bearer setup) |
| `LSCore.Auth.Role.Contracts` | Role auth contracts: `ILSCoreAuthRoleEntity`, `LSCoreAuthRoleAttribute` |
| `LSCore.Auth.Role.Domain` | Role manager implementation |
| `LSCore.Auth.Role.DependencyInjection` | Role middleware and DI registration |
| `LSCore.Auth.Permission.Contracts` | Permission auth contracts: `ILSCoreAuthPermissionEntity`, `LSCoreAuthPermissionAttribute` |
| `LSCore.Auth.Permission.Domain` | Permission manager implementation |
| `LSCore.Auth.Permission.DependencyInjection` | Permission middleware and DI registration |
