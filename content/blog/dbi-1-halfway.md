+++
author = "Kirill Müller"
date = "2016-12-06"
draft = false
weight = 180
title = "Halfway through “Improving DBI”"
description = "Intermediate status report on the “Improving DBI” project"
+++

In early 2016 the R Consortium partially accepted my “Improving DBI”
proposal. An important part is the design and implementation of a
testable DBI specification. Initially I also proposed to make three DBI
backends to open-source databases engines (RSQLite, RMySQL, and
RPostgres) compatible to the new DBI specification, but funding allows
to work on only one DBI backend. I chose RSQLite for a number of
reasons:

-   It is a very important package, judging by the number of reverse
    CRAN and Bioconductor dependencies
-   It’s easy to work with, because everything (including the database
    engine) is bundled with the package
-   It seemed to be the most advanced package, closest to the (yet to be
    completed) DBI specification
-   An [informal Twitter
    poll](https://twitter.com/krlmlr/status/712950420969283584) supports
    this decision by a tiny margin

The project has reached an important milestone, with the release of
RSQLite 1.1. This post reports the progress achieved so far, and
outlines the next steps.

RSQLite
-------

While the RSQLite API has changed very little (hence the minor version
update), it includes a complete rewrite of the original 1.0.0 sources in
C++. This has considerably simplified the code, which makes future
maintenance easier, and allows us to take advantage of the more
sophisticated memory management tools available in Rcpp, which help
protect against memory leaks and crashes.

RSQLite 1.1 brings a number of improvements:

-   New strategy for prepared queries: Create a prepared query with
    `dbSendQuery()` or `dbSendStatement()` and bind values with `dbBind()`.
    This allows you to efficiently re-execute the same query/statement
    with different parameter values iteratively (by calling `dbBind()`
    several times) or in a batch (by calling `dbBind()` once with a
    data-frame-like object).
-   Support for inline parametrised queries via the param argument to
    `dbSendQuery()`, `dbGetQuery()`, `dbSendStatement()` and `dbExecute()`, to
    protect from [SQL injection](https://xkcd.com/327/).
-   The existing methods `dbSendPreparedQuery()` and `dbGetPreparedQuery()`
    have been soft-deprecated, because the new API is more versatile,
    more consistent and stricter about parameter validation.
-   Using UTF8 for queries and parameters: this mean that non-English
    data should just work without any additional intervention.
-   Improved mapping between SQLite’s cell-types and R’s column-types.

See the [release
notes](https://github.com/rstats-db/RSQLite/releases/tag/v1.1) for
further changes.

The rewrite was implemented by Hadley Wickham before the “Improving DBI”
project started, and has been available for a long time on GitHub.
Nevertheless, the CRAN release has proven much more challenging than
anticipated, because so many CRAN and Bioconductor packages import it.
(Maintainers of reverse dependencies might remember multiple e-mails
where I was threatening to release RSQLite “for real”.) My aim was to
break as little existing code as possible. After numerous rounds of
revdep-checking and improving RSQLite, I’m proud to report that the vast
majority of reverse dependencies pass their checks just as well (and as
quickly!) as they did with v1.0.0. Most tests from v1.0.0 are still
present in the current codebase. This means that non-packaged code also
has a good chance to work unchanged. I’m happy to work with package
maintainers or users whose code breaks after the update.

DBI
---

I have also released several DBI updates to CRAN, mostly to introduce
new generics such as `dbBind()` (for parametrized/prepared queries) or
`dbSendStatement()` and `dbExecute()` (for statements which don’t return
data). The definition of a formal DBI specification is part of the
project, a [formatted
version](http://rstats-db.github.io/DBI/DBIspec.html) is updated
continuously.

DBItest
-------

In addition to the textual specification in the DBI package, the DBItest
package provides backend independent tests for DBI packages. It can be
easily used by package authors to ensure that they follow the DBI
specification. This is important because it allows you to take code that
works with one DBI backend and easily switch to a different backend
(providing that they both support the same SQL dialect). Literate
programming techniques using advanced features of roxygen2 help keeping
both code and textual specifications in close proximity, so that
amendments to the text can be easily tracked back to changes of the test
code, and vice versa.

Next steps
----------

The rest of the project will focus on finalizing the specification in
both code and text (mostly discussed on GitHub in the issue trackers for
the [DBI](https://github.com/rstats-db/DBI/issues) and
[DBItest](https://github.com/rstats-db/DBItest/issues) projects). At
least one new helper package (to handle 64-bit integer types) will be
created, and DBI, DBItest, and RSQLite will see yet another release: The
first two will finalize the DBI specification, and RSQLite will fully
conform to it.

The development happens entirely on GitHub in repositories of the
[rstats-db](https://github.com/rstats-db) organization. Feel free to try
out development versions of the packages found there, and to report any
problems or ideas at the issue trackers.
