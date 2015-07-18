---
description:
  "There is a growing number of migration toolkits for Python: Alembic,
  sqlalchemy-migrate, GenericSetup, zope.generations, south, etc. All
  of these focus on migrating a specific type of database, and some
  even on migration within a specific type of application (Zope or Django).
  They work very well, but when you need to deal with larger applications
  they no longer suffice."
---
There is a growing number of migration toolkits for Python:
[Alembic](http://alembic.readthedocs.org/en/latest/) and
[sqlalchemy-migrate](http://pypi.python.org/pypi/sqlalchemy-migrate/0.7.2) for
applications using [SQLAlchemy](http://sqlalchemy.org),
[GenericSetup](http://pypi.python.org/pypi/Products.GenericSetup) for
[Zope](http://zope.org/) products,
[repoze.evolution](http://pypi.python.org/pypi/repoze.evolution) and
[zope.generations](http://pypi.python.org/pypi/zope.generations) for
applications using the [ZODB](http://zodb.org/),
[south](http://south.aeracode.org/) for Django applications, and many more.
All these packages use essentially the same approach:

* introduce a versioning scheme for your database. This is either an increasing
  number (most systems), or a chain of hashes (Alembic)
* track the current version in the database.
* allow developers to write migration code to move from one version to another.

The differences are mostly in the details: some packages support downgrades,
some can automaticalyl detect schema changes and generate migration code,
others can deal with branches, etc. All thse toolkits make two assumptions
they operate in: there is exactly one storage system (generally SQL), and the
migration code can be defined in one place. These assumptions are very
reasonable for most applications, but for more complex applications they do
not hold.

Applications may be using multiple storage systems. For example a travel
website will deal with a lot of images and migbt use a relational database to
store metadata while storing the raw images directly on the filesystem. A CMS
system might use an object store to easily handle documents, but also use a
fast key-value storage to track user behaviour. That means a migration
framework must be able to deal with multiple storage systems at the same time.

There is another related complication: storage systems may be replaced
completely during the lifetime of an applications. Perhaps you started with a
SQL database but you discover you data is inheretenly hierarchical in nature
so a document store will be better, or the read/write request ratio is turning
out to be very different than you were initially expecting. In situations like
that switching to a different storage system, or combining multiple storage
systems, may be the right thing to do. This implies another requirement: no
assumptions about presence of any storage system may be made. That means
storing a version number in a predefined place in a specific database will not
work, which is something all current migration toolkits do.

There is one final aspect in this analysis: complex applications tend to be
composed of many separate components, each of which may define their own part
of the schema or perhaps even use their own storage system. Consider our travel
website example: the handling of images might be implemented by a reusable
component that you will also use in an online shopping site you are going
to build. The website will define its own models which need to interact
with the image data, and both the image compoment and the website may have
their own migrations. This leads to our final requirement: a migration tool
must be able to discover and run migrations for all components in a software
stack

Summaring the above we can define several requirements:

* be able to handle multiple types and instances of storage systems.
* must not rely on any specific storage system to be available to
  store its own data.
* allow running migrations for all components used in an application.

Unfortunately I am not aware of a (Python) migration toolkit that can fulfill
these requirements at this moment. 
