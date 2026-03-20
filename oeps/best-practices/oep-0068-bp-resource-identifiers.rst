.. _OEP-68 Learing Content Identifiers:

OEP-68: Learning Content Identifiers
####################################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-0068 <OEP-68 Resource Identifiers>`
   * - Title
     - Resource Identifiers
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
fields, database columns, REST API fields, and event data schemas.

To avoid subtle issues with performance, portability, and correctness, we must be careful to
use the correct kind of identifier building content data models or referencing content.
For example:

* Including Open edX instance-specific information in an identifier can cause content to
  break when it's transferred to other instances. For example, exporting course content with
  references to the integer primary keys of learner teams (rather than the teams' slugs)
  makes those content-team relationships invalid when transferred to any other instance.

* Including learning-context-specific information in an identifier can cause content to break
  when it's reused in another learning context. For example, when copying a component and its
  media files from course run X to course run Y, the transfer format must describe the
  component-media relationships *without* reference to the course run keys for X or Y,
  otherwise the copied component in Y may erronously try to reference media files from 
  course run X (or, we'd have to complicate the pasting code with special logic to strip
  the identifiers of reference to X, which is likely to be confusing and brittle).

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

There are five recognized categories of identifier in Open edX code. When naming an identifier,
first determine which category it belongs to, then apply the naming convention for that category. If
an identifier does not fit any category, choose a name that does not collide with any of the five
conventions, so that readers are not misled.

These conventions apply wherever identifiers are named: Python variables, parameters, and
attributes; Django model field names and database column names; REST API request and response field
names; and event data schema fields.

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
     - Locally-scoped slug-like string component of an OpaqueKey
     - ``str``
     - ``*_code``
   * - OpaqueKey
     - Parsed key object scoped to one Open edX instance
     - subclass of ``OpaqueKey``
     - * ``*_key``
   * - OpaqueKey String
     - Serialized form of an OpaqueKey
     - ``str``
     - ``*_key_string`` (or ``*_key`` when unambiguous)
   * - UUID
     - Globally unique identifier scoped across all Open edX instances
     - ``uuid.UUID``
     - ``*_uuid``
   * - UUID String
     - Serialized form of a UUID
     - ``str``
     - ``*_uuid_string`` (or ``*_uuid`` when unambiguous)

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

* Blah

**How to name**: Variables and fields holding a parsed ``OpaqueKey`` object should use the suffix
``_key``.

.. code-block:: python

   # Preferred
   course_key: CourseKey = CourseKey.from_string("course-v1:Axim+Demo+2026")
   usage_key: UsageKey = ...
   collection_key: LibraryCollectionLocator = ...

   def get_course(course_key: CourseKey) -> CourseOverview: ...

Please note that, for historical reasons, concrete OpaqueKey subclasses use the suffix ``Locator``
instead of ``Key``. For all intents and purposes, this distinction can be ignored by consumers. They
are all ``_keys``. In the future, we will unify all OpaqueKey classes to be named ``*Key``.

.. _openedx/opaque-keys: https://github.com/openedx/opaque-keys

OpaqueKey Strings
-----------------

The serialized (string) form of an OpaqueKey is distinct from the parsed object. When the context
makes it unambiguous that the value is a string—such as inside a Django form, serializer, or REST
API field—it is acceptable to use ``*_key`` for the serialized form as well.

When there is any ambiguity about whether a value is a parsed ``OpaqueKey`` object or its string
serialization, use the suffix ``*_key_string`` to make the distinction explicit.

.. code-block:: python

   # Unambiguous context (serializer field): *_key is fine
   class BlockSerializer(serializers.Serializer):
       usage_key = serializers.CharField()

   # Ambiguous context: use *_key_string to distinguish from the parsed object
   def _get_context_key_if_valid(serializer) -> LearningContextKey | None:
       usage_key_string = serializer.cleaned_data.get('usage_key')
       if not usage_key_string:
           return None
       try:
           return UsageKey.from_string(usage_key_string).context_key
       except InvalidKeyError:
           return None

Please note that OpaqueKey Strings should only be used at the boundaries of the platform (REST APIs,
external events, logging, etc.). Within the system, parsed OpaqueKey objects are always preferred,
as they protect against serialization-deserialization errors and provide type safety.

.. note::

   On the frontend, parsed ``OpaqueKey`` objects are not available; OpaqueKeys are always plain
   strings. Therefore, a ``*KeyString`` suffix is not needed in frontend code—``*Key`` is always
   acceptable.

UUIDs
=====

A UUID (Universally Unique Identifier) is a 128-bit identifier that uniquely identifies a resource
across *all* Open edX instances. Unlike primary keys and OpaqueKeys, UUIDs are not scoped to a
single database or instance.

**When to use**: Use UUIDs when you need a stable identifier that remains meaningful across Open edX
instances—for example, in cross-instance event data, external integrations, and shared databases. If
the identifier only needs to be unique within a single instance, an OpaqueKey or primary key is more
appropriate.

**How to name**: The preferred Python type for a UUID is ``uuid.UUID``. Variables and fields holding
a UUID should use the suffix ``_uuid``.

.. code-block:: python

   import uuid

   discussion_uuid: uuid.UUID = thread.uuid
   enrollment_uuid: uuid.UUID = enrollment.uuid

   def get_discussion_thread(discussion_uuid: uuid.UUID) -> DiscussionThread: ...

   class DiscussionThread(models.Model):
       uuid = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)

