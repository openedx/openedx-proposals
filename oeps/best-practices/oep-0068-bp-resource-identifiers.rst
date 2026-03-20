.. _OEP-68 Resource Identifiers:

OEP-68: Resource Identifiers
##############################

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

Open edX code uses several distinct types of resource identifiers. To make code unambiguous and readable, we adopt consistent naming conventions for each type, and provide guidance on when each type should be used. This OEP defines five categories of identifiers—Primary Keys, Codes, OpaqueKeys, UUIDs, and Other—and specifies the naming conventions for each. These conventions apply across Python code, database column names, REST API fields, event schemas, and any other context where identifiers appear.

Motivation
**********

Identifiers are ubiquitous: they appear as Python variables, function parameters, Django model fields, database columns, REST API fields, and event data schemas. Without consistent naming, it is easy to confuse different kinds of identifiers. For example, does ``course_id`` hold an integer primary key, a parsed ``OpaqueKey`` object, or a serialized OpaqueKey string? That ambiguity causes bugs—such as passing a string where an object is expected—and makes code harder to read and review.

It is also common for the same conceptual entity to be identified by more than one kind of identifier simultaneously. A library collection, for instance, might be referenced by both a database primary key (``collection_pk``) and a globally-scoped OpaqueKey (``collection_key``). Without a convention, it is easy to conflate these or pass the wrong one to a function.

Consistent naming makes identifier types immediately apparent from the name alone, reduces the need for comments or type annotations to disambiguate, and helps developers quickly spot likely bugs at a glance.

Specification
*************

There are five recognized categories of identifier in Open edX code. When naming an identifier, first determine which category it belongs to, then apply the naming convention for that category. If an identifier does not fit any category, choose a name that does not collide with any of the five conventions, so that readers are not misled.

These conventions apply wherever identifiers are named: Python variables, parameters, and attributes; Django model field names and database column names; REST API request and response field names; and event data schema fields.

Summary
=======

.. list-table::
   :header-rows: 1
   :widths: 20 30 25 25

   * - Category
     - What it is
     - Python type
     - Naming convention
   * - Primary Key
     - Auto-incremented integer row identifier
     - ``int``
     - ``pk``, ``*_pk``
   * - Code
     - Locally-scoped slug-like string component of an OpaqueKey
     - ``str``
     - ``*_code``
   * - OpaqueKey
     - Parsed key object scoped to one Open edX instance
     - subclass of ``OpaqueKey``
     - ``*_key``
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

Primary Keys
============

Every Django model uses ``django.db.models.BigAutoField`` as its primary key. This is an auto-incremented integer assigned by the database and meaningful only within that database; it is not a stable cross-service or cross-instance identifier.

**When to use**: Primary keys are the default choice for referencing database rows within a service. Always use primary keys for foreign key relationships on Django models. When query efficiency is a concern, prefer primary keys over OpaqueKeys. However, primary keys are highly instance-specific—a given integer has no meaning outside the database that assigned it—so when identifiers need to be meaningful beyond a single instance, an OpaqueKey or UUID is more appropriate.

**How to name**: The preferred name for a primary key is ``pk`` (for the key of the current model) or ``<entity>_pk`` for foreign references. The suffix ``_pk`` makes it immediately clear that the value is a database-internal integer, not a globally meaningful identifier.

.. code-block:: python

   # Preferred
   user_pk = user.pk
   collection_pk = collection.pk

   def get_block(block_pk: int) -> LibraryBlock: ...

   class Membership(models.Model):
       user_pk = models.BigIntegerField()  # FK stored as raw int
       collection = models.ForeignKey(Collection, db_column="collection_pk", ...)

The names ``id`` and ``<entity>_id`` are historically common in Open edX code and remain acceptable, but ``pk`` and ``<entity>_pk`` are preferred for new code.

Codes
=====

