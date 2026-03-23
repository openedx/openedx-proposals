.. _OEP-38 Data Modeling Best Practices:

OEP-38: Data Modeling Best Practices
#####################################

+---------------+------------------------------------------------------------+
| OEP           | :ref:`OEP-0038 <OEP-38 Data Modeling Best Practices>`      |
+---------------+------------------------------------------------------------+
| Title         | Data Modeling Best Practices                               |
+---------------+------------------------------------------------------------+
| Last Modified | 2026-03-23
+---------------+------------------------------------------------------------+
| Author        | Brian Beggs <bbeggs@edx.org>                               |
+---------------+------------------------------------------------------------+
| Arbiter       | Albemarle                                                  |
+---------------+------------------------------------------------------------+
| Status        | Accepted                                                   |
+---------------+------------------------------------------------------------+
| Type          | Best Practices                                             |
+---------------+------------------------------------------------------------+
| Created       | 2019-05-01                                                 |
+---------------+------------------------------------------------------------+
| Review Period | 2019-08-01 - 2019/10/01                                    |
+---------------+------------------------------------------------------------+
| Resolution    | 'Original pull request'                                    |
+---------------+------------------------------------------------------------+

Abstract
********

In order to obtain the highest possible value from data collected in the
Open edX ecosystem, this document attempts to provide guiding thoughts and
principles on data modeling.

.. contents::

What this OEP explicitly does not cover
***************************************

-  Data warehousing

-  Data pipelines

-  Event data


Definitions
***********

**BI** - Business intelligence. Technologies, applications and practices for
the collection, integration, analysis, and presentation of business
information. The purpose of Business Intelligence is to support better
business decision making.

**Composite Key** - In the context of relational databases, a composite key
is a combination of two or more columns in a table that can be used to
uniquely identify each row in the table. Uniqueness is only guaranteed
when the columns are combined; when taken individually, the columns do
not guarantee uniqueness.

**CUD** - Create, Update, Delete. These are some of the actions that may
happen to data in a database system.

**CRUD** - Create, Read, Update, Delete. Same as CUD but with the *Read* operation added.

**Data Best Practices** - These practices are designed to help teams create
rich and efficient data models within the Open edX ecosystem. They are not
standards but guidelines to help teams think about how to store data.

**Data Dimension** - A Data Dimension is a set of data attributes pertaining
to something of interest to a business. Dimensions are things like
"customers", "products", "stores" and "time". For users of Data
Warehouses, data dimensions are entry points to numeric facts (e.g.
sale, profit, revenue) that a business wishes to monitor.

**Data Modeling** - A process used to define and analyze data requirements
needed to support the business processes within the scope of
corresponding information systems in organizations.

**Data Modeling Standards** - The most basic and standard design principles
for data modeling. These standards must be adhered to when creating new
models or updating existing models in the Open edX ecosystem.

**IDA** - Independently Deployed Application. Similar to a microservice or
stand alone application.


Open edX Ecosystem
==================

Used in this document, the phrase "Open edX Ecosystem" includes the
services that comprise and support the running of an Open edX
installation (e.g.,. edx.org). This includes:

- The LMS and Studio

- IDAs such as Notes, Ecommerce, and Forums

- Backend systems such as data analytics and operational logging

- Third party services used to extend functionality of applications, such as Optimizely, Google Analytics, and Sailthru


Specification
*************

Guiding Principles
==================
This section is intended to give the reader a framework to think about data modeling at edX.

Data is an asset
----------------

edX uses data to decide which people should receive marketing emails,
who passes or fails a course, or how much to pay our partners. Our
partners use our data to target users and to
refine and improve their courses. The data we collect today is being
used to advance academic research about online learning and pedagogy.
Decision making at edX should be data-driven and based on this collected data.

This data is one of our most valuable assets and it
should be a first-order concern. Save everything (disk is cheap). The more data we are able to collect about users,
behavior, and system state now, the more opportunities we’ll have to
improve our decision-making in the future.

Use industry best practices
---------------------------

Data stored in the OpenEdX ecosystem should adhere to industry best
practices. For example, since an industry best practice uses numerical
identifiers to identify rows of data, the Open edX ecosystem should also use
numerical identifiers. Adhering to industry best practices and the
practices outlined in this document ensures our data is approachable for
experienced engineers and new hires alike and allows us to leverage 3rd
party tools more easily to assist in the analysis phase.

Don’t make me think
-------------------

Steve Krug, the author of “Don’t Make Me Think”, says: “Your objective
should always be to eliminate instructions entirely by making everything
self-explanatory, or as close to it as possible.” This principle should
also be applied to data modeling. Data models should have descriptive
names clearly identifying the data that the model holds. The field names
should make sense to most people familiar with the domain, without
needing much clarifying documentation.

