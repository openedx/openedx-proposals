.. _OEP-69 Open edX REST API Conventions:

OEP-69: Open edX REST API Conventions
#####################################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-69 <OEP-69 Open edX REST API Conventions>`
   * - Title
     - Open edX REST API Conventions
   * - Last Modified
     - 2026-07-06
   * - Authors
     - Abdul-Muqadim (abdul.muqadim@arbisoft.com), Taimoor Ahmed (taimoor.ahmed@arbisoft.com), Muhammad Faraz Maqsood (faraz.maqsood@arbisoft.com)
   * - Arbiter
     - Feanil Patel (feanil@axim.org)
   * - Status
     - Draft
   * - Type
     - Best Practice
   * - Created
     - 2026-07-05
   * - Review Period
     - <start - target end dates for review>
   * - Resolution
     - <link to the discussion where the final status is decided>
   * - References
     -
       * `Request for community review on ADRs for standardizing the Open edX platform API endpoints <https://discuss.openedx.org/t/request-for-community-review-on-adrs-for-standardizing-the-open-edx-platform-api-endpoints/18717>`_
       * `FC-0118 epic issue #38137: "Add ADRs for technical Recommendations on Open edX REST API Standards" <https://github.com/openedx/openedx-platform/issues/38137>`_
       * `openedx-platform ADRs 0025–0037 (docs/decisions) <https://github.com/openedx/openedx-platform/commit/36a192d980faada26a423905f2a64c471928761b>`_
       * `ADR 0038: REST API URL Structure (PR #39003) <https://github.com/openedx/openedx-platform/pull/39003>`_
       * :ref:`OEP-4 Application Authorization (Scopes)`
       * :ref:`OEP-42 Authentication`
       * :ref:`OEP-66 User Authorization`
       * :ref:`OEP-65 Frontend Composability <OEP-65 Frontend Composibility>`
       * :ref:`OEP-68 Content Identifiers <OEP-68 Learning Content Identifiers>`
       * :ref:`OEP-21 Deprecation and Removal <OEP-21 DEPR>`

Abstract
********

This OEP consolidates a set of Architecture Decision Records (ADRs) developed
under Funded Contribution **FC-0118** into a single, authoritative set of REST
API conventions for the Open edX platform. It establishes clear, shared patterns
for how HTTP APIs should be built going forward, and describes how existing
endpoints should be brought in line incrementally.

The conventions align the platform more closely with the Django REST Framework
(DRF) and with widely accepted REST and OpenAPI best practices, covering
serializers, view structure, authentication, authorization, pagination,
filtering and sorting, response shaping, error responses, documentation,
versioning, and endpoint topology.

The individual ADRs live in the `openedx-platform docs/decisions folder
<https://github.com/openedx/openedx-platform/tree/master/docs/decisions>`__ (``0025`` through ``0038``) as the detailed, code-adjacent record of
each decision; every convention below links directly to its source ADR and to that
ADR's code example. This OEP is the consolidated, cross-cutting reference (the "one big, clear REST API Conventions"
document requested during community review) and is the canonical statement of the
conventions for the Open edX community.

Motivation
**********

This effort proposes to standardize Open edX APIs in a practical and incremental
way. The intent is **not** to redesign the entire platform or introduce breaking
changes unnecessarily, but to establish clear, shared patterns for how APIs should
be built going forward, while gradually improving existing endpoints. By aligning
more closely with DRF and with widely accepted REST and OpenAPI best practices, the
platform can offer a more consistent and reliable API experience. In addition, it
will lower integration costs for external systems, make APIs reusable, and provide
a stronger foundation for future growth.

Beyond the general improvement of the codebase and the developer experience (DX),
one of the goals of this project is to facilitate the integration of Open edX with
agentic generative AI: by standardizing API endpoints, enforcing strict schema
consistency, and providing thorough API documentation, we greatly ease the creation
of remote tools that can then be used by AI agents. In particular, it becomes much
easier to expose the platform through machine-discoverable tools (for example, MCP
servers and tools) that operate reliably against predictable, well-documented
endpoints.

The current state of the platform's REST surface works against these goals:

* **Inconsistent construction.** Many endpoints manually build JSON responses
  instead of using serializers, fragment a single resource across several views
  instead of using ViewSets, and re-implement validation, permission, and logging
  boilerplate per view.
* **Inconsistent contracts.** Pagination, filtering, sorting, error shapes, and
  authentication vary from app to app, forcing every consumer to write
  endpoint-specific logic.
* **Poor discoverability.** Documentation is incomplete and, where it exists, is
  produced through an OpenAPI 2.0 toolchain that is no longer a viable long-term
  home. This blocks SDK generation, contract testing, and reliable machine
  consumption.