A code is a short, locally-scoped string identifier. Codes follow slug rules: they consist only of alphanumeric characters, hyphens, and underscores (equivalent to Django's ``SlugField``). They are case-sensitive. A code identifies a resource or aspect of a resource *within some enclosing context*—it is not globally unique on its own.

For example: a ``block_code`` and ``type_code`` together identify a block within a course run; a ``run_code`` and ``course_code`` together identify a course run within an organization. Codes are also the identifiers used in off-platform content archives, which cannot rely on foreign keys.

**When to use**: Codes almost always exist as components that are composed into an OpaqueKey. If you find yourself reaching for a code-like identifier that is *not* destined to become part of an OpaqueKey, consider whether one of the other identifier types is more appropriate.

**How to name**: Variables and fields holding a code should use the suffix ``_code``.

.. code-block:: python

   org_code = "Axim"
   library_code = "ChemLib"
   type_code = "problem"
   block_code = "Atoms6"

   def get_library(org_code: str, library_code: str) -> ContentLibrary: ...

OpaqueKeys
==========

An ``OpaqueKey`` (defined in `openedx/opaque-keys`_) is a parsed Python object formed by joining a set of codes with a prefix that identifies the resource type (e.g. ``lb:`` for library blocks, ``course-v1:`` for course runs). This structure gives OpaqueKeys a coherency that primary keys lack: because an OpaqueKey is deterministic from its component codes, two separate Open edX instances that share the same codes for a resource will produce the same OpaqueKey for it.

**When to use**: When choosing between a primary key and an OpaqueKey for the same resource, consider two things. First, primary keys are highly instance-specific—a given integer has no meaning outside the database that assigned it—whereas OpaqueKeys have internal coherency that makes them meaningful across instances. Second, OpaqueKeys are more readable than primary keys, making them better suited for logs, URLs, admin views, reporting, events, REST APIs, and advanced user interfaces. For database queries and foreign key relationships, prefer primary keys for efficiency.

**How to name**: Variables and fields holding a parsed ``OpaqueKey`` object should use the suffix ``_key``.

.. code-block:: python

   # Preferred
   course_key: CourseKey = CourseKey.from_string("course-v1:Axim+Demo+2026")
   usage_key: UsageKey = ...
   collection_key: LibraryCollectionLocator = ...

   def get_course(course_key: CourseKey) -> CourseOverview: ...

Please note that, for historical reasons, concrete OpaqueKey subclasses use the suffix ``Locator`` instead of ``Key``. For all intents and purposes, this distinction can be ignored by consumers. They are all ``_keys``. In the future, we will unify all OpaqueKey classes to be named ``*Key``.

.. _openedx/opaque-keys: https://github.com/openedx/opaque-keys

OpaqueKey Strings
-----------------

The serialized (string) form of an OpaqueKey is distinct from the parsed object. When the context makes it unambiguous that the value is a string—such as inside a Django form, serializer, or REST API field—it is acceptable to use ``*_key`` for the serialized form as well.

When there is any ambiguity about whether a value is a parsed ``OpaqueKey`` object or its string serialization, use the suffix ``*_key_string`` to make the distinction explicit.

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

Please note that OpaqueKey Strings should only be used at the boundaries of the platform (REST APIs, external events, logging, etc.). Within the system, parsed OpaqueKey objects are always preferred, as they protect against serialization-deserialization errors and provide type safety.

.. note::

   On the frontend, parsed ``OpaqueKey`` objects are not available; OpaqueKeys are always plain strings. Therefore, a ``*KeyString`` suffix is not needed in frontend code—``*Key`` is always acceptable.

UUIDs
=====

A UUID (Universally Unique Identifier) is a 128-bit identifier that uniquely identifies a resource across *all* Open edX instances. Unlike primary keys and OpaqueKeys, UUIDs are not scoped to a single database or instance.

**When to use**: Use UUIDs when you need a stable identifier that remains meaningful across Open edX instances—for example, in cross-instance event data, external integrations, and shared databases. If the identifier only needs to be unique within a single instance, an OpaqueKey or primary key is more appropriate.

**How to name**: The preferred Python type for a UUID is ``uuid.UUID``. Variables and fields holding a UUID should use the suffix ``_uuid``.

.. code-block:: python

   import uuid

   discussion_uuid: uuid.UUID = thread.uuid
   enrollment_uuid: uuid.UUID = enrollment.uuid

   def get_discussion_thread(discussion_uuid: uuid.UUID) -> DiscussionThread: ...

   class DiscussionThread(models.Model):
       uuid = models.UUIDField(default=uuid.uuid4, editable=False, unique=True)

UUID Strings
------------

The serialized (string) form of a UUID is distinct from the ``uuid.UUID`` object. When the context makes it unambiguous that the value is a string—such as inside a Django serializer or REST API field—it is acceptable to use ``*_uuid`` for the serialized form as well.

When there is any ambiguity about whether a value is a ``uuid.UUID`` object or its string serialization, use the suffix ``*_uuid_string`` to make the distinction explicit.

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

As with OpaqueKey Strings, UUID Strings should only be used at the boundaries of the platform. Within the system, ``uuid.UUID`` objects are always preferred.

.. note::

   On the frontend, ``uuid.UUID`` objects are not available; UUIDs are always plain strings. Therefore, a ``*UuidString`` suffix is not needed in frontend code—``*Uuid`` is always acceptable.

Other Identifiers
=================

Not every identifier fits neatly into one of the above categories. When an identifier does not, choose a name that does not use any of the suffixes ``_pk``, ``_key``, ``_key_string``, ``_code``, ``_uuid``, or ``_uuid_string``, so that readers are not misled into assuming a type or scope that does not apply.

**When to use**: Use this guidance when you have confirmed that an identifier does not fit any of the four named categories above. Before settling on a novel identifier type, consider whether a primary key, OpaqueKey, code, or UUID would serve the purpose.

**How to name**: Choose a name that is evocative of the identifier's specific meaning and that does not collide with any of the conventions above. For inspiration, consider the ``refname`` field on ``PublishableEntity`` objects in ``openedx-learning``. A ``refname`` correlates a database entity with its representation in off-platform content archives. It is not a primary key (which would be database-specific), not a code (because it may contain non-slug characters), and not an OpaqueKey (because it cannot be parsed into a globally-scoped identifier). By choosing the name ``refname``—which collides with none of the conventions above—the code signals clearly that this identifier is its own distinct thing.

Rationale
*********

The five categories above cover the vast majority of identifiers that appear in Open edX Python code. Keeping the naming conventions to a small, well-defined set makes them easy to learn and apply consistently. The ``_pk`` / ``_key`` / ``_code`` / ``_uuid`` suffixes were chosen because they are short, distinct from one another, and directly evocative of the kind of identifier they represent.

The ``_key_string`` and ``_uuid_string`` conventions were chosen over alternatives like ``_key_str`` or ``_uuid_str`` for readability. The word "string" more clearly signals to a reader that the value is a plain string rather than a Python object.

Backward Compatibility
**********************

Much existing Open edX code uses ``id`` and ``_id`` suffixes for primary keys, and some code uses ``_key`` for both parsed OpaqueKey objects and their string serializations without distinction. This OEP does not require renaming existing identifiers; that would be a large and risky churn. New code should follow these conventions, and existing code may be updated opportunistically during refactors.

Reference Implementation
************************

The conventions in this OEP reflect and formalize naming patterns already in use in several Open edX repositories, including ``openedx-learning`` and ``openedx-content-libraries``.

Rejected Alternatives
*********************

**Using** ``_id`` **for primary keys**: The ``_id`` suffix is ambiguous—it could refer to any kind of identifier. The ``_pk`` suffix is more specific and aligns with Django's own ``pk`` attribute name.

**Using** ``_key`` **for all string identifiers**: Overloading ``_key`` to mean "any string that identifies something" would eliminate the useful distinction between globally-scoped OpaqueKeys, locally-scoped codes, and serialized vs. parsed representations.

Change History
**************

2026-03-18: Initial draft.
