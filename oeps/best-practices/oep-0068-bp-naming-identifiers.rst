.. _OEP-68 Naming Identifiers:

OEP-68: Naming Identifiers
###########################

.. list-table::
   :widths: 25 75

   * - OEP
     - :ref:`OEP-0068 <OEP-68 Naming Identifiers>`
   * - Title
     - Naming Identifiers
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

Open edX code uses several distinct types of identifiers. To make code unambiguous and readable, we adopt consistent naming conventions for each type. This OEP defines four categories of identifiers—Primary Keys, OpaqueKeys, Codes, and OpaqueKey Strings—and specifies the naming conventions for each. These conventions apply across Python code, database column names, REST API fields, event schemas, and any other context where identifiers appear.

Motivation
**********

Identifiers are ubiquitous: they appear as Python variables, function parameters, Django model fields, database columns, REST API fields, and event data schemas. Without consistent naming, it is easy to confuse different kinds of identifiers. For example, does ``course_id`` hold an integer primary key, a parsed ``OpaqueKey`` object, or a serialized OpaqueKey string? That ambiguity causes bugs—such as passing a string where an object is expected—and makes code harder to read and review.

It is also common for the same conceptual entity to be identified by more than one kind of identifier simultaneously. A library collection, for instance, might be referenced by both a database primary key (``collection_pk``) and a globally-scoped OpaqueKey (``collection_key``). Without a convention, it is easy to conflate these or pass the wrong one to a function.

Consistent naming makes identifier types immediately apparent from the name alone, reduces the need for comments or type annotations to disambiguate, and helps developers quickly spot likely bugs at a glance.

Specification
*************

There are four recognized categories of identifier in Open edX code. When naming an identifier, first determine which category it belongs to, then apply the naming convention for that category. If an identifier does not fit any category, choose a name that does not collide with any of the four conventions, so that readers are not misled.

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
   * - OpaqueKey
     - Globally-scoped parsed key object
     - subclass of ``OpaqueKey``
     - ``*_key``
   * - OpaqueKey String
     - Serialized form of an OpaqueKey
     - ``str``
     - ``*_key_string`` (or ``*_key`` when unambiguous)
   * - Code
     - Locally-scoped slug-like string
     - ``str``
     - ``*_code``

Primary Keys
============

Every Django model uses ``django.db.models.BigAutoField`` as its primary key. This is an auto-incremented integer assigned by the database and meaningful only within that database; it should not be used as a stable cross-service or cross-instance identifier.

The preferred name for a primary key is ``pk`` (for the key of the current model) or ``<entity>_pk`` for foreign references. The suffix ``_pk`` makes it immediately clear that the value is a database-internal integer, not a globally meaningful identifier.

.. code-block:: python

   # Preferred
   user_pk = user.pk
   collection_pk = collection.pk

   def get_block(block_pk: int) -> LibraryBlock: ...

   class Membership(models.Model):
       user_pk = models.BigIntegerField()  # FK stored as raw int
       collection = models.ForeignKey(Collection, db_column="collection_pk", ...)

The names ``id`` and ``<entity>_id`` are historically common in Open edX code and remain acceptable, but ``pk`` and ``<entity>_pk`` are preferred for new code.

OpaqueKeys
==========

An ``OpaqueKey`` (defined in `openedx/opaque-keys`_) is a parsed Python object that globally and stably identifies a resource across an entire Open edX instance. OpaqueKeys are serialized with a type prefix followed by a colon (e.g. ``course-v1:``, ``lb:``) and are meaningfully exchangeable across services and instances.

Variables and fields holding a parsed ``OpaqueKey`` object should use the suffix ``_key``.

.. code-block:: python

   # Preferred
   course_key: CourseKey = CourseKey.from_string("course-v1:Axim+Demo+2026")
   usage_key: UsageKey = ...
   collection_key: LibraryCollectionLocator = ...

   def get_course(course_key: CourseKey) -> CourseOverview: ...

.. _openedx/opaque-keys: https://github.com/openedx/opaque-keys

OpaqueKey Strings
=================

The serialized (string) form of an OpaqueKey is distinct from the parsed object. When the context makes it unambiguous that the value is a string—such as inside a Django form, serializer, or REST API field—it is acceptable to use ``*_key`` for the serialized form as well.

When there is any ambiguity about whether a value is a parsed ``OpaqueKey`` object or its string serialization, use the suffix ``*_key_string`` to make the distinction explicit.

.. code-block:: python

   # Unambiguous context (serializer field): *_key is fine
   class BlockSerializer(serializers.Serializer):
       usage_key = serializers.CharField()

   # Ambiguous context: use *_key_string to distinguish from the parsed object
   def resolve_usage_key(usage_key_string: str) -> UsageKey:
       return UsageKey.from_string(usage_key_string)

Codes
=====

A code is a short, locally-scoped string identifier. Codes follow slug rules: they consist only of alphanumeric characters, hyphens, and underscores (equivalent to Django's ``SlugField``). They are case-sensitive. Unlike OpaqueKeys, codes are not globally unique on their own; they identify a resource or aspect of a resource *within some enclosing context*.

Codes are the building blocks of OpaqueKeys. For example, the serialized ``UsageKey`` ``"lb:Axim:ChemLib:problem:Atoms6"`` is composed of the prefix ``lb:`` and the following codes:

.. code-block:: python

   org_code = "Axim"
   library_code = "ChemLib"
   type_code = "problem"
   block_code = "Atoms6"

Variables and fields holding a code should use the suffix ``_code``.

.. code-block:: python

   def get_library(org_code: str, library_code: str) -> ContentLibrary: ...

Other Identifiers
=================

Not every identifier fits neatly into one of the above categories. When an identifier does not, choose a name that does not use any of the suffixes ``_pk``, ``_key``, ``_key_string``, or ``_code``, so that readers are not misled into assuming a type or scope that does not apply.

For inspiration, consider the ``refname`` field on ``PublishableEntity`` objects in ``openedx-learning``. A ``refname`` correlates a database entity with its representation in off-platform or cross-platform ZIP archives. It is not a primary key (which would be database-specific), not a code (because it may contain non-slug characters), and not an OpaqueKey (because it cannot be parsed into a globally-scoped identifier). By choosing the name ``refname``—which collides with none of the conventions above—the code signals clearly that this identifier is its own distinct thing.

Rationale
*********

The four categories above cover the vast majority of identifiers that appear in Open edX Python code. Keeping the naming conventions to a small, well-defined set makes them easy to learn and apply consistently. The ``_pk`` / ``_key`` / ``_code`` suffixes were chosen because they are short, distinct from one another, and directly evocative of the kind of identifier they represent.

The ``_key_string`` convention for serialized OpaqueKeys was chosen over alternatives like ``_key_str`` or ``_key_serialized`` for readability. The word "string" more clearly signals to a reader that the value is a plain string rather than a Python object.

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