* **Unclear evolution.** Versioned and unversioned paths coexist with no
  consistent deprecation contract, so clients cannot tell which endpoint is
  authoritative or how long it will last.

These problems are individually addressable, but they were decided separately as a
series of ADRs. Reviewers noted that it had become "a bit confusing as to what info
is found in which ADR" and asked for the decisions to be consolidated into one clear
"REST API Conventions" document. This OEP is that document.

Specification
*************

The conventions below are the normative output of this OEP. Each convention names
its source ADR in ``openedx-platform`` for the fully detailed rationale, code
references, and rollout notes. The key words **MUST**, **MUST NOT**, **SHOULD**,
and **MAY** are used in their conventional (RFC 2119) sense.

All conventions share three cross-cutting principles:

* **Incremental, backward-compatible migration.** New code follows these
  conventions immediately. Existing endpoints are migrated over time. Where a
  change would be backward-incompatible, it **MUST** be delivered as a new API version
  and the old version deprecated per :ref:`OEP-21 Deprecation and Removal <OEP-21 DEPR>`.
* **DRF as the integration surface.** These conventions standardize how endpoints
  are expressed in DRF; they do not mandate a specific business-logic or policy
  engine behind that surface.
* **Machine-first contracts.** Predictable, documented, schema-covered contracts
  are treated as a first-class requirement so that external systems and AI agents
  can consume the platform reliably.

Structure and framework
=======================

Convention 1: Use DRF serializers for all request and response handling
-----------------------------------------------------------------------

*Source:* `ADR 0025: Standardize Serializer Usage Across APIs <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0025-standardize-serializer-usage.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0025-standardize-serializer-usage.rst#code-example-target-serializer-usage>`__).

All Open edX REST APIs **MUST** use DRF serializers for request and response handling
rather than hand-constructing JSON.

* API views **MUST** define explicit serializers for request and response
  handling, and use them for both input validation and output formatting.
* Manual JSON construction (``dict`` building, ``HttpResponse(json.dumps(...))``)
  **MUST** be replaced with serializer-based responses.
* Serializers **MUST** be documented with field ``help_text`` and validation
  rules.
* Input and output serializers are often different: the request body may accept
  only a subset of fields, while the response includes computed or read-only
  fields the caller cannot set. When they differ, define them as separate
  serializers. ``serializer_class`` (or ``get_serializer_class``) supplies only a
  single **default**, which ``drf-spectacular`` applies to *both* the request and
  response schema, so on its own it documents at most one side correctly. Any
  action whose request and response differ **MUST** therefore declare both
  explicitly with ``@extend_schema(request=..., responses=...)`` (Convention 11);
  ``serializer_class`` is just the fallback for the case where one serializer
  serves both.

.. code-block:: python

   class CourseEnrollmentInputSerializer(serializers.Serializer):
       """Validates the request body for enrollment creation."""
       course_id = serializers.CharField(help_text="The course to enroll in.")
       mode = serializers.CharField(default="audit", help_text="Enrollment mode.")

   class CourseEnrollmentOutputSerializer(serializers.Serializer):
       """Shapes the response: includes read-only fields not accepted on input."""
       course_id = serializers.CharField(help_text="The enrolled course.")
       mode = serializers.CharField(help_text="Active enrollment mode.")
       is_active = serializers.BooleanField(help_text="Whether the enrollment is active.")
       created = serializers.DateTimeField(help_text="Enrollment creation timestamp.")

   class CourseEnrollmentViewSet(viewsets.ViewSet):
       # Fallback default: used by actions (e.g. retrieve) whose response is this
       # serializer and which take no request body; drf-spectacular reads it when
       # @extend_schema does not override.
       serializer_class = CourseEnrollmentOutputSerializer

       def retrieve(self, request, pk=None):
           enrollment = _get_enrollment(request.user, pk)      # uses the fallback
           return Response(CourseEnrollmentOutputSerializer(enrollment).data)

       @extend_schema(                                          # request != response
           request=CourseEnrollmentInputSerializer,
           responses={201: CourseEnrollmentOutputSerializer},
       )
       def create(self, request):
           input_serializer = CourseEnrollmentInputSerializer(data=request.data)
           input_serializer.is_valid(raise_exception=True)
           enrollment = _enroll_user(request.user, **input_serializer.validated_data)
           return Response(CourseEnrollmentOutputSerializer(enrollment).data, status=201)

Convention 2: Consolidate related endpoints into DRF ViewSets
-------------------------------------------------------------

