:tocdepth: 1

.. sectnum::


Abstract
========

Alert Production Database (APDB) schema will likely need updates during its lifetime.
This technical note discusses possible issues and approaches for implementing schema versioning and migration tools.


Introduction
============

Alert Production Database (APDB) schema is relatively simple but like any other piece of software it is expected to evolve over time.
Some aspects of the schema are controlled by its description in `sdm_schemas`_ package, that description provides table and column declarations for science payload.
Other aspects can be controlled by APDB code which can add additional entities to the schema necessary for the implementation.
Both of these aspects are likely to evolve over time, which makes it necessary to support schema versioning and schema migration for APDB and its related PPDB instance.

Versioning and version compatibility is a complicated issue, it is unlikely that we can build an ideal solution for it from very beginning and it will need more than one iteration.
Here I try to outline issues that are already known and options for solving them based on experience with other parts of Rubin Data Management software.


Schema construction and versioning
==================================

The actual schema created when APDB is initialized is a combination of the definition in `sdm_schemas`_ (in `yml/apdb.yaml`) and what the APDB software implementation adds to those definitions.
Each of these two pieces can evolve separately and change over time, changing the resulting database schema.
Presently there are two implementations of APDB, one is based on relational databases and the ``sqlalchemy`` package, and another uses ``Cassandra`` as a storage backend.
These two implementations may use different mappings of the definitions in ``sdm_schemas`` to actual database schema, and their code can evolve independently.
The database schema thus depends on the version of the ``sdm_schemas`` definitions and the version of the code used to generate the schema.
The schema and code versions may also change independently, though there may be cases when new ``sdm_schemas`` definitions also require code updates (e.g. to handle new types or data transformations).
While the code that creates database schema does it based on the ``sdm_schemas`` description, reversing that process and reconstructing the schema description from a database is not always possible.

The code that works with the existing database needs to be compatible with the schema in the database.
In practical terms it is impossible to examine a given database schema and decide whether it is compatible with the current schema description and APDB code.
There needs to be a separate mechanism to identify versions of the schema description and the code and compare those against the corresponding versions used to create the database.

APDB, of course, is not unique in this respect, practically every project using databases has to handle schema migration sooner or later.
Many such projects operate in a controlled environment where software version upgrades are always tied to database schema upgrades, making compatibility issues rather trivial.
APDB will likely face more complex issues as it one schema definition can potentially run with multiple software releases, and database schema upgrades may need to be delayed for a variety of reasons.
In such conditions it is necessary to implement rules for schema compatibility to allow multiple software versions to operate concurrently if schema changes allow it.


Version compatibility
---------------------

Checking versions compatibility and supporting software compatibility across multiple versions is not a trivial problem.
One popular approach for expressing whether two versions are compatible consists in using the `Semantic Versioning`_ convention for version numbers, usually used to check API compatibility.
In this approach, the version of the software is expressed as a ``MAJOR.MINOR.PATCH`` triplet with defined compatibility rules:

- versions that differ in ``MAJOR`` number are not compatible,
- versions that differ in ``MINOR`` number are backward-compatible,
- versions that differ in ``PATCH`` number only are fully compatible.

This compatibility scheme works is sufficient (if implemented correctly) for API-level versioning, but for things like database schemas or wire-level protocols that deal with physical data layouts it is much harder to use.
Frequently, software that has to support database schema migration avoids these issues by making any schema changes fully incompatible and using just a single number (or random string) to express schema version.


Schema versioning in Rubin DM packages
--------------------------------------

I am aware of two packages in DM software that implement database schema versioning.

First is ``QServ`` which has few internal database schemas that need occasional updates.
Because these internal databases are never shared across multiple ``QServ`` instances, they do not need to care about version compatibility.
``QServ`` implements a special procedure that migrates databases schemas as necessary on a deployment of a new ``QServ`` version on an existing database.
This procedure uses home-built version management system called `smig`_ which uses hand-written SQL or Python migration scripts.
As there are no compatibility concerns, `smig`_ uses sequential integer numbers for schema versions.
There are several schemas (meaning a set of tables as opposed to actual database schema) in ``QServ`` managed by this mechanism; each schema has its own version number and a set of migration classes/scripts.
The current database schema version is recorded in the database in a format which is schema specific, usually in a separate small table which keeps metadata for the corresponding schema.

The second package is ``daf_butler`` and it has more elaborate versioning scheme.
There could be multiple Rubin releases accessing the same Butler registry database, so it is essential to support a reasonable compatibility between releases.
The butler database has a complicated schema split into several domains controlled by multiple modules/classes (also called *managers*).
For each manager we define a version number which has the format of a semantic version (*MAJOR.MINOR.PATCH*); the version number is defined in the manager code.
When schema for a manager is created or upgraded, its current version is stored in a special table with a name ``butler_attributes`` which also records other metadata in a key-value format.
Compared to regular semantic versioning, versions in ``daf_butler`` have somewhat different compatibility rules:

