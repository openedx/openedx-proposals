OEP-68: Multi-Tenant Support in Open edX Mobile Apps
####################################################

 .. list-table::
   :widths: 25 75

   * - OEP
     - OEP-68 Multi-Tenant Mobile Support
   * - Title
     - Multi-Tenant Support in Open edX Mobile Apps
   * - Last Modified
     - 2026-08-28
   * - Authors
     -
       * Rawan Matar <rawan@zeitlabs.com>
       * Ivan Stepanok <ivan.stepanok@raccoongang.com>

   * - Arbiter
     - TBD
   * - Status
     - Draft
   * - Type
     - Architecture Decision
   * - Created
     - 2025-03-18
   * - Review Period
     - TBD
   * - References
     -
       * `Multi-tenant PR <https://github.com/zeit-labs/openedx-app-ios-contrib/pull/10>`_

Abstract
********
This OEP proposes adding **multi-tenant support** to the Open edX mobile applications.
Currently, the apps support only a single institution per build.
This proposal introduces a mechanism for users to select their institution (tenant) at runtime,
with each tenant providing its own branding, configuration, and API endpoints.
This reduces duplication and enables a white-labeling approach
within a single mobile app module.

Motivation
**********
The Open edX mobile app is increasingly used by multiple institutions.
Releasing separate app builds for each tenant creates duplication, higher maintenance burden,
and inconsistent user experience. Multi-tenancy enables:

* A single app build for multiple institutions.
* Dynamic branding (logo, color scheme, names).
* Institution-specific API and authentication endpoints.
* Future extensibility to fetch tenant definitions dynamically.
* Preserves current behavior for single-tenant deployments — if only one tenant is configured, the app works exactly as it does today, with no selector or added flow.

Current State
=============
- Mobile apps support only one tenant per build (YAML-based config).
- Persistence layer uses a single Core Data/SQLite DB, risking data leakage across tenants.
- Push notification manager doesn't account for tenant context.
- Branding and login flows are static per build.

Decision
********
We propose supporting **multi-tenancy** in one app build by:

* Providing a tenant selector screen at login, and a separate switcher reachable
  post-login from Settings.
* Sourcing tenant metadata (branding, API URLs, auth URLs, etc.) from a remote JSON
  endpoint fetched at launch, cached locally.
* Allowing multiple tenants to hold valid, concurrent sessions on the same device, with
  runtime switching between them that doesn't require reinstallation or re-login for an
  already-signed-in tenant.
* Isolating tenant data within a single shared persistence store, scoped by tenant key,
  rather than maintaining separate database instances per tenant.
* Enhancing push notifications with tenant context.
* Preserving current behavior for single-tenant deployments — if only one tenant is
  configured, the selector/switcher is skipped and the app works exactly as it does
  today.

Specification
*************

Tenant Selection
================
- On launch, show a tenant selector if multiple tenants are configured.
- If only one tenant exists, preserve the current flow.
- A tenant switcher is also reachable post-login, from Settings, without going through
  the launch screen — selecting an already-signed-in tenant there goes straight to its
  home screen (see Concurrent Sessions); selecting a tenant with no session shows login
  for it.

Concurrent Sessions
====================
- Selecting or switching to a tenant does not require re-authenticating if that tenant
  already has a valid session; more than one tenant can hold a valid, concurrently
  active session on the same device at the same time.
- A per-tenant "has a session" indicator (shown in the tenant list/switcher) reflects
  whether that tenant's credentials are currently present, independent of which tenant
  is the *active* one.
- Switching the active tenant never clears another tenant's session or data.
- A per-tenant configuration flag (``is_switch_tenant_login_enabled``) can force login
  even when a session already exists for that tenant, for tenants that want
  re-authentication on every switch regardless of session state.
- A forced logout (e.g. an expired/invalid token) clears that tenant's session, cached
  data. A voluntary logout (Settings) clears the session but currently leaves downloaded course content in place.

Tenant Configuration
====================
Tenant metadata is sourced as JSON, but not from a single static file bundled with the
app. At launch, the app:

1. Fetches the tenant catalog from a fixed, hardcoded remote JSON endpoint (a public
   gist), always live-first so that branding/theme edits published there take effect on
   the next launch.
2. Caches the fetched catalog locally on success.
3. Falls back to the last successfully cached catalog if the live fetch fails (no
   network, non-2xx response, malformed JSON, timeout).