*Source:* `ADR 0028: Migrating RESTful & Legacy Django API Endpoints to Standard DRF ViewSets <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0028-migrate-restful-and-legacy-api-endpoints-using-standard-drf-viewsets.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0028-migrate-restful-and-legacy-api-endpoints-using-standard-drf-viewsets.rst#illustrative-example>`__).

Related actions on a resource **MUST** be consolidated into a single DRF ViewSet rather
than fragmented across separate class-based or function-based views per HTTP action.

* Related actions (``list``, ``retrieve``, ``create``, ``update``, ``destroy``)
  **MUST** be consolidated into a single ViewSet per resource, registered via a DRF
  Router for consistent, predictable URLs.
* All ViewSets **MUST** use explicit serializers (Convention 1) for both request
  validation and response formatting.
* ViewSets **MUST** optimize ``get_queryset`` with ``select_related`` /
  ``prefetch_related`` to match the fields their serializers require, preventing
  N+1 regressions. Migrations **SHOULD** include a comparison of SQL query counts.
* Multi-method handler functions (a single function dispatching DELETE/POST/PUT)
  **MUST** be refactored into action-specific methods.
* Legacy Django views **SHOULD** be migrated to ViewSets; ``APIView`` **SHOULD** be
  used for non-resource endpoints where a ViewSet is not a good fit.

Convention 3: Consolidate RPC-style action endpoints onto their resource
------------------------------------------------------------------------

*Source:* `ADR 0031: Merge Similar Endpoints <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0031-merge-similar-endpoints.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0031-merge-similar-endpoints.rst#code-example-target-unified-endpoint>`__).

This convention targets one anti-pattern: a family of separate, action-named
top-level URLs for a single resource (an RPC style, e.g. ``/enable_x``,
``/start_x``, ``/regenerate_x``), each re-implementing the same validation,
permission, and audit logic. These **SHOULD** be consolidated onto the resource so
clients have one integration point and the shared logic lives in one place.

This is *not* about HTTP verbs: it does not mean collapsing ``GET``/``POST`` or
list/detail into one function. Multiple verbs on a resource stay as separate methods
in one ViewSet (Convention 2), even when list and detail logic differ substantially;
the target is redundant *URLs*, not the verbs. A resource that only needs the
standard CRUD verbs is already covered by Convention 2 and out of scope here.

Following the ADR, consolidate onto a single URL that takes an ``action``/``mode``
field to distinguish the operation (the certificate cluster in the linked ADR uses
this form). Where the operations read better as distinct named routes, expose them
as DRF ``@action`` sub-routes on the resource's ViewSet instead.

* Shared validation, audit logging, and response shaping **SHOULD** move into a
  common service layer or mixin.
* Consolidation **MUST NOT** flatten the authorization model: keep a coarse access
  check at the view, and enforce each operation's specific permission in its handler
  or service.
* The consolidated endpoint **MUST** document its ``@action`` routes or ``mode``
  values (with request/response shapes) in OpenAPI (Convention 11), and legacy action
  URLs **MUST** be kept as deprecated aliases (a ``Deprecation`` header) for a
  transition window, per :ref:`OEP-21 Deprecation and Removal <OEP-21 DEPR>`.

For example, a course-run resource that grew one action URL per lifecycle operation
(each repeating the same staff-permission check and audit logging)::

   POST /api/courses/v1/publish_course_run
   POST /api/courses/v1/unpublish_course_run
   POST /api/courses/v1/archive_course_run

consolidates onto the resource, each operation a named sub-route behind one coarse
gate:

.. code-block:: python

   class CourseRunViewSet(viewsets.ViewSet):
       permission_classes = [IsCourseStaff]          # coarse gate for every action

       @action(detail=True, methods=["post"])
       def publish(self, request, pk): ...

       @action(detail=True, methods=["post"])
       def archive(self, request, pk): ...

   # POST /api/courses/v1/course_runs/{id}/publish/
   # POST /api/courses/v1/course_runs/{id}/archive/

Authentication and authorization
================================

Convention 4: Standardize authentication on JWT + session
---------------------------------------------------------

*Source:* `ADR 0034: Standardize Authentication Patterns and Security Schemes <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0034-unify-auth-oauth2-dot-v2.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0034-unify-auth-oauth2-dot-v2.rst#code-examples-authentication-patterns-by-use-case>`__), with the `ADR 0017 pointer <https://github.com/openedx/openedx-platform/blob/master/openedx/core/djangoapps/oauth_dispatch/docs/decisions/0017-standardize-authentication-patterns.rst>`__. See also :ref:`OEP-42 Authentication`.

