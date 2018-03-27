+++
author = "Kirill Müller"
date = "2017-05-15"
draft = false
weight = 180
title = "Improving DBI: A Retrospect"
description = "Summary of the “Improving DBI” project"
+++

The "Improving DBI" project, funded by the R consortium, started about a year ago.
It includes

1. the definition and implementation of a testable specification for DBI,
2. making `RSQLite` fully compliant to the new specification.

Besides the established `DBI` and `RSQLite` packages, I have spent a lot of time on the new `DBItest` package.
Final updates to these packages will be pushed to CRAN end of May this year, to give downstream maintainers some time to accommodate.

The follow-up project "Establishing DBI" will focus on fully DBI-compliant backends for MySQL/MariaDB and PostgreSQL, and on minor updates to the specs where appropriate.


## `DBItest`: Specification

A comprehensive backend-agnostic test suite for DBI backends is provided by the new `DBItest` package.
When the project started, it was merely a collection of test cases.
I have considerably expanded the test cases and provided a human-readable description for each, using literate programming techniques powered by `roxygen2`.
The `DBI` package weaves these chunks of text to a single document that describes all test cases covered by the test suite, the textual *DBI specification*.
This approach ensures that further updates to the specification are reflected in both the automatic tests and the text.

This package is aimed at backend implementers, who now can programmatically check with very little effort if their DBI backend conforms to the DBI specification.
The verification can be integrated in the automated tests which are run as part of R's package check mechanism in `R CMD check`.
The `odbc` package, a new DBI-compliant interface to the ODBC interface, has been using `DBItest` from day one to enable test-driven development.
The `bigrquery` package is another user of `DBItest`.

Because not all DBMS support all aspects of DBI, the `DBItest` package allows to restrict which parts of the specification are tested, or "tweak" certain aspects of the tests, e.g., the format of placeholders in parametrized queries.
Adapting to other DBMS may require more work due to subtle differences in the implementation of SQL between various DBMS.


## `DBI`: Definition

This package has been around since 2001, it defines the actual *DataBase Interface* in R.
I have taken over maintenance, and released versions 0.4-1, 0.5-1, and 0.6-1, with release of version 0.7 pending.

The most prominent change in this package is, of course, the textual [DBI specification](https://cran.r-project.org/web/packages/DBI/vignettes/spec.html), which is included as an HTML vignette in the package.
The documentation for the various methods defined by `DBI` is obtained directly from the specification.
These help topics are combined in a sensible order to a single, self-contained document.
This format is useful for both DBI users and implementers: users can look up the behavior of a method directly from its help page, and implementers can browse a comprehensive document that describes all aspects of the interface.
I have also revised the description and the examples for all help topics.

Other changes include:

- the definition of new generics `dbSendStatement()` and `dbExecute()`, for backends that distinguish between queries that return a table and statements that manipulate data,
- the new `dbWithTransaction()` generic and the `dbBreak()` helper function, thanks Barbara Borges Ribero,
- improved or new default implementations for methods like `dbGetQuery()`, `dbReadTable()`, `dbQuoteString()`, `dbQuoteIdentifier()`,
- internal changes that allow methods that don't have a meaningful return value to return silently,
- translation of a helper function from C++ to R, to remove the dependency on `Rcpp` (thanks Hannes Mühleisen).

Fortunately, none of the changes seemed to have introduced any major regressions with downstream packages.
The [news](https://github.com/rstats-db/DBI/blob/master/NEWS.md) contain a comprehensive list of changes.


## `RSQLite`: Implementation

`RSQLite` 1.1-2 is a complete rewrite of the original C implementation.
Before focusing on compliance to the new DBI specification, it was important to assert compatibility to more than 100 packages on CRAN and Bioconductor that use `RSQLite`.
These packages revealed many usage patterns that were difficult to foresee.
Most of these usage patterns are supported in version 1.1-2, the more esoteric ones (such as supplying an `integer` where a `logical` is required) trigger a warning.

Several rounds of "revdep checking" were necessary before most packages showed no difference in their check output compared to the original implementation.
The downstream maintainers and the Bioconductor team were very supportive, and helped spotting functional and performance regressions during the release process.
Two point releases were necessary to finally achieve a stable state.

Supporting 64-bit integers also was trickier than anticipated.
There is no built-in way to represent 64-bit integers in R.
The `bit64` package works around this limitation by using a `numeric` vector as storage, which also happens to use 8 bytes per element, and providing coercion functions.
But when an integer column is fetched, it cannot be foreseen if a 64-bit value will occur in the result, and smaller integers must use R's built-in `integer` type.
For this purpose, an efficient data structure for collecting vectors, which is capable of changing the data type on the fly, has been implemented in C++.
This data structure will be useful for many other DBI backends that need support for a 64-bit integer data type, and will be ported to the `RKazam` package in the follow-up project.

Once the DBI specification was completed, the process of making `RSQLite` compliant was easy: enable one of the disabled tests, fix the code, make sure all tests pass, rinse, and repeat.
If you haven't tried it, I seriosly recommend test-driven development, especially when the tests are already implemented.

The upcoming release of `RSQLite` 2.0 will require stronger adherence to the DBI specification also from callers.
Where possible I tried to maintain backward compatibility, but in some cases breaks were inevitable because otherwise I'd have had to introduce far too many exceptions and corner cases in the DBI spec.
For instance, row names are no longer included by default when writing or reading tables.
The original behavior can be reenabled by calling `pkgconfig::set_config()`, so that packages or scripts that rely on row names continue to work as before.
(The setting is active for the duration of the session, but only for the caller that has called `pkgconfig::set_config()`.)
I'm happy to include compatibility switches for other breaking changes if necessary and desired, to achieve both adherence to the specs and compatibility with existing behavior.

A comprehensive list of changes can be found in the [news](https://github.com/rstats-db/RSQLite/blob/master/NEWS.md).


## Other bits and pieces

The `RKazam` package is a ready-to-use boilerplate for a DBI backend, named after the hypothetical DBMS used as example in a DBI vignette.
It already "passes" all tests of the `DBItest` package, mostly by calling a function that skips the current test.
Starting a DBI backend from scratch requires only copying and renaming the package's code.

R has limited support for time-of-day data.
The `hms` package aims at filling this gap.
It will be useful especially in the follow-up project, because SQLite doesn't have an intrinsic type for time-of-day data, unlike many other DBMS.


## Next steps

The ensemble CRAN release of the three packages `DBI`, `DBItest` and `RSQLite` will occur in parallel to the startup phase for the "Establishing DBI" follow-up project.
This project consists of:

- Fully DBI compatible backends for MySQL/MariaDB and Postgres
- A backend-agnostic C++ data structure to collect column data in the `RKazam` package
- Support for spatial data

In addition, it will contain an update to the DBI specification, mostly concerning support for schemas and for querying the structure of the table returned for a query.
Targeting three DBMS instead of one will help properly specify these two particularly tricky parts of DBI.
I'm happy to take further feedback from users and backend implementers towards further improvement of the DBI specification.


## Acknowledgments

Many thanks to the R Consortium, who has sponsored this project, and to the many contributors who have spotted problems, suggested improvements, submitted pull requests, or otherwise helped make this project a great success.
In particular, I'd like to thank Hadley Wickham, who suggested the idea, supported initial development of the `DBItest` package, and provided helpful feedback; and Christoph Hösler, Hannes Mühleisen, Imanuel Costigan, Jim Hester, Marcel Boldt, and @thrasibule for using it and contributing to it.
I enjoyed working on this project, looking forward to "Establishing DBI"!