Consider the analytics user
---------------------------

For every hour of engineer-time spent creating a data model, many
more are spent using the model and analyzing its data. Taking
the time to consider how the data will be used, and thoughtfully design
a data representation, can potentially save many hours during the
analysis phase.

This is especially true since changing data models, once they are in
use, can be time-consuming. A data model change in a core application
can affect many other systems. Small changes to a data model could
possibly cause hours or days of work for different teams throughout the
organization. An example of the types of work that may occur while changing a model:

-  Data engineering and Analytics to update their workflows

-  BI teams to update reports that utilize this data,

-  Devops to deploy migrations to large tables, with the risk of
      potential downtime.

Due to the potentially high cost of changing a model, it pays to get it
as correct as possible the first time.

Think about the person who will analyze this data later and the person
who manages the system day to day. (That person could possibly be future
you!) Try to do as much careful design up front to make your quality of
life better later.

History is best (but events are a close second)
-----------------------------------------------

Historical data accuracy is best when history about a change is captured at the database level. Changes in data relating to financial systems (e.g. enrollments, payments, course price changes) should be stored historically where the change is made, preferably in the same system as the system of record. This allows us to reconstruct the data at different points with much greater reliability.

If, for some reason, it is not possible to create history at the time of
entry (for example, if we expect the table to become too large or the
writes to be too frequent), it is recommended that an event be issued in its
place.


Data Modeling Standards
=======================

The standards below are designed to ensure edX can gain the highest value and insights from the data.
The application of these standards is the most basic level of support to which all applications in the Open edX ecosystem
should adhere. When creating new applications or models please ensure the models being created conform to the following.

Use a numeric primary key
-------------------------

-  It is recommended to use `BigAutoField`_.

.. _BigAutoField: https://docs.djangoproject.com/en/2.2/ref/models/fields/#bigautofield

-  Do not use composite based primary keys. Use a primary key column.

Have updated and created timestamps
-----------------------------------

-  The preferred method for doing this in Open edX Django applications is to inherit the `TimeStampedModel`_ class.

.. _TimeStampedModel: https://django-model-utils.readthedocs.io/en/latest/models.html#timestampedmodel

- Time should be stored in UTC time by setting `USE_TZ=True`_ in your python config.

.. _USE_TZ=True: https://docs.djangoproject.com/en/2.2/topics/i18n/timezones/#overview

-  If for some reason you can not inherit from `TimeStampedModel`_ use the following naming conventions:

   -  Created date should be named: “created”

   -  Updated date should be named: “modified”

Foreign keys
--------------------------

-  Data should be joined using primary keys wherever possible

-  Foreign keys should use a naming convention of :code:`<object_name>_id` where object name is the name of the table of the foreign key relationship.

-  Do not join on things such as username, email address, or other dimensions of data that may change over time

-  Do not join on PII

- Joining between IDAs should be done by using a universally unique identifier (UUID)

- In Django use `Attributes for fields with relations`_ to identify and link models with relationships.

.. _Attributes for fields with relations: https://docs.djangoproject.com/en/2.2/ref/models/fields/#module-django.db.models.fields.related

History for models involved with enrollments, courses and course metadata, or data involving financial payments and transactions
--------------------------------------------------------------------------------------------------------------------------------

-  History using django-simple-history_.

.. _django-simple-history: https://django-simple-history.readthedocs.io/en/latest/

-  Remember to `backfill history`_ for existing models.

.. _backfill history: https://django-simple-history.readthedocs.io/en/latest/quick_start.html#existing-projects

-  Where Django simple history is not an option, the following data should be captured:

   -  Fields that were changed

   -  Date & time of the change

   -  The foreign key of the user who initiated the change

Use the correct data type for a column
--------------------------------------

-  Don’t use a :code:`IntegerField` when a :code:`BooleanField` would do.

-  Use :code:`BigIntegerField` for your foreign keys

-  Don’t store an Integer field as `CharField`_.

.. _CharField: https://docs.djangoproject.com/en/2.2/ref/models/fields/#charfield

- Store UUID's as `UUIDField`_ with a max length that matches the max length of the UUID.

.. _UUIDField: https://docs.djangoproject.com/en/2.2/ref/models/fields/#uuidfield


Each column in a table should only store a single fact or dimension
-------------------------------------------------------------------

- If a column could be a mix of integer data and character data it is best to store these items as 2 different fields in the database

Models should be Annotated for PII
----------------------------------

-  All models in the Open edX ecosystem should be tagged for PII using `code annotations`_ by following OEP-30_.

.. _code annotations: https://github.com/openedx/code-annotations>
.. _OEP-30: https://github.com/openedx/openedx-proposals/blob/master/oeps/oep-0030-arch-pii-markup-and-auditing.rst