- versions that differ in ``MAJOR`` number are not compatible, just like in semantic versioning,
- versions that differ in ``MINOR`` number are backward-compatible, but only for reading, and not compatible for write access,
- versions that differ in ``PATCH`` number only are fully compatible.

Schema upgrade tools for Butler database are implemented on top of `Alembic`_ which is based on ``SQLAlchemy``.
As Butler schema is more complicated than a typical Alembic-managed database and has non-trivial migration paths, a special Alembic-based tool `daf_butler_migrate`_ was developed to support schema migrations.
`DMTN-191`_ provides additional information about schema migration for Butler database.


Schema versioning for APDB
==========================

To handle schema versioning for APDB it needs a mechanism to identify and record versions and the tools to check version compatibility and perform schema migrations for existing databases.

As mentioned above, the APDB schema is a product of two entities - ``sdm_schemas`` definitions and ``dax_apdb`` code - each of these two can evolve independently.
To identify the database schema version, the versions of both ``sdm_schemas`` and ``dax_apdb`` have to be known.
Presently ``sdm_schemas`` (or rather `felis`_) does not provide a way to define or access version numbers for the schema that it defines, so it will have to be extended to allow specification of the version number, and possibly to include some form of compatibility rules.
Similarly, ``dax_apdb`` does not provide a way to specify its code version, it has to be extended as well to include that information (independently for SQL and Cassandra implementations).


Version recording
-----------------

The database needs to record the versions with which it was created (or later upgraded).
One common approach for this is to define a separate metadata table that can record various additional information about the database itself.
This table can be a simple key-value storage with two string columns, indexed by a key value; this approach is used for ``daf_butler`` schema management tools.
Both versions used to create database schema will be recorded in the metadata table, one possible example choosing names for keys could be::

    key                           | value
   -------------------------------+----------
    version:sdm_schemas/apdb.yaml | 1.2.0
    version:dax_apdb/ApdbSql      | 2.0.1

Addition of this metadata table to the existing databases would be a schema change in itself, which could probably be managed by the migration tools.


Schema migration tools
----------------------

Schema upgrades for existing databases will require a migration tool which will know about all existing versions of the schema and corresponding scripts to migrate from one version to later versions.
Different backends (SQL and Cassandra) may share some of the migration tool functionality but will likely have a different set of migration scripts.
For SQL-based implementation it is natural to use `Alembic`_ as it solves many of the migration issues.
`Alembic`_ cannot be used directly with Cassandra and it is unlikely that it can be adapted for use with Cassandra with a reasonable effort, more likely Cassandra backend will need a separate tooling.

Many ideas from `daf_butler_migrate`_ can be reused for implementing APDB migration tool, including command line interface and organization of the Alembic migration scripts.
Cassandra-specific tooling can be added to the same command-line tool to provide a uniform interface, if possible.


Implementation path
===================

If the above model looks reasonable then an implementation plan for adding schema versions to APDB could look like:


- Extend ``felis`` to support versions in schema definitions as a schema-level ``version`` key.
  These versions can be any strings, ``felis`` is not going to interpret them.
  We could use semantic version numbers for APDB schemas or provide more explicit specification of compatibility in ``felis`` (e.g. ``compatible_versions`` key).

- Add a starting version number to ``apdb.yaml``, ``0.0.1`` may be a good start, leaving ``0.0.0`` as a placeholder for pre-metadata version.

- Add a starting version numbers to ``ApdbSql`` and ``ApdbCassandra`` classes.
  Extend these classes with the option of reading stored version numbers from a metadata table, if that exists, and check compatibility.
  APDB code could assume that missing metadata table in the schema means version ``0.0.0`` for both schema and code version.
  Add an interface of reading/writing key/value pairs to metadata table.

- Define the format of the version numbers and rules of their compatibility.
  A couple of possible option, as already mentioned above, could be:

  - Use semantic versioning, difference in major version means incompatible versions.
    Difference in patch version mans completely compatible version.
    Difference in minor version could mean backward compatibility (maybe for reading only).
    Incompatibility should result in exception, compatible versions should not produce any diagnostics.

  - Version could be a single consecutive number (1, 2, 3, and so on) and special rules should be defined in the schema and code to express compatibility.
    For example, ``apdb.yaml`` could specify its current version as ``7`` and also specify that it is fully compatible wit version ``6`` and read-compatible with versions ``5`` and ``4``.

- Implement migration tool, borrowing some ideas from ``daf_butler_migrate``.
  Implement first migration as adding metadata table and populating it with the current version numbers.



.. _sdm_schemas: https://github.com/lsst/sdm_schemas
.. _felis: https://github.com/lsst/felis
.. _Semantic Versioning: https://semver.org/
.. _smig: https://github.com/lsst/qserv/blob/main/src/schema/README.md
.. _Alembic: https://alembic.sqlalchemy.org/
.. _DMTN-191: https://dmtn-191.lsst.io/
.. _daf_butler_migrate: https://github.com/lsst-dm/daf_butler_migrate