UUID Strings
------------

The serialized (string) form of a UUID is distinct from the ``uuid.UUID`` object. When the context
makes it unambiguous that the value is a string—such as inside a Django serializer or REST API
field—it is acceptable to use ``*_uuid`` for the serialized form as well.

When there is any ambiguity about whether a value is a ``uuid.UUID`` object or its string
serialization, use the suffix ``*_uuid_string`` to make the distinction explicit.

.. code-block:: python

   # Unambiguous context (serializer field): *_uuid is fine
   class ThreadSerializer(serializers.Serializer):
       discussion_uuid = serializers.UUIDField()

   # Ambiguous context: use *_uuid_string to distinguish from the UUID object
   def _get_thread_by_discussion(discussion_uuid_string: str) -> DiscussionThread:
       try:
           discussion_uuid = uuid.UUID(discussion_uuid_string)
       except ValueError:
           raise BadDiscussionUUID(discussion_uuid)
       return DiscussionThread.objects.get(uuid=discussion_uuid)

As with OpaqueKey Strings, UUID Strings should only be used at the boundaries of the platform.
Within the system, ``uuid.UUID`` objects are always preferred.

.. note::

   On the frontend, ``uuid.UUID`` objects are not available; UUIDs are always plain strings.
   Therefore, a ``*UuidString`` suffix is not needed in frontend code—``*Uuid`` is always
   acceptable.

Other Identifiers
=================

Not every identifier fits neatly into one of the above categories. When an identifier does not,
choose a name that does not use any of the suffixes ``_pk``, ``_key``, ``_key_string``, ``_code``,
``_uuid``, or ``_uuid_string``, so that readers are not misled into assuming a type or scope that
does not apply.

TODO mention version numbers

**When to use**: Use this guidance when you have confirmed that an identifier does not fit any of
the four named categories above. Before settling on a novel identifier type, consider whether a
primary key, OpaqueKey, code, or UUID would serve the purpose.

**How to name**: Choose a name that is evocative of the identifier's specific meaning and that does
not collide with any of the conventions above. For inspiration, consider the ``refname`` field on
``PublishableEntity`` objects in ``openedx-learning``. A ``refname`` correlates a database entity
with its representation in off-platform content archives. It is not a primary key (which would be
database-specific), not a code (because it may contain non-slug characters), and not an OpaqueKey
(because it cannot be parsed into a globally-scoped identifier). By choosing the name
``refname``—which collides with none of the conventions above—the code signals clearly that this
identifier is its own distinct thing.

Rationale
*********

The five categories above cover the vast majority of identifiers that appear in Open edX Python
code. Keeping the naming conventions to a small, well-defined set makes them easy to learn and apply
consistently. The ``_pk`` / ``_key`` / ``_code`` / ``_uuid`` suffixes were chosen because they are
short, distinct from one another, and directly evocative of the kind of identifier they represent.

The ``_key_string`` and ``_uuid_string`` conventions were chosen over alternatives like ``_key_str``
or ``_uuid_str`` for readability. The word "string" more clearly signals to a reader that the value
is a plain string rather than a Python object.

Backward Compatibility
**********************

Much existing Open edX code uses ``id`` and ``_id`` suffixes for primary keys, and some code uses
``_key`` for both parsed OpaqueKey objects and their string serializations without distinction. This
OEP does not require renaming existing identifiers; that would be a large and risky churn. New code
should follow these conventions, and existing code may be updated opportunistically during
refactors.

Reference Implementation
************************

The conventions in this OEP reflect and formalize naming patterns already in use in several Open edX
repositories, including ``openedx-learning`` and ``openedx-content-libraries``.

Rejected Alternatives
*********************

**Using** ``_id`` **for primary keys**: The ``_id`` suffix is ambiguous—it could refer to any kind
of identifier. The ``_pk`` suffix is more specific and aligns with Django's own ``pk`` attribute
name.

**Using** ``_key`` **for all string identifiers**: Overloading ``_key`` to mean "any string that
identifies something" would eliminate the useful distinction between globally-scoped OpaqueKeys,
locally-scoped codes, and serialized vs. parsed representations.

Change History
**************

2026-03-18: Initial draft.
