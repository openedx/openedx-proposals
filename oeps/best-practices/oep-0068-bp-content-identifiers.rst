.. _OEP-68 Learning Content Identifiers:

OEP-68: Learning Content Identifiers
####################################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-0068 <OEP-68 Learning Content Identifiers>`
   * - Title
     - Learning Content Identifiers
   * - Last Modified
     - 2026-03-28
   * - Authors
     - Kyle McCormick <kyle@axim.org>
   * - Arbiter
     - Sarina Canelake <sarina@axim.org>
   * - Status
     - Draft
   * - Type
     - Best Practice
   * - Created
     - 2026-03-28
   * - Review Period
     - 2026-03-28 - 2026-04-13
   * - Resolution
     - TBD

.. contents::
   :local:
   :depth: 1

Abstract
********

Open edX code has four kinds of identifiers for learning content: **Integer Primary Keys**,
**Codes**, **OpaqueKeys**, and **UUIDs**. Choosing the wrong kind—or naming it ambiguously—can
cause subtle bugs, broken exports, and painful code reviews. This OEP explains what each kind
is, when to reach for it, and how to name it consistently.

The primary audience is **developers working on openedx-core, openedx-platform, and other
repositories that manage learning content**. Plugin developers, site administrators, and
nontechnical Open edX Core Contributors who work with learning content will also find these
concepts useful.

Motivation
**********

Identifiers show up everywhere: Python variables, function parameters, Django model fields,
database columns, REST API fields, event data schemas. Using the right identifier for the right
job matters. Here are some ways things can go wrong when the wrong kind is used:

* **Instance-specific identifiers break exports.** Integer primary keys are assigned by a
  specific database and are meaningless anywhere else. For example, if exported course content
  references the primary key of a learner team (rather than the team's slug), those
  content-team relationships become invalid the moment the content is imported into a
  different instance.

* **Context-specific identifiers break reuse.** When copying a component and its media files
  from course run X to course run Y, the transfer format must describe those relationships
  *without* mentioning X or Y by name. Otherwise the copied component in Y may erroneously
  try to reference media files from course run X.

* **Mixing version-aware and version-agnostic identifiers breaks lookups.** If
  learner-facing code queries a cache using a version-agnostic key, but cache entries were
  stored under version-specific keys, the cache will never be hit.

* **String foreign keys make databases slow.** Long strings make indexes unnecessarily large.
  The table joining components to media files uses integer primary keys internally—only at
  export time are those keys resolved into portable strings.

Consistent naming also makes code review easier. Type annotations help, but a lot of
Open edX code operates in contexts where types aren't checked: REST APIs, frontend JavaScript,
tracking events, legacy Python modules, and log output. When a variable is named ``course_key``
you immediately know it holds an OpaqueKey; when it's named ``course_pk`` you know it's a
database integer. This OEP aims to:

* Help developers make informed decisions about which kind of identifier to use.
* Give every identifier a name that makes its kind obvious—to reviewers, future developers,
  site admins, data researchers, and power users—even when no type annotations are present.

Specification
*************

Open edX code uses four categories of learning content identifier. When working with an
identifier, figure out which category it belongs to, decide whether it's the right fit for
the job, and then apply the naming convention below. Use your judgement when adopting
conventions in existing code—backwards compatibility and consistency with surrounding code
come first.

These conventions apply everywhere learning content is referenced: Python variables, JS
variables, Django model field names, REST API arguments, event schema fields, admin
interfaces, application logs, and so on.

Summary
=======

.. list-table::
   :header-rows: 1
   :widths: 18 28 15 22 17

   * - Category
     - What it is
     - Python type
     - Naming convention
     - Storage
   * - Integer Primary Key
     - Auto-incremented integer row identifier
     - ``int``
     - * ``id``, ``*_id`` on the model.
       * ``pk``, ``*_pk`` everywhere else.
     - ``BigAutoField``
   * - Code
     - Locally-scoped slug-like string
     - ``str``
     - * ``*_code``
     - ``code_field``
   * - OpaqueKey
     - Codes composed together into a semi-readable instance-wide identifier
     - subclass of ``OpaqueKey``
     - * ``*_key`` for parsed OpaqueKey objects
       * ``*_key`` or ``*_key_str`` for serialized OpaqueKey strings
     - ``OpaqueKeyField``
   * - UUID
     - Globally unique identifier scoped across all Open edX instances
     - ``uuid.UUID``
     - * ``*_uuid`` for parsed UUID objects
       * ``*_uuid`` or ``*_uuid_str`` for serialized UUID strings
     - ``UUIDField``

Integer Primary Keys
====================

Every Open edX Django model should declare an auto-incrementing integer primary key.

**When to use:** Primary keys are the default way to reference a database row within a single
deployed service on a single instance. Always use them for Django model foreign key relationships—as
integers, they can be indexed with almost no overhead, making lookups, joins, and constraint
enforcement as fast as possible.
The trade-off is that primary keys are meaningless outside the database that assigned them.

**How to name:** On Django models, the primary key is ``id`` and foreign keys automatically
get the ``_id`` suffix (e.g. ``collection_id``). Outside of model definitions—in variable
names, REST APIs, event schemas, and so on—use the suffix ``_pk`` instead (e.g.
``collection_pk``). When accessing the primary key on a django model, prefer ``.pk``, e.g.
``collection.pk``.  "id" is an overloaded term that means many things; ``_pk`` leaves no
doubt that the value is a Django model integer primary key.

**How to store:** By default, use ``django.db.models.BigAutoField`` for all primary keys.
In rare cases—when a model has very few rows (like an enumeration) or receives a massive
number of foreign key references—use the smaller ``django.db.models.AutoField`` instead.

.. code-block:: python

   @dataclass
   class TeamMembershipInfo:
       user_pk: int
       user_fullname: str
       team_pk: int
       team_name: str

   def get_memberships(user_pk: int) -> Iterable[TeamMembershipInfo]: ...

Existing Open edX code frequently uses ``<object>_id`` for variables holding primary keys.
This is acceptable, but ``<object>_pk`` is preferred for new code.

Codes
=====

A code is a short, slug-like string that identifies something *within a specific enclosing
context*. Codes typically contain alphanumeric characters, hyphens, underscores, and periods,
and are generally case-sensitive. A code alone is not globally unique—its meaning depends on
the scope it lives in. Multiple codes can be combined to identify something in a broader scope.

For example:

* A ``block_code`` distinguishes one XBlock usage from others of the same type within a
  learning context.
* A ``block_code`` and ``type_code`` together uniquely identify an XBlock usage among all
  blocks in a learning context.
* An ``org_code``, ``course_code``, ``run_code``, ``block_code``, and ``type_code`` together
  uniquely identify an XBlock usage among all blocks on an Open edX instance.

**When to use:** Codes are the right choice when you want an identifier that is descriptive
but deliberately limited in scope. This makes them ideal for transfer formats, where
*leaving information out* is just as important as including it. For example, OLX links blocks
together using ``block_code`` and ``type_code``, but intentionally omits
``org_code/course_code/run_code``—so the OLX can be imported into any instance, course, or
library without modification.

**How to name:** Variables and fields holding a code should use the suffix ``_code``.

Historically, codes have been called "slugs" or "shortnames", and existing code may use
suffixes like ``_slug``, ``_id``, or no suffix at all (e.g. ``org``, ``run``,
``block_type``). The suffix ``_code`` is preferred for new code.

**How to store:** By default, store codes in a case-sensitive ``CharField`` of length 255
with a regex validator. A factory function is available at ``openedx_django_lib.fields.code_field``.


OpaqueKeys
==========

An ``OpaqueKey`` (defined in `openedx/opaque-keys`_) is an immutable Python object that
bundles a "key type" and one or more codes to uniquely identify a resource within an Open edX
instance. Think of it as a structured, semi-human-readable instance-scoped identifier.

``OpaqueKey`` is an abstract base class organized into a hierarchy: abstract intermediate
classes represent broad concepts (like ``LearningContextKey`` and ``UsageKey``), while concrete
subclasses represent specific resource types (like ``CourseLocator`` and ``LibraryUsageLocatorV2``).
Each concrete type serializes to a predictable string format.

**A note on cross-instance usage:** OpaqueKeys are designed for use within a single instance,
but the same OpaqueKey can naturally appear in multiple instances. For example, if you export a
course with org code ``Axim``, course code ``Chem101``, and run code ``Spring2026`` from one
instance and import it elsewhere using the same codes, all the blocks will have identical
OpaqueKeys. This is useful: external tools like catalog integrations, sync workflows, and
reporting scripts can use OpaqueKeys to correlate data across instances.

However, OpaqueKeys have limits. Content can diverge after import, learner data stays
instance-specific, and Open edX doesn't enforce any global meaning to OpaqueKeys across
boundaries. If you need an identifier that is truly unique across all instances, use UUIDs
instead.

For example:

* ``"course-v1:Axim+Chem101+Spring2026"`` contains:

  * ``course-v1`` (key type: "course run")
  * ``Axim`` (the org code)
  * ``Chem101`` (the course code)
  * ``Spring2026`` (the run code)

* ``"lb:Axim:ChemLib:problem:Atoms6"`` contains:

  * ``lb`` (key type: "library block")
  * ``Axim`` (the org code)
  * ``ChemLib`` (the library code)
  * ``problem`` (the type code)
  * ``Atoms6`` (the component/block code)

**When to use:** Both integer primary keys and OpaqueKeys uniquely identify a resource within
an instance, so when do you choose one over the other?

* **Use primary keys for database relationships.** They're far more efficient for joins,
  lookups, and constraint enforcement.
* **Use OpaqueKeys where quasi-readability matters.** OpaqueKey strings carry semantic
  information, making them a better fit for URLs, error logs, event data, and admin or
  power-user UIs.
* **Prefer fewer internal OpaqueKey references for flexibility.** Because OpaqueKeys embed
  human-readable codes, a site admin might want to rename a course and change its run code.
  This is rare, but keeping internal OpaqueKey references to a minimum makes the platform
  more adaptable to these operations in the future.

**How to name:**

* Python variables and attributes holding a parsed ``OpaqueKey`` object → ``_key`` suffix.
* Django Model Fields and Serializer Fields that convert between OpaqueKey objects and
  strings → ``_key`` suffix.
* REST APIs, event data fields, and other external representations → ``_key`` suffix.
* When parsed objects and raw strings co-exist in the same context (e.g. a parsing
  function) → use ``_key_str`` for the raw string to disambiguate.
* Frontend variables → ``*Key`` suffix. ``*KeyString`` is not needed because parsed
  OpaqueKey objects don't exist on the frontend.

.. code-block:: python

   # Preferred
   course_key: CourseKey = CourseKey.from_string("course-v1:Axim+Demo+2026")
   usage_key: UsageKey = ...
   collection_key: LibraryCollectionLocator = ...

   def get_course(course_key: CourseKey) -> CourseOverview: ...

Prefer passing parsed ``OpaqueKey`` objects over raw strings whenever possible—they're
type-safe and keep all parsing logic in one place. Use the specific ``OpaqueKey`` subclass
as a type annotation wherever it's known.

**How to store:** The best way to store an OpaqueKey is an ``OpaqueKeyField`` subclass such
as ``UsageKeyField``, providing automatic marshalling between OpaqueKey objects and their
string representations in SQL VARCHARs.

💡 **Historical note:** Concrete OpaqueKey subclasses use the suffix ``Locator`` instead of
``Key`` for historical reasons. This distinction can be ignored by consumers—they're all
``_keys``. We plan to rename all ``*Locator`` classes to ``*Key`` in the future.

⚠️ **Historical hazard:** Older types like ``CourseLocator`` can represent both
version-aware and version-agnostic identifiers. This has led to all sorts of bugs and is
widely considered a mistake. New OpaqueKey classes should refer to exactly one kind of
resource—version information must be either always present or always absent, never optional.

.. _openedx/opaque-keys: https://github.com/openedx/opaque-keys

UUIDs
=====

A UUID (Universally Unique Identifier) is a 128-bit identifier that is unique across *all*
Open edX instances, not just one. Unlike primary keys and OpaqueKeys, UUIDs have no
dependency on any particular database or deployment.

**When to use:** Use UUIDs when an object needs an identity that is stable and globally
unique across every Open edX instance, allowing the objects to be shared outside and
across instances without risk of collision. For example:

* Learner certificates awarded on different Open edX instances should have distinct
  UUIDs, even if their instance-local identifiers (course run key, user primary
  key) are identical.
* Changelog entries should have distinct UUIDs, even if the changes are identical.

**How to name:**

* Python variables and attributes holding a parsed ``uuid.UUID`` object → ``_uuid`` suffix.
* Django Model Fields and Serializer Fields that convert between UUID objects and strings →
  ``_uuid`` suffix.
* REST APIs, event data fields, and other external representations → ``_uuid`` suffix.
* When parsed objects and raw strings co-exist in the same context → use ``_uuid_str``
  for the raw string to disambiguate.
* Frontend variables → ``*Uuid`` suffix. ``*UuidString`` is not needed because parsed UUID
  objects are not used in Open edX frontend code.

**How to store:** The best way to store a UUID is a ``UUIDField``.

Not every object needs a universal identity. Consider the need before defining a ``UUDIField``.

Other Identifiers
=================

Not every identifier fits neatly into one of the four categories above. When that happens,
choose a name that avoids the reserved suffixes ``_pk``, ``_key``, ``_key_str``,
``_code``, ``_uuid``, and ``_uuid_str``. Using a different name signals clearly to
readers that this identifier has its own semantics and shouldn't be treated as one of the
standard types.

Examples:

* ``refname`` on ``PublishableEntity`` in ``openedx-core`` — this field correlates a
  database entity with its representation in off-platform content archives. It isn't a
  primary key (those are database-specific), a code (it may contain non-slug characters),
  or an OpaqueKey (it can't be parsed into an instance-scoped identifier). The name
  ``refname`` signals that it's something distinct.

* ``version_num`` — an integer used as part of the identity of several version-aware content
  models. It resembles a Code conceptually (it identifies a version within a versioned
  entity), but it's an integer rather than a string, so ``_code`` would be misleading.

* ``BlockRef`` — a 2-tuple of ``(type_code, block_code)`` that locally identifies a block
  usage within a learning context. Historically this was called ``BlockKey``, which proved
  confusing because ``block_key`` is also used for ``UsageKeys``, which identify a block
  *across an entire instance*.

Consequences
************

Start: New conventions
======================

* ``_pk`` for integer primary key variables.
* ``_code`` for codes (and, the term "code" in general)
* ``BlockRef`` and ``block_ref`` for 2-tuples of ``(type_code, block_code)``

Stop: Old patterns to drop
==========================

* ``_id`` for OpaqueKeys (e.g. ``course_id``)
* ``_id`` for codes (e.g., ``block_id``)
* ``_key`` for codes (e.g., ``collection_key``)
* ``_key`` for non-OpaqueKey ref strings (e.g. ``PublishableEntity.key``)
* ``BlockKey`` and ``block_key``
* In OpaqueKeys, ``*Locator`` classes will be renamed to ``*Key``

Continue: Already widely adopted
================================

* ``id`` field on models for integer primary keys.
* ``_key`` for OpaqueKey objects.
* ``_uuid`` for UUID objects.
* ``_key_str`` and ``_uuid_str`` for stringified OpaqueKeys and UUIDs.

Migration plan
==============

* The guidance above applies immediately to new code.
* Start retroactively applying guidance in ``openedx-core``, which has few references to update.
* Move on to ``opaque-keys``, probably after Verawood.
* Eventually, time permitting, consider updating existing variables and renaming model fields in ``openedx-platform``.
* Whenever renaming classes, keep old names as aliases to new ones.
* Whenever renaming fields, use ``@property`` to make readonly backcompat aliases.

Change History
**************

2026-03-23
==========

* Initial proposal
* `Pull request #773 <https://github.com/openedx/openedx-proposals/pull/773>`_
