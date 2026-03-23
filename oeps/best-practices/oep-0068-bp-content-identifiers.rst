.. _OEP-68 Learing Content Identifiers:

OEP-68: Learning Content Identifiers
####################################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-0068 <OEP-68 Learning Content Identifiers>`
   * - Title
     - Learning Content Identifiers
   * - Last Modified
     - 2026-03-18
   * - Authors
     - Kyle McCormick <kyle@axim.org>
   * - Arbiter
     - Sarina Canelake <sarina@axim.org>
   * - Status
     - Draft
   * - Type
     - Best Practice
   * - Created
     - 2026-03-18
   * - Review Period
     - TBD
   * - Resolution
     - TBD
   * - References
     -

.. contents::
   :local:
   :depth: 1

Abstract
********

When making references to learning content it's important to choose the right kind of
identifier and name it in a way that is obvious to others. We describe five
categories of identifiers: Intger Primary Keys, Codes, Opaque Keys, and UUIDs. We describe
when to use them and how to name them. Following these guidelines will help keep
the platform content coherent, correct, portable, and efficient.

The primary audience of this OEP is **developers working on openedx-core, openedx-platform
and other repositories which manage learning content**. However, plugin
developers, site administrators, and nontechnical Open edX Core Contributors
who interface with learning content may all benefit from understanding these
concepts.

Motivation
**********

Identifiers are ubiquitous: they appear as Python variables, function parameters, Django model
fields, database columns, REST API fields, and event data schemas. It's important to choose
the right identifier for the right job. For example:

* Including Open edX instance-specific information in an identifier can cause content to
  break when it's transferred to other instances. For example, exporting course content with
  references to the integer primary keys of learner teams (rather than the teams' slugs)
  makes those content-team relationships invalid when transferred to any other instance.

* Including learning-context-specific information in an identifier can cause content to break
  when it's reused in another learning context. For example, when copying a component and its
  media files from course run X to course run Y, the transfer format must describe the
  component-media relationships *without* reference to the course run keys for X or Y,
  otherwise the copied component in Y may erronously try to reference media files from 
  course run X.

* Mingling version-aware and version-agnostic identifiers can lead to unexpected lookup failures.
  For example, if learner-facing code queries a course run cache for the latest content without
  specifying a version number, but the cache entries are saved using identifiers that include
  the version number, then the cache will never be hit.

* Using long strings for foreign keys instead of integers can make indexes unnecessarily large
  and inefficient. For example, the table which joins components and media files uses
  their integer primary keys rather than paths or slugs. Only when exporting are the
  integer keys resolved into a portable string format.

It's easier to review code for appropriate usage of identifiers when it's obvious which kind
of identifier is expected to be in any given variable, model field, etc. Type annotations
go a long way towards helping this, but there are many situations which types are unchecked
and/or everything has been serialized into dicts and strings (such as REST APIs, frontend code,
tracking events, legacy untyped Python modules, and logs). So, this OEP aims to:

* Help developers make informed decisions about which types of identifiers to use.
* Help developers name their variables and fields so that, whether or not type annotations
  are available, the type of identifiers is obvious to the audience (revewiers, future
  developers, site admins, data researchers, advanced platform users, etc.).

Specification
*************

Here are five recognized categories of learning content identifier in Open edX code.
When using an identifier, first determine which category it belongs to, then consider
if it's appropriate for the job at hand. Finally, apply the appropriate naming convention,
as long as doing is backwards compatible and consistent with surrounding code (some judgement required here).

These conventions apply wherever Open edX learning content is referenced: Python variables, JS
variables, Django model field names, REST API arguments, event schema fields, admin interfaces,
application logs, and so on.

Summary
=======

.. list-table::
   :header-rows: 1
   :widths: 20 30 25 25

   * - Category
     - What it is
     - Python type
     - Naming convention
   * - Integer Primary Key
     - Auto-incremented integer row identifier
     - ``int``
     - * ``id``, ``*_id`` on the model.
       * ``pk``, ``*_pk`` everywhere else.
   * - Code
     - Locally-scoped slug-like string
     - ``str``
     - * ``*_code``
   * - OpaqueKey
     - Codes composed together into a semi-readable instance-wide identifier
     - subclass of ``OpaqueKey``
     - * ``*_key`` for parsed OpaqueKey objects
       * ``*_key`` or ``*_key_string`` for serialized OpaqueKey strings
   * - UUID
     - Globally unique identifier scoped across all Open edX instances
     - ``uuid.UUID``
     - * ``*_uuid`` for parsed UUID objects
       * ``*_uuid`` or ``*_uuid_string`` for serialized UUID strings

Integer Primary Keys
====================

Every Open edX Django model should use ``django.db.models.BigAutoField`` as its primary key. This is an auto-
incremented integer assigned by the database and meaningful only within that database.

**When to use**: Primary keys are the default choice for referencing database rows within an IDA
within an instance. Always use primary keys for foreign key relationships on Django models. As
integers, these keys an be indexed in the database with little overhead, so they're unmatched
in efficiency when it comes to quickly lookup up data, joining data across tables, or
enforcing data constriants.
However, primary keys become meangliness across IDAs or instances--in these situations, one
should use Codes, OpaqueKeys, or UUIDs (more on that below).

**How to name**: By default, define every Django model with an integer primary key named ``id``.
When defining references to other models (e.g. ``collection``) Django will automatically define the
underlying integer foreign key using the suffix ``_id`` (e.g. ``collection_id``). However, "id" is a
highly overloaded term and can mean many different things outside the context of a Django model.
So, in all other circumstances (variable names, REST APIs, event schemas, etc.) it's preferred to use
the suffix ``_pk`` (e.g. ``collection_pk``). This makes it completely unambiguous that the value
is expected to be the integer primary key of Django model.

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

A code is a short, slug-like, locally-scoped string identifier. Codes generally contain alphanumeric
characters, hyphens, underscores, and periods. They are generally (but not always) case sensitive.
A code identifies something *within some enclosing context*—it is not globally unique on its own.
Codes can be joined together to identify something within a broader scope.

For example:

* A ``block_code`` differentiates a particular XBlock usage from other usages of
  the same type within a learning context.
* A ``block_code`` and ``type_code`` together differentiate a particular XBlock
  usage from all blocks in a learning context.
* An ``org_code``, ``course_code``, ``run_code``, ``block_code`` and ``type_code`` together
  differentate a particular XBlock usage from all blocks on an Open edX instance.

**When to use**: Codes are ideal for identifying something in that way that's both semantically descriptive
and intentionally limited in specificity. This makes them ideal for transfer formats,
where omission of information is just as important as inclusion of information.
For example, OLX links blocks together using their ``block_code`` and ``type_code``, but *without*
their ``org_code/course_code/run_code``--this way, the OLX can easily be imported into others instances, courses,
or libraries.

**How to name**: Variables and fields holding a code should use the suffix ``_code``.

Historically, codes are sometimes called "slugs" or "shortnames". Existing code may use suffixes like ``_slug``,
``_id``, or no suffix at all (``org``, ``run``, ``block_type``, etc.). The suffix ``_code`` is preferred for new code.

OpaqueKeys
==========

An ``OpaqueKey`` (defined in `openedx/opaque-keys`_) is an immutable Python object
consisting of a "key type" and one or more codes which, all together, uniquely identify
some resource on an Open edX instance. OpaqueKey is an abstract base class; it is organized
into hiearchy of subclasses, with abstract intermediate classes for concepts (like
``LearningContextKey`` and ``UsageKey``) and concrete subclasses for specific key
types (like ``CourseLocator``, ``LibraryUsageLocatorV2``).
Each concrete key type serializes to a predictable string representation.

For example:

* ``"course-v1:Axim+Chem101+Spring2026"`` includes:
  
  * ``course-v1`` (the key type is "course run")
  * ``Axim`` (the org code)
  * ``Chem101`` (the course code)
  * ``Spring2026`` (the run code)

* ``"lb:Axim:problem:Atoms6"``

  * ``lb`` (the key type is "library block")
  * ``Axim`` (the org code)
  * ``ChemLib`` (the library code)
  * ``problem`` (the type code)
  * ``Atom6`` (the component/block code)

**When to use**: Integer primary keys and OpaqueKeys both uniquely identify a resource across an
Open edX instance. When choosing between the two, consider the following:

* Integer primary keys are by far the most efficient and reliable method to relate tables within a database.
* When displayed, OpaqueKeys provide more information. This can be good in cases where quasi-readability
  matters, such as URLs, error logs, event data, and UIs for admins and power-users.
* Because they are human-readable, site admins may want to change the code identifying a piece of content,
  which would break any OpaqueKey reference to that content. This generally doesn't happen, but by limiting
  the number of internal OpaqueKey references, we make the platform more flexible to support these sort
  of operations (e.g. changing a course's run code) in the future.

**How to name**: 

* Python variables and attributes holding a parsed ``OpaqueKey`` object should use the suffix ``_key``.
* Fields which marshal between OpaqueKey objects and their serialized strings, such as Django Model
  Fields or Serializer Fields, should also use the suffix ``_key``.
* REST APIs, event data fields, or other external serializations of OpaqueKeys should all also use
  the suffix ``_key``.
* When parsed ``OpaqueKey`` objects and serialized key strings co-exist in the same context,
  such as a parsing function, use the suffix ``_key_string`` to disambiguate serialized strings.
* Frontend variables should use the suffix ``*Key``. The suffix ``*KeyString`` is not necessary,
  because parsed OpaqueKeys do not exist on the frontend.

.. code-block:: python

   # Preferred
   course_key: CourseKey = CourseKey.from_string("course-v1:Axim+Demo+2026")
   usage_key: UsageKey = ...
   collection_key: LibraryCollectionLocator = ...

   def get_course(course_key: CourseKey) -> CourseOverview: ...

Please note that it's preferrable to pass around the parsed ``OpaqueKey`` object
whenever it's available--compared to the serialized key string, it's more
type-safe and centralizes all the parsing logic. Developers are also encouraged to use
``OpaqueKey`` subclasses as type annotations wherever appropriate.

Hint: For historical reasons, concrete OpaqueKey subclasses use the suffix ``Locator``
instead of ``Key``. For all intents and purposes, this distinction can be ignored by consumers. They
are all ``_keys``. In the future, we will unify all OpaqueKey classes to be named ``*Key``.

.. _openedx/opaque-keys: https://github.com/openedx/opaque-keys

UUIDs
=====

A UUID (Universally Unique Identifier) is a 128-bit identifier that uniquely identifies a resource
across *all* Open edX instances. Unlike primary keys and OpaqueKeys, UUIDs are not scoped to a
single database or instance.

**When to use**: Use UUIDs when you want to give an object an identity that is unique across all
Open edX instances.

**How to name**:

* Python variables and attributes holding a parsed ``uuid.UUID`` object should use the suffix ``_uuid``.
* Fields which marshal between UUID objects and their serialized strings, such as Django Model
  Fields or Serializer Fields, should also use the suffix ``_uuid``.
* REST APIs, event data fields, or other external serializations of UUIDs should all also use
  the suffix ``_uuid``.
* When parsed UUID objects and serialized UUID strings co-exist in the same context,
  such as a parsing function, use the suffix ``_uuid_string`` to disambiguate serialized strings.
* Frontend variables should use the suffix ``*Uuid``. The suffix ``*UuidString`` is not necessary,
  because parsed UUID objects are not used in Open edX frontend code.

Other Identifiers
=================

Not every identifier fits neatly into one of the above categories. When an identifier does not,
choose a name that does not use any of the suffixes ``_pk``, ``_key``, ``_key_string``, ``_code``,
``_uuid``, or ``_uuid_string``, so that readers are not misled into assuming a type or scope that
does not apply.

Examples:

* The ``refname`` field on ``PublishableEntity`` objects in ``openedx-core``. A ``refname``
  correlates a database entity
  with its representation in off-platform content archives. It is not a primary key (which would be
  database-specific), not a code (because it may contain non-slug characters), and not an OpaqueKey
  (because it cannot be parsed into a globally-scoped identifier). By choosing the name
  ``refname``—which collides with none of the conventions above—the code signals clearly that this
  identifier is its own distinct thing.

* The integer ``version_num`` is used as part of the identity several version-aware content models.
  It is like a Code, because it identifies a thing (a version) within a local context (a versioned entity).
  However, it's not a string, so we don't use the suffix ``_code``.

* A ``BlockRef`` is a 2-tuple consisting of ``(type_code, block_code)``, locally  identifying a block usage
  within a learning context. Historically, this is often referred to as ``BlockKey``, but this has been very
  confusing, as ``block_key`` is also used to refer to ``UsageKeys``, which identifies a block usage
  *across an entire instance*. 

Rationale
*********

The five categories above cover the vast majority of identifiers that appear in Open edX Python
code. Keeping the naming conventions to a small, well-defined set makes them easy to learn and apply
consistently. The ``_pk`` / ``_key`` / ``_code`` / ``_uuid`` suffixes were chosen because they are
short, distinct from one another, and directly evocative of the kind of identifier they represent.

The ``_key_string`` and ``_uuid_string`` conventions were chosen over alternatives like ``_key_str``
or ``_uuid_str`` for readability. The word "string" more clearly signals to a reader that the value
is a plain string rather than a Python object.

Consequences & Backward Compatibility
*************************************

Start: New conventions
======================

* ``_pk`` for integer primary key variables.
* ``_key_string`` and ``_uuid_string`` for stringified OpaqueKeys and UUIDs.
* ``_code`` for codes (and, the term "code" in general)
* ``BlockRef`` and ``block_ref`` for 2-tuples of ``(type_code, block_code)``

Stop: Old patterns to drop
==========================

* ``_id`` for OpaqueKeys (e.g. ``course_id``)
* ``_id`` for codes (e.g., ``block_id``)
* ``BlockKey`` and ``block_key``
* In OpaqueKeys, ``*Locator`` classes will be renamed to ``*Key``
 
Continue: Already widely adopted
================================

* ``id`` field on models for integer primary keys.
* ``_key`` for OpaqueKey objects.
* ``_uuid`` for UUID objects.

Migration plan
==============

* The guidance above applies immediately to new code.
* Start retroactively applying guidance ``openedx-core``, which has few references to update.
* Move on to ``opaque-keys``, probably after Verawood.
* Eventually, time permitting, consider updating existing variables and renaming model fields in ``openedx-platform``.
* Whenever renaming classes, keep old names as aliases to new ones
* Whenever renaming fields, use ``@property`` to make readonly backcompat aliases

Change History
**************

2026-03-18: Initial draft.