``JwtAuthentication`` **MUST** be the standard authentication mechanism for all DRF API
endpoints that serve user-authenticated requests, with ``SessionAuthentication`` as
the accepted browser scheme (this is the platform default).

* New endpoints **MUST NOT** set ``authentication_classes`` explicitly unless
  deviating from the platform default (which supplies JWT + session). Deviations
  (e.g. service-to-service JWT-only) **MUST** be explicit and commented.
* ``BearerAuthentication`` / ``BearerAuthenticationAllowInactiveUser`` (and their
  deprecated aliases ``OAuth2Authentication`` /
  ``OAuth2AuthenticationAllowInactiveUser``) **MUST NOT** be used in new code, and
  existing endpoints **MUST** be audited to remove them. The primary migration
  target is the ``view_auth_classes`` decorator, which removes Bearer auth from 49+
  endpoints at once.
* Endpoints that must enforce active-user status **SHOULD** do so with a permission
  class, not an authentication class (JWT and ``...AllowInactiveUser`` session
  classes do not check ``user.is_active``).

.. note::

   OAuth2/DOT and JWT are not competing mechanisms: Django OAuth Toolkit issues
   JWTs as OAuth2 access tokens, validated by ``JwtAuthentication``. Only the
   ``BearerAuthentication`` family is deprecated.

Convention 5: Use DRF ``permission_classes`` as the authorization surface
-------------------------------------------------------------------------

*Source:* `ADR 0026: Standardize Permission Classes Across APIs <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0026-standardize-permission-classes.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0026-standardize-permission-classes.rst#code-example-target-permission-usage>`__). See also :ref:`OEP-66 User Authorization`.

All Open edX REST APIs **MUST** use DRF ``permission_classes`` as the primary
authorization surface, replacing custom decorators and inline role checks.

* Create reusable permission classes for common patterns (course staff, global
  staff, object ownership) and apply them consistently across similar endpoints.
* This convention standardizes the **DRF integration surface**, not the underlying
  policy engine. Permission classes **MAY** delegate to legacy checks or a newer
  policy engine (e.g. Casbin / ``openedx-authz``) and the backend **MUST** stay
  pluggable so endpoints can migrate without changing endpoint-level authorization
  structure.
* For list endpoints, endpoint access (``permission_classes``), record visibility
  (queryset scoping in ``get_queryset``), and user-driven filtering (``django-filter``)
  are **distinct concerns** and **MUST** be kept separate, per the queryset-scoping
  pattern in :ref:`OEP-66 User Authorization`.

Request and response conventions
================================

Convention 6: GET requests MUST be idempotent with respect to domain state
--------------------------------------------------------------------------

*Source:* `ADR 0030: Ensure GET is Idempotent <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0030-ensure-get-requests-are-idempotent.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0030-ensure-get-requests-are-idempotent.rst#code-example>`__).

A ``GET`` handler **MUST** be read-only with respect to **domain state**: the
transactional records that drive application behavior (enrollments, grades, profile
fields, access records).

* ``GET`` handlers **MUST NOT** create, update, or delete domain-state records.
* openedx-events and Django signals **MUST NOT** be fired from ``GET`` handlers;
  their receivers can perform downstream writes that are invisible to the handler.
  State-changing side effects **MUST** move to explicit ``POST``/``PUT``/``PATCH``
  endpoints.
* **Non-domain writes**, pure analytics (``tracker.emit``, Segment events,
  read-count increments), are **permitted** in a ``GET`` handler provided the
  response does not depend on them and no openedx-events or Django signals are
  involved.
* Regression tests **SHOULD** assert that ``GET`` handlers do not mutate domain
  state.

Convention 7: Standardize pagination on ``DefaultPagination``
-------------------------------------------------------------

*Source:* `ADR 0032: Standardize Pagination Across APIs <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0032-standardize-pagination-usage.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0032-standardize-pagination-usage.rst#code-example-target-pagination-usage>`__).

All flat list endpoints **MUST** paginate using ``DefaultPagination`` (or a subclass)
from ``edx-drf-extensions``: ``page``/``page_size`` parameters, default page size
10, maximum 100.

* Paginated responses **MUST** include the standard envelope: ``count``, ``next``,
  ``previous``, ``num_pages``, ``current_page``, ``start``, and ``results``.
* Endpoints on ``APIView`` (rather than ``GenericAPIView``/``ListAPIView``) **MUST**
  invoke the pagination API manually.
* Per-endpoint ``page_size`` overrides (e.g. smaller for mobile) are acceptable when
  justified, but **MUST** subclass ``DefaultPagination`` rather than swap in an
  unrelated class.
* Migrating ``limit``/``offset`` endpoints to ``page``/``page_size`` is a breaking
  change and **MUST** be versioned.