Tenant fields (the remote JSON uses lower snake_case; the bundled YAML equivalent uses
upper snake_case — both are accepted and normalized to the same model):

* ``name`` — tenant identifier.
* ``tenant_name`` — localized display name (e.g. ``{"en": ..., "ar": ...}``).
* ``oauth_client_id`` — required; the tenant's OAuth client ID.
* ``api_host_url`` (plus a separate hidden-login-flow variant) — the tenant's API host.
* ``sso_url`` (plus a separate successful-SSO-login variant).
* ``environment_display_name``.
* ``is_switch_tenant_login_enabled`` — forces login on switch even with an existing
  session (see Concurrent Sessions).
* ``ui_components`` — per-tenant UI feature toggles.
* ``logo_url`` — an ``http(s)://`` URL for a remote logo, or a bundled asset name to use
  a compiled-in image instead.
* ``header_background_url`` — remote only, no bundled-asset fallback.
* ``theme`` — optional ``{ "light": {...}, "dark": {...} }`` blocks of per-color hex
  overrides; falls back to deriving light/dark automatically from ``color`` when absent.

Example (remote schema)::

  {
    "tenants": [
      {
        "name": "tenant_a",
        "tenant_name": { "en": "University A", "ar": "..." },
        "oauth_client_id": "tenant-a-client-id",
        "api_host_url": "https://tenant_a.example.com",
        "sso_url": "https://tenant_a.example.com/sso",
        "logo_url": "https://example.com/logo.png",
        "is_switch_tenant_login_enabled": false,
        "theme": {
          "light": { "accent_color": "#FF5733" },
          "dark": { "accent_color": "#FF7A50" }
        }
      }
    ]
  }

Database Management
===================
- All tenants share a single Core Data/SQLite persistent store; there is no separate database instance or file per tenant, and no ``opendx_{tenant_id}_db``-style naming.
- Isolation is enforced by scoping every persisted row/key to a ``tenantKey`` — reads, writes, and delete predicates are namespaced rather than routed to a separate store. Keychain and on-disk downloads follow the same tenant-key-namespaced pattern.
- Switching the active tenant does not reload or rebuild the persistence stack; it changes which ``tenantKey`` subsequent reads/writes are scoped to.

Push Notifications
==================
- One device token is maintained, and it is the same Firebase project's token for the whole app — there is no per-tenant Firebase project; all tenants currently share one.
- The token is (re)registered against every tenant the device currently has an active session with, not just the current tenant, so notifications can reach a signed-in tenant even while another tenant is active in the foreground.
- Unregistration on a forced (session-expired) logout is implemented for the active tenant; unregistration on a voluntary logout is not yet wired up.
- **Payloads do not currently include a tenant identifier.** No field in the push payload identifies which tenant a notification originated from, so the client cannot yet route an incoming notification to the correct tenant, prompt a "switch tenant?" confirmation, or resolve a deep link against the right backend.
- This is an open gap, not a designed-but-unbuilt detail: closing it requires a backend/payload change (e.g. adding a ``tenant_id`` field to the payload) before any client-side tenant-routing logic can be built. It also requires each tenant's own backend to implement the token-registration endpoint, including an unregister (``active: false``) case.

Tradeoffs
=========
We considered three approaches:

**1. Single App + Bundled Config (proposed)**

- **Pros**:
  - Single codebase and release process, reducing duplication.
  - Works offline immediately since tenant definitions are bundled.
  - Reliable theming and branding without network dependency.
- **Cons**:
  - Updating tenant metadata used to require an app update — no longer true; see "As implemented" below.
  - Larger app size if many assets are included.
- **Implications**:
  - **Push notifications**: One device token, re-registered per tenant with an active session; payload-based routing by ``tenant_id`` is not yet implemented (see Push Notifications).
  - **Theming**: Strong, since assets/colors are local.
  - **Offline**: Strong support — tenants always available.

**2. Remote Configuration**

- **Pros**:
  - Tenant metadata can change without an app update.
  - Centralized management of tenant definitions.
- **Cons**:
  - Requires robust caching and fallbacks to work offline.
  - Higher runtime complexity and security considerations.
- **Implications**:
  - **Push notifications**: Requires re-registration if tenant config changes.
  - **Theming**: More flexible, but must be cached to avoid broken UI offline.
  - **Offline**: Dependent on last successful fetch.

**3. Separate Builds (current state)**