Have sane default values
------------------------

- A model should have default values that make sense for the application

- `CharField`_ should be defined with the `null=True`_ option.

.. _null=True: https://docs.djangoproject.com/en/2.2/ref/models/fields/#null

   - This is to improve post processing. Defaulting `CharField`_ to null enables us to better derive the intent of the user. If the field is null no intent was made to enter that field. If the field is blank a string was entered and was modified later by the user.

- For example if you are adding a boolean to flag that a learner has not yet activated their account, the default value should be set to False, not None.

The database layer should preserve uniqueness
-----------------------------------------------

- If a model needs to preserve uniqueness between many fields use `unique_together`_.

.. _unique_together: https://docs.djangoproject.com/en/2.2/ref/models/options/#unique-together

Best Practices
==============

These practices are designed to help teams create rich and efficient data models within the OpenedX ecosystem.
They are not standards but guidelines to help teams think about how they are storing data.

Columns should store only exactly what the name describes
---------------------------------------------------------

- The name of a column in a table should accurately describe the data in that table.

- If a column is named ``course_id`` it should only store the course's primary key.
  Not a string course run key, not a UUID. For more details on naming content identifiers,
  see :ref:`OEP-68 Learning Content Identifiers`_.

Deleting data
-------------

-  It is better to have a column to mark the record as inactive than to remove the data from the system using the SQL delete keyword. These models should use Django's `SoftDeletableModel`_.

.. _SoftDeletableModel: https://django-model-utils.readthedocs.io/en/latest/models.html#softdeletablemodel

-  Please note that GDPR may require that data be deleted. If a field is determined to contain PII and falls under the realm of GDPR, that data should be deleted from the system, or obfuscated from the system. `For more information about GDPR and how to delete user data from edx please refer to this documentation`_.

.. _For more information about GDPR and how to delete user data from edx please refer to this documentation: https://openedx.atlassian.net/wiki/spaces/PLAT/pages/930021733/User+Retirement+Tutorial+for+Developers

Don’t trap the data
-------------------

-  Each piece of information should have its own column. Avoid storing
      data in blob fields or as JSON in the database.

- Another example is a concatenated string with a separator. It is best to treat these data items in 2 distinct fields.

-  Don’t store encoded (pickle, json, other) objects in the database! If
      you need to run the python environment to decode the data, analysts who use SQL will have a difficult time querying and decoding this data.

Store everything
----------------

-  Storage is cheap!

-  If you are unsure whether you should store something in the database or add history the answer is almost always yes. Store the data and add history. It can be removed later if it is found unnecessary.

-  Still not sure? The default answer is yes.

Use methods on models to access data
------------------------------------

- CRUD operations should access models via methods on models (where they exist), instead of querying managers directly.

-  For example, prefer creating something like
      CourseEnrollment.is_enrolled(...) rather than having views check
      CourseEnrollment.objects.filter(...).exists().

-  This allows us to more easily make internal representation changes
      like switching to using a “deleted” flag instead of deleting the
      row.

-  This also reduces the likelihood that people will query models in a
      non-performant way (e.g. sorting by an unindexed field).

Enforce logical constraints at the database layer
-------------------------------------------------

-  Don’t allow impossible states to be represented in the database.

-  If your code expects a 1:1 relationship, use Django's `Unique`_ instead of trying to enforce the constraint in
      Python.

.. _Unique: https://docs.djangoproject.com/en/2.2/ref/models/fields/#unique

-  Python will not save you from race conditions. Database constraints will.

-  For example, an enrollment should have a unique constraint on
      (course_id, user_id), since a given user should only have one
      enrollment per course. In this case you should use Django's `unique_together`_.

Keep indexes to a minimum
-------------------------

-  Create indexes only on the fields necessary to make queries performant

-  Keep in mind that indexes cost space and have their own set of performance concerns.

-  Over-indexing data could actually make the database less performant (slower writes/updates)

Developer Responsibility
************************

It is the responsibility of the developer to adhere to all of the
standards in the Data Modeling Standards section of this document.

Code Reviewer Responsibility
****************************

The code reviewer is responsible for ensuring the standards set forth in
the Data Modeling Standards section of this document are met.

Responsibility for Third-party Service Integrations
***************************************************

Adhere to the same standards.

Backward Compatibility
**********************

Data models that are not within the standards of this document do not need to be updated to adhere to OEP-38 standards.

Change History
**************

2026-03-23
==========

* Add link to :ref:`OEP-68 Learning Content Identifiers`.
* `Pull request #773 <https://github.com/openedx/openedx-proposals/pull/773>`_

2019-05-01
==========

* Initial proposal
* `Pull request #119 <https://github.com/openedx/openedx-proposals/pull/119>`_