* **Tree-shaped endpoints** (course blocks, taxonomy, OLX, progress trees) are out
  of scope for item-count pagination and **MUST** instead follow the depth-cap or
  flat-child-list patterns in Convention 9.

Convention 8: Standardize filtering and sorting on ``django-filter``
--------------------------------------------------------------------

*Source:* `ADR 0033: Standardize Filtering/Sorting Parameters <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0033-standardize-filter-sort-using-django-filter.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0033-standardize-filter-sort-using-django-filter.rst#code-examples>`__).

List endpoints that support filtering **MUST** use ``django-filter``, with a single,
documented ``ordering`` parameter for sorting.

* Parameter names **MUST** be standardized and documented (e.g. use ``course_key``
  consistently, per :ref:`OEP-68 Content Identifiers <OEP-68 Learning Content Identifiers>`, rather than ``course`` /
  ``course_key`` variants).
* Sorting **MUST** use DRF's ``ordering`` parameter with a documented allow-list of
  fields.
* Filters and ordering **MUST** be discoverable via OpenAPI.
* When renaming parameters, support the old name as a deprecated alias, return a
  ``Deprecation`` warning header, mark it ``deprecated: true`` in the schema, and
  remove it only after a defined window.

Convention 9: Reduce deeply nested JSON via minimal / flattened views
---------------------------------------------------------------------

*Source:* `ADR 0036: Reduce Deeply Nested JSON via Minimal/Flattened Views <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0036-normalize-deeply-nested-json-apis.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0036-normalize-deeply-nested-json-apis.rst#code-example>`__).

Complex resources **MUST** offer a way to reduce payload depth and size rather than
always returning deeply nested trees.

* Provide a minimal representation option via ``?view=minimal`` and/or explicit
  field selection ``?fields=...``.
* For server-to-server and automated clients, **prefer references/IDs with
  follow-up endpoints** over embedding entire trees by default. Frontend/MFE clients
  that genuinely need a complete view in one load **MAY** request an embedded full
  representation via an explicit opt-in (``?view=full`` / ``?fields=<list>``) so the
  heavy payload is never the default.
* For **tree-shaped endpoints**, unbounded child lists **MUST** be bounded using one of:

  * **Pattern A (depth cap + child-fetch URLs):** return structure to a fixed
    ``?depth=N`` (default 2) and provide ``children_url`` for deeper subtrees.
  * **Pattern B (flat paginated child sub-resource):** expose
    ``/.../<parent_id>/children/?page=...`` following Convention 7.

* A ``?fields=...`` request that would expand a large child collection **MUST NOT**
  silently return everything unbounded; returning HTTP 400 is preferable to
  truncating or emitting an oversized payload. The Course Blocks API
  (``?depth=N`` + ``requested_fields``) is the canonical reference.
* Minimal vs full variants **MUST** be documented in OpenAPI.

Convention 10: Standardize error responses (RFC 9457-style)
-----------------------------------------------------------

*Source:* `ADR 0029: Standardize Error Responses <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0029-standardize-error-responses.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0029-standardize-error-responses.rst#code-example-target-shape>`__).

All non-2xx responses **MUST** use a single, predictable, structured JSON error object
built on a platform-level DRF exception handler.

* Errors **MUST** use correct HTTP status codes (4xx/5xx); the platform **MUST NOT**
  mask failures behind HTTP 200 + ``success: false``.
* The error body **MUST** carry these core fields:

  * ``type``: an absolute URI identifying the problem class (opaque, stable
    identifier; need not be dereferenceable at runtime).
  * ``title``: short, developer/operator-facing summary of the error class.
  * ``status``: the HTTP status code.
  * ``detail``: a stable, developer-facing, English explanation of this occurrence,
    safe for APM/log aggregation. (This deviates from RFC 9457 intentionally;
    user-facing copy lives in ``user_message``.)
  * ``instance``: the request path that produced the error (e.g. ``/api/courses/v1/``).
  * ``user_message`` *(optional)*: a translatable, display-safe string for UIs.

* Validation errors **MUST** include an ``errors`` member mapping each invalid
  field/path to a list of message strings (maps directly onto DRF's
  ``ValidationError.detail``).
* A shared ``type`` catalog **MUST** be maintained. Initial entries:

  .. list-table::
     :header-rows: 1
     :widths: 55 10 35

     * - URI
       - Status
       - When to use
     * - ``https://docs.openedx.org/errors/not-found``
       - 404
       - Resource does not exist
     * - ``https://docs.openedx.org/errors/authz``
       - 403
       - Authenticated but not authorized
     * - ``https://docs.openedx.org/errors/authn``
       - 401
       - Not authenticated
     * - ``https://docs.openedx.org/errors/validation``
       - 400
       - Request body / query-param validation failure
     * - ``https://docs.openedx.org/errors/rate-limited``
       - 429
       - Rate limit exceeded
     * - ``https://docs.openedx.org/errors/internal``
       - 500
       - Unexpected server error

