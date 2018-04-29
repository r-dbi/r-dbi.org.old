+++
author = "Kirill Müller"
date = "2018-03-28"
draft = true
weight = 180
title = "Done “Establishing DBI”!?"
description = "Summary of the “Establishing DBI” project"
+++

The "Establishing DBI" project, funded by the R consortium, started about a year ago.
It includes the completion of two new backends, *RPostgres* and *RMariaDB*, and a quite a few interface extensions and specifications.
This blog post showcases only the visible changes, a substantial amount of work went into extending the DBI specification and making the three open-source database backends compliant to it.

The DBI specification has been formulated in the preceding R consortium project, "Improving DBI". It is both an automated test suite and a human-readable description of behavior, implemented in the *DBItest* package. For this project, I extended this specification and could also use it to implement *RPostgres* and *RMariaDB*: for once, test-driven development was pure pleasure, because the tests were already there!

## Release of *RPostgres* and *RMariaDB*

The *RPostgres* and *RMariaDB* packages are complete rewrites of the *RPostgreSQL* and *RMySQL* packages, respectively. These packages use C++ (with *Rcpp*) as glue between R and the native database libraries. A reimplementation and release under a different name has made it much easier to implement the DBI specification: only listing temporary tables is not 
supported by *RMariaDB* (due to a limitation of the DBMS), all other parts of the specification are fully covered.

Projects that use *RPostgreSQL* or *RMySQL* can continue to do so, or switch to the new backends at their own pace (which is likely to require some changes to the code). For new projects I recommend *RPostgres* or *RMariaDB* to take advantage of the thorougly tested codebases and of the consistency across backends.


## Quoting literal values

When working on the database backends, it has become apparent that quoting strings and identifiers isn't quite enough. Now there is a way to quote arbitrary values, i.e. convert them to a string that can be pasted into a SQL query:



- `dbQuoteLiteral()`

## Schema support

- `Id()`
- `dbUnquoteIdentifier()`
- `dbListObjects()`

## More fine-grained creation of tables

- `dbCreateTable()`
- `dbAppendTable()`

## Helpers

- `dbIsReadOnly()`
- `dbCanConnect()`

## Postgres geometry column support

## C++ classes?

## Enhanced tests

## Work to do

### "Hard" issues

1. The test suite for the DBI specificaton in `DBItest` is currently designed to run as part of the package checks.  The next step is to support running the test suite against a particular R + DBMS installation, to ensure that code interoperating with that DBMS in that environment runs as expected.

1. Users expect the hard disk or the DBMS to be the limiting factor for loading data, but DBI still lacks a consistent interface for fast data import.

1. The syntax for query placeholders currently depends on the DBMS.  A consistent interface would be useful, in particular for implementers of packages that compute on the database.

1. The `RPostgres` package now has special handling for geometry data.  A generic extension to arbitrary data types via hooks would allow e.g. returning JSON data directly as a `"json"` class without user-initiated manual conversion.

### "Soft" issues

1. Some users reported installation problems on specific architectures, or connectivity problems with certain databases, or other specific issues.  Making the new backends accessible for various combinations of OS/hardware, software, and configuration, will help the adoption of the new packages.

1. The internal architecture of the DBI specification in `DBItest` requires a bit of reworking. Currently, it is difficult to understand a test failure without inspecting the source code of `DBItest`.  It is difficult to locate the source of a failure in the specification and in the code.  Ideally, each test failure would come with a precise link to the part of the specification that is violated, and with a simple sequence of DBI method calls that allow replicating the failure externally.

1. The communication related to the projects has been rather terse so far.  The new website https://r-dbi.org can host blog posts highlighting different aspects of DBI, and serve as a resource for advice on connecting R with databases and computing on the database.  This includes coordination and support for developments around DBI like `r citet_pkg("sqlr")`, an interface for data definition statements on top of DBI.

1. All operations on DBI currently block until a result is available or the DBMS has indicated completion.  Asynchronous operations allow parallel processing of multiple queries or statements, however some research is necessary to understand to what extent this can be supported realistically in DBI and for the existing backends.

These lists are not comprehensive, new issues may surface over time, or the importance of issues mentioned above may fade.


## Acknowledgments

Many thanks to the R Consortium, who has sponsored this project, and to the many contributors who have spotted problems, suggested improvements, submitted pull requests, or otherwise helped make this project a great success.
In particular, I'd like to thank Hadley Wickham, who suggested the idea, supported initial development of the `DBItest` package, and provided helpful feedback; and Christoph Hösler, Hannes Mühleisen, Imanuel Costigan, Jim Hester, Marcel Boldt, and @thrasibule for using it and contributing to it.
I enjoyed working on this project, looking forward to "Establishing DBI"!