- **Pros**:
  - Full isolation per tenant (bundle IDs, push notifications, store presence).
  - Simple runtime logic.
- **Cons**:
  - High maintenance and CI/CD overhead.
  - Risk of divergence across builds.
- **Implications**:
  - **Push notifications**: Simplest — separate certs per build.
  - **Theming**: Static and guaranteed.
  - **Offline**: Guaranteed, but at the cost of scalability.

**As implemented**: the shipped design is a hybrid of Option 1 and Option 2 above, not
pure Option 1. On every launch the app fetches the tenant catalog from a remote JSON
endpoint first (Option 2's behavior), caches it, and only falls back to a bundled
``config.yaml`` tenant list (Option 1's behavior) if the live fetch and the cache both
fail. This keeps Option 1's offline guarantee as a fallback while gaining Option 2's
ability to update branding/theme without an app release, at the cost of Option 2's added
runtime complexity (a network call and cache/fallback chain on every launch).

Impact on Single-Tenant Deployments
===================================
For single-tenant deployments, this proposal introduces minimal change:

- If only one tenant is defined in the configuration, the tenant selector is skipped and the app behaves exactly as today.
- Branding, theming, and login flows remain static to that single tenant.
- No additional runtime overhead is introduced for single-tenant cases.

Alternatives
************
- Separate app per tenant (high maintenance).
- Remote configuration (scalable but adds backend dependency).
- Custom login screen only (limited branding flexibility).

Impact
******
- **Users**: Access multiple institutions in one app.
- **Institutions**: Branded experiences without custom builds.
- **Developers**: Reduced duplication and simpler long-term maintenance.
- **Backend**: Each tenant's own backend must support the app's push-token registration endpoint, including an unregister (``active: false``) case. Including a ``tenant_id`` (or similar) field in push notification payloads is required before tenant-aware push routing can be built client-side — not yet done.

Future Work
***********
Implemented since the original draft:

* Fetch tenant metadata dynamically from an API — done (remote-first with cached/bundled fallback; see Tenant Configuration).
* Allow tenant switching post-login via settings/profile — done (see Tenant Selection, Concurrent Sessions).

Still open:

* Adopt design tokens for consistent theming across platforms.
* Add a tenant identifier to push notification payloads, so an incoming push can be routed to the correct tenant and a "switch tenant?" confirmation (with deep-link completion) can be shown on tap — blocked on a backend/payload change.
* Unregister the push token.
* Background sync across all signed-in tenants, and a "manage sessions" view to log out of a tenant that isn't the one currently being viewed.
* Per-tenant Firebase projects, if a tenant ever needs isolated push/analytics infrastructure instead of the current single shared project.

Change History
**************
2025-03-18
==========
* Initial draft created based on ADR.

2025-09-23
==========
* Added Tradeoffs and Single-Tenant Impact sections.

2026-08-28
==========
* Revised "Decision" and "Database Management" to reflect the shipped implementation: tenant data is isolated within a single shared, tenant-key-scoped persistence store rather than separate per-tenant database instances.
* Revised "Push Notifications" and the bundled-config tradeoff to reflect that push payloads do not yet carry a tenant identifier — server-side routing by ``tenant_id`` is an open item blocked on a backend/payload change, not a fully implemented feature.
* Revised "Tenant Configuration" to reflect that tenant metadata is fetched from a remote JSON endpoint first (cached, with a bundled ``config.yaml`` fallback), not from a single static bundled JSON file, and updated the field list/example to match the real schema (``oauth_client_id``, ``theme`` light/dark blocks, etc.).
* Added a "Concurrent Sessions" subsection documenting that multiple tenants can hold valid sessions simultaneously, and that switching tenants no longer tears down the outgoing tenant's session/data (only an explicit logout does).
* Added a note to "Tenant Selection" that a tenant switcher is also reachable post-login from Settings.
* Added an "As implemented" note to "Tradeoffs" clarifying the shipped design is a hybrid of the bundled-config and remote-configuration options, not pure Option 1.
* Expanded the "Impact" section's Backend bullet with concrete backend requirements (per-tenant token-registration endpoint support, tenant identifier in payloads).
* Moved two completed items ("fetch tenant metadata dynamically", "post-login tenant switching") from "Future Work" to done, and replaced the remaining Future Work list with the concrete open items identified during implementation (push tenant-routing, voluntary-logout unregister, background multi-tenant sync, deep-link preservation, per-tenant Firebase).