* JSON requests **MUST** receive JSON error bodies: the platform **MUST NOT**
  return Django's HTML error page for JSON requests. In production
  (``DEBUG=False``), 5xx / unhandled exceptions **MUST** return a generic body
  (no stack traces or internal detail). CORS headers **MUST** be preserved on error
  responses.
* The error shape **MUST** be registered as a reusable ``drf-spectacular`` component
  (``#/components/schemas/ErrorResponse``) referenced by all 4xx/5xx docs.

Error-format changes are treated as backward-compatible (well-behaved clients
tolerate extra JSON fields), so the default migration path is in-place. Teams with
clients tightly coupled to a legacy shape **MAY** version per Convention 12.

Documentation, versioning, and endpoint topology
================================================

Convention 11: Document every endpoint with ``drf-spectacular`` (OpenAPI 3.x)
-----------------------------------------------------------------------------

*Source:* `ADR 0027: Standardize API Documentation & Schema Coverage <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0027-standardize-api-documentation-and-schema-coverage.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0027-standardize-api-documentation-and-schema-coverage.rst#code-example-target-documentation-usage>`__).

All Open edX REST APIs **MUST** use ``drf-spectacular`` with ``@extend_schema`` for
complete, machine-readable OpenAPI 3.x documentation.

* Every endpoint **MUST** document request/response schemas, status codes, and
  error conditions, with descriptions and examples for complex endpoints.
* ``api-doc-tools`` (the platform shim over ``drf-yasg``) and the transitive
  ``drf-yasg`` dependency **MUST** be deprecated and removed once no consumers
  remain. ``drf-yasg`` only emits OpenAPI 2.0 and will not gain 3.x support, so it is
  not a viable long-term home. Swapping ``drf-spectacular`` in *underneath*
  ``api-doc-tools`` was rejected: the two libraries are not drop-in compatible at any
  meaningful layer (spec version, decorator API, parameter primitives, docstring
  semantics, schema view/settings, auth handling, extension architecture, and
  generated-schema output all differ).
* Existing ``api-doc-tools`` usages (``@schema``, ``@schema_for``, ``parameter()``,
  ``get_schema_view``) **MUST** be migrated directly to ``@extend_schema`` /
  ``OpenApiParameter`` / ``OpenApiResponse`` / ``SpectacularAPIView`` and
  ``SPECTACULAR_SETTINGS``. Both toolchains **MAY** co-exist during the transition;
  new endpoints **MUST** use ``drf-spectacular`` directly.

Convention 12: Version every new API and enforce compatibility in CI
--------------------------------------------------------------------

*Source:* `ADR 0037: API Versioning Strategy <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0037-api-versioning-strategy.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0037-api-versioning-strategy.rst#code-example-routing-pattern>`__). See also :ref:`OEP-21 Deprecation and Removal <OEP-21 DEPR>`.

* Every new API endpoint **MUST** include an explicit version in its URL path
  (e.g. ``/api/foo/v1/``). Unversioned paths are not permitted for new APIs.
* When multiple versions coexist, clients **MUST** prefer the highest available
  version; older versions are retained only during an active deprecation window.
* CI tooling **MUST** diff the OpenAPI schema against the base branch and fail the
  build when a backward-incompatible change (removed fields, changed types, removed
  endpoints) is introduced without a version bump.
* Removing an old version **MUST** follow :ref:`OEP-21 Deprecation and Removal <OEP-21 DEPR>`:
  file a DEPR issue (in the ``openedx/public-engineering`` project), mark the version
  ``deprecated: true`` in the schema and docstring, provide a migration guide, and
  communicate a removal timeline aligned with the release cycle (minimum one named
  release).
* Existing unversioned endpoints are out of scope for immediate migration, but new
  work on those services **SHOULD** add versioned paths.

Convention 13: One canonical front-end configuration endpoint; no per-user data on it
-------------------------------------------------------------------------------------

*Source:* `ADR 0035: Canonical MFE Configuration Endpoint <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0035-canonical-mfe-configuration-endpoint.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0035-canonical-mfe-configuration-endpoint.rst#code-example-configuration-endpoint-vs-user-context-resources>`__; partially supersedes the MFE Config API ADR 0001). See also :ref:`OEP-65 Frontend Composability <OEP-65 Frontend Composibility>`.

* ``/api/frontend_site_config/v1/`` (``FrontendSiteConfigView``, aligned with
  frontend-base's ``SiteConfig`` under OEP-65) is the **canonical** MFE / front-end
  runtime configuration endpoint. New consumers **MUST** target it.
* ``/api/mfe_config/v1`` (``MFEConfigView``) is legacy and on a deprecation path;
  existing consumers migrate to the canonical endpoint.
* **No new configuration endpoints.** If configuration needs grow, the canonical
  endpoint is extended via URL-path versioning (``/v2/``) rather than adding parallel
  surfaces.
* **User-contextual data is not configuration.** User roles, enrollments, and
  course-specific permissions **MUST NOT** be served as fields on a configuration
  payload: the configuration endpoints are unauthenticated and shared-cached, and
  such data is a set of first-class resources belonging at resource-oriented,
  authenticated endpoints.
* If splitting user context out increases request counts, that is solved on the
  client (batching or a Backend-for-Frontend, of which the Learning MFE courseware
  BFF is existing precedent), **not** by collapsing distinct resources into one
  multi-purpose endpoint.

Convention 14: Standardize REST API URL structure
-------------------------------------------------

*Source:* `ADR 0038: Standardize REST API URL Structure <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0038-standardize-rest-api-url-structure.rst>`__ (`code example <https://github.com/openedx/openedx-platform/blob/master/docs/decisions/0038-standardize-rest-api-url-structure.rst#code-examples>`__).

Every REST API URL **MUST** match one canonical shape, so a client that knows one
endpoint can infer the next and tooling can address resources it has not seen::

   /api/{api_name}/v{N}/{collection}[/{identifier}[/{sub_collection}[/{sub_id}]]]/

* **Leading** ``/api/`` **prefix.** Every programmatic endpoint lives under
  ``/api/``, never mid-path. HTML views, XBlock runtime handlers, fixed-form authn
  paths, and operational endpoints (``/heartbeat``) are out of scope.
* **Singular API name, plural collections** (``/api/enrollment/v1/enrollments/``).
  A resource gets exactly two base URLs: ``{collection}/`` and
  ``{collection}/{identifier}/``.
* **Name the domain, not the code.** The API name describes a subject area, not the
  implementing Django app, package layout, or an MFE screen; each concept has one
  name platform-wide.
* **Lowercase** ``snake_case`` **segments** are concrete resource nouns, not
  ``kebab-case``, abstract names (``items``, ``data``), or view/screen names.
* **Version follows the API name and is major-only** (``v1``, never ``v0.5``, never
  a leading ``/api/v1/...``). This fixes only the version's position and form;
  versioning semantics stay with Convention 12.
* **One route, one address.** The trailing slash is required and every ``re_path``
  is anchored with ``^`` and ``$``.
* **Hierarchy is logical, capped at one level of nesting.** Anything addressable by
  its own opaque key is a top-level collection; the parent and all filtering,
  sorting, and pagination live in the query string (Conventions 7-9). A parent key
  **MUST NOT** occupy the member-identifier slot.
* **Opaque-key identifiers**, resolved by shared path converters, never database
  primary keys on an external API; the current user is addressed as ``me``.
* **Verb-free resource paths**: an operation is a ``POST`` to a noun (Convention 3).
* **One shared namespace across LMS and CMS.** A path served by both **MUST**
  resolve to the same contract; ``{api_name}`` values are reserved platform-wide;
  service-specific behavior is distinguished by name (``/api/authoring/v1/`` for
  Studio, ``/api/course/v1/`` for the LMS), and each endpoint has one canonical
  address. Each mount declares its own full ``/api/{api_name}/v{N}/`` prefix, and
  Django URL names are unique, descriptive ``snake_case``.

Existing endpoints migrate under :ref:`OEP-21 Deprecation and Removal <OEP-21 DEPR>`
and are never broken: mount the conforming path alongside the legacy one, mark the
legacy route ``deprecated: true``, and remove it after at least one named release.
A CI check walks each service's composed URL resolver to enforce the
machine-checkable rules.

Rationale
*********

These conventions were developed as fourteen separate ADRs so each decision could
be reasoned about, reviewed, and merged against the code it affected. That
granularity was valuable during authoring but, as community reviewers observed,
made the overall picture hard to follow: "a bit confusing as to what info is found
in which ADR." Consolidating them into a single OEP gives contributors and external
integrators one authoritative reference for "how Open edX REST APIs should look,"
while the per-decision ADRs remain in ``openedx-platform`` next to the code as the
detailed record.

Design decisions throughout favor:

* **DRF-native patterns** (serializers, ViewSets, ``permission_classes``,
  ``django-filter``, ``DefaultPagination``, ``drf-spectacular``) over bespoke
  platform machinery, because they are well understood, well documented, and produce
  consistent OpenAPI output.
* **Reusing what already exists** in the platform where it is sound:
  ``DefaultPagination`` from ``edx-drf-extensions``, ``django-filter`` usage in
  several apps, the Course Blocks API's ``depth``/``requested_fields`` behavior, and
  the existing Learning MFE BFF, to minimize migration churn.
* **Machine-readability as a first-class requirement**, motivated directly by the
  goal of making the platform reliably consumable by external systems and AI agents.

Where a decision deviates from an external standard (for example, the ``detail``
field in Convention 10 relative to RFC 9457), the deviation is deliberate and is
documented in the source ADR.

Backward Compatibility
**********************

This OEP is intentionally incremental and, taken as a whole, backward-compatible in
its rollout posture:

* New code follows these conventions immediately; existing endpoints are migrated
  over time.
* Any change that would break an existing contract (e.g. migrating ``limit``/``offset``
  to ``page``/``page_size``, changing URL structure during ViewSet consolidation, or
  altering a response contract) **MUST** be delivered as a **new API version**, with
  the old version deprecated per :ref:`OEP-21 Deprecation and Removal <OEP-21 DEPR>`.
* Merged/consolidated endpoints (Convention 3) and renamed filter parameters
  (Convention 8) **MUST** retain legacy URLs/parameters as deprecated aliases that
  emit ``Deprecation`` headers for a defined window.
* Error-response standardization (Convention 10) is treated as backward-compatible,
  since well-behaved clients tolerate additional JSON fields; teams with tightly
  coupled clients may version instead.
* CI enforcement of OpenAPI compatibility (Convention 12) is the mechanism that
  keeps these guarantees honest going forward.

Reference Implementation
************************

The reference implementation is the ongoing FC-0118 work in ``openedx-platform``,
tracked under the umbrella issue `#38137
<https://github.com/openedx/openedx-platform/issues/38137>`_ and recorded as the
ADRs in the `docs/decisions folder <https://github.com/openedx/openedx-platform/tree/master/docs/decisions>`__ (``0025`` through ``0038``, plus
the `pointer ADR 0017 <https://github.com/openedx/openedx-platform/blob/master/openedx/core/djangoapps/oauth_dispatch/docs/decisions/0017-standardize-authentication-patterns.rst>`__). Priority
migration targets identified by the ADRs include the Enrollment API ViewSet
consolidation, the ``drf-spectacular`` rollout, the ``BearerAuthentication``
deprecation via the ``view_auth_classes`` decorator, and the canonical
front-end configuration endpoint. This OEP may move to "Final" once the conventions
have representative, merged implementations across the highest-impact endpoints.

Rejected Alternatives
*********************

Because this OEP consolidates fourteen ADRs, the full set of alternatives is
recorded per-decision in the source ADRs. The most significant rejected
alternatives are:

* **Leaving the decisions as fourteen separate ADRs only.** Rejected in response to
  community review requesting a single, clear conventions document.
* **Adopting ``dataclasses``/``pydantic`` instead of DRF serializers** (Convention 1).
  Deferred: migrating from two existing patterns to a third adds complexity without
  clear near-term benefit while the platform is still being made consistent.
* **Swapping ``drf-spectacular`` in underneath ``api-doc-tools``** (Convention 11).
  Rejected: the libraries are not drop-in compatible, so consumers would need
  migration regardless, while locking the project into maintaining a wrapper.
* **Standardizing on ``LimitOffsetPagination`` or ``CursorPagination``**
  (Convention 7). Rejected in favor of the already-prevalent, numbered-page
  ``DefaultPagination``, which fits existing MFE controls and bookmarkable deep links.
* **Adopting ``drf-standardized-errors``** (Convention 10). Considered but not
  adopted: it would add a core dependency, and the required CORS/non-``APIView``
  behavior would mean overriding most of the library anyway.
* **Treating unversioned paths as the default "stable" surface** (Convention 12).
  Rejected: unversioned aliases create ambiguity and provide no deprecation
  contract.
* **Adding a third MFE configuration endpoint, or folding per-user data into
  configuration** (Convention 13). Rejected: both worsen the ambiguity and safety
  problems the convention exists to remove.

Change History
**************

2026-07-05
==========

* Document created: consolidates ``openedx-platform`` ADRs 0025–0038 (FC-0118) into a
  single REST API Conventions OEP.
* `Pull request #805 <https://github.com/openedx/openedx-proposals/pull/805>`_
