+++
author = "Kirill Müller"
date = "2021-12-21"
draft = false
weight = 180
title = "Maintaining DBI, 4/4"
description = "Summarizing the progress of 2021"
+++

## What is DBI?

The {DBI} package (**d**ata**b**ase **i**nterface) connects R to database management systems (DBMS).
It is an interface that only offers a common interface.
Connectivity to DBMS is established through dedicated [backend packages](https://github.com/r-dbi/backends#readme) such as
[RPostgres](https://rpostgres.r-dbi.org/), [RMariaDB](https://rmariadb.r-dbi.org/), and [RSQLite](https://rsqlite.r-dbi.org/).
If you're new to DBI, read the [introductory tutorial](https://dbi.r-dbi.org/articles/dbi) to get started.

The current version of DBI is 1.1.2.
In this blog post, a summary of recent developments in {DBI} and related packages is provided.
See the three previous blog posts for summaries from earlier years.

- [1/4: December 2018](https://www.r-dbi.org/blog/dbi-3-1/)
- [2/4: December 2019](https://www.r-dbi.org/blog/dbi-3-2/)
- [3/4: January 2021](https://www.r-dbi.org/blog/dbi-3-3/)

The post concludes with an outlook on future development in DBI and related packages.


## Recent developments in DBI

A few updates happened since the last blog post:

- DBI 1.1.1 -> 1.1.2 ([NEWS](https://dbi.r-dbi.org/news/))
- RMariaDB 1.1.0 -> 1.2.1 ([NEWS](https://rmariadb.r-dbi.org/news/))
- RPostgres 1.3.1 -> 1.4.3 ([NEWS](https://rpostgres.r-dbi.org/news/))
- RSQLite 2.2.2 -> 2.2.9 ([NEWS](https://rsqlite.r-dbi.org/news/))
- DBItest 1.7.0 -> 1.7.2 ([NEWS](https://dbitest.r-dbi.org/news/))

This section discusses a few noteworthy improvements, both visible and internal.

### Clickable method documentation

The DBI method reference on <https://dbi.r-dbi.org/reference/> has been enhanced with clickable links to known DBI backends.
This makes it easier to access your database: some backends provide extended functionality through optional arguments in their methods, and these should be documented in the backend's reference.

### Full support for AWS Redshift

Thanks to Adam Foryś, Redshift support is now on par with regular Postgres databases in the RPostgres package.
Both databases pass the entire DBItest test suite, except for Redshift's missing support for BLOB data.
Use `Redshift()` instead of `Postgres()` to connect to a Redshift cluster.

### Faster table imports

Previous versions of {RMariaDB} and {RPostgres} relied on passing multiple values to `dbBind()` for writing tables.
This is very slow for sending data to the database, because each row requires a communication roundtrip to the server.
To improve the situation, {RMariaDB} now uses `LOAD DATA LOCAL INFILE` to load data from a temporary CSV file.
Because recent versions of the MySQL database server disable this capability, this must be explicitly enabled by passing `load_data_local_infile = TRUE` to `dbConnect()`.
For {RPostgres}, `dbAppendTable()` now uses the same fast strategy as `dbWriteTable()` for writing data.

### Windows compatibility

{RMariaDB} now can use the `caching_sha2_password` plugin on Windows which was disabled on previous versions.
This is important for connecting to recent versions of MySQL which require this plugin.

### Extended data types for SQLite

Thanks to Eric Anderson, {RSQLite} now returns typed data for columns declared with `DATE`, `TIME` and `TIMESTAMP`.
Set `extended_types = TRUE` in `dbConnect()` to enable this feature.

### Interrupt handling

The `check_interrupts = TRUE` argument to `dbConnect()` for Postgres database now correctly cancels the query and returns to the user as soon as Ctrl+C (or Escape in RStudio) is pressed.
Thanks to Mateusz Żółtak for tests and discussion.

### Automation

{RMariaDB} is tested with all combinations of MariaDB and MySQL client + server on GitHub Actions.
{RPostgres} is tested with all versions of PostgreSQL >= 10 on GitHub Actions.
This ensures that future updates of these packages continue to work in all scenarios.
All tests also run daily, this ensures that upstream updates do not break our backend.

Thanks to the automated checks for upgraded SQLite3 versions, the vendored code could be updated continuously shortly after upstream releases.
{RSQLite} now uses SQLite3 3.37.0, released to CRAN about 10 days after the upstream release.

### Simpler upgrade path for DBItest

It is now simpler to bring changes to DBItest to CRAN.
Backends declare the version of {DBItest} they support with the new `dbitest_version` tweak.
New tests in DBItest are skipped if the declared version is too low.
Skipped tests are reported in the test results and can be fixed independently of the {DBItest} releases.

### Inlined Boost headers

The {BH} package is a header-only C++ package with more than 10000 files.
Installing this package proved challenging on some file systems, namely Amazon Elastic File System.
By inlining only the required files into {RSQLite}, {RMariaDB} and {RPostgres}, it is no longer necessary to install {BH} to use these packages, and the total number of files required to build these packages is greatly reduced.
Thanks to RStudio for supporting this change.

### Reorganized structure of the R code

DBI uses S4 to declare its classes, generics, and methods.
Code for the methods is declared with `setMethod()`, with a syntax similar to this:

```r
setMethod("foo", c("myclass", "character"), function(x, char_arg, ...) {
  ...
})
```

This notation has been replaced with the following way of writing the same code, throughout all packages mentioned in this post:

```r
foo_myclass_character <- function(x, char_arg, ...) {
  ...
}

setMethod("foo", c("myclass", "character"), foo_myclass_mycharacter
```

This brings the following advantages:

- From the list of files it's obvious which methods are implemented in a package,
- Code is easier to read and navigate,
- The source code of the implementation of an S4 method can be displayed easily via `mypkg:::foo_myclass_character`.

The code transformation was semi-automated, with the help of a [script](https://github.com/pre-processing-r/rpp/blob/main/script/un-s4.R) that uses infrastructure from the "Pre-processing R code" project.


## Future work

The {DBI} package provides a low-level interface for database connectivity, with a [limited scope](https://www.r-dbi.org/blog/dbi-3-3/).
Data query and manipulation tasks outside of {DBI}'s limited scope are left to packages like [dbplyr](https://dbplyr.tidyverse.org/), [dm](https://cynkra.github.io/dm/), [dbx](https://github.com/ankane/dbx) and [rquery](https://winvector.github.io/rquery/).

{DBI} is based on S4, a system for object-oriented programming in R.
It offered several advantages over S3, e.g. strictness and multiple dispatch.
One of the weaknesses encountered when working with on the DBI projects was rigidity: once a generic is defined and published, it is very difficult to change its definition.

The first release of {DBI} happened about 20 years ago.
Many packages use it to access databases or provide a [backend](https://github.com/r-dbi/backends#readme) to a DBMS.
Combined with the rigidity it seems very difficult to extend {DBI} or even add new generics: there is pressure to "get it right" with the first attempt.

The DBI specification aims at standardizing the features of {DBI}, the {DBItest} package provides a test suite.
Due to differences between database implementations, not all features of the DBI specification and not all tests are supported by all backend packages.
Backends declare, by means of "tweaks", which tests to run in what way.
This solves the problem of implementing a test suite that works for many backends.
However, general-purpose clients only can assume which capabilities are supported by a backend or a connection, a formal way to declare these capabilities are missing.

Based on these observations, for extending DBI, it may be worthwhile to address the following two issues first and foremost:

1. Formal declaration of capabilities,
1. Decoupling the user's from the implementer's interface.

The new [dbi3 repository](https://github.com/r-dbi/dbi3) contains a collection of issues that will be much easier to address after these changes are in place.

### Capabilities

Backends declare explicitly which features of DBI a particular connection supports.
Examples for existing functionality include:

- Support for BLOBs (not available e.g. in Redshift)
- Support for logical columns (not available e.g. in SQL Server or SQLite)
- Support for named or nested transactions (not standardized)
- Which placeholder to use for parameterized queries (different across databases)

In the future, backends may indicate:

- if asynchronous queries are supported (important for web development),
- if the database supports SQL or a different query language (DBI currently assumes SQL).

The list of possible capabilities is maintained by DBI.
An updated version of {DBItest} can rely solely on these capabilities.
Users can query these capabilities and act based on the results, but very often will not need to -- read on.

### Separate user interface

The idea here is that users of DBI should not *need to* call methods implemented by backends directly.
Instead, a separate user interface based on plain functions -- a [facade](https://en.wikipedia.org/wiki/Facade_pattern) -- is provided.
This user interface can be designed to be sufficient for the overwhelming majority of users and use cases, and at the same time contribute to simpler code with fewer duplication in backend packages.

The new user interface performs tasks that are common to all database backends (e.g. validation of arguments), and calls methods provided by the backends, determined by the declared capability.
This leads to fewer code that needs to be reimplemented across backends.
For example, a rich `dbi_write_table()` function that optionally creates and writes data to a database table might:

- if the backend supports transactions:
  - call `dbi_is_transacting()` to determine if the statement is inside a transaction
  - call `dbi_begin_transaction()` if not inside a transaction
  - always call `dbi_begin_transaction(name = "...")` if the backend supports named transactions
- call `dbi_remove_table()` and/or `dbi_create_table()` if necessary
- call `dbi_append_table()` 
- call `dbi_commit()` on success or `dbi_rollback()` on failure if transactions are supported

In turn, for appending rows to a table, `dbi_append_table()` might check if the backend supports streamed upload or if it should create SQL to insert rows.
In the latter case, the SQL would be constructed using quoted literals obtained from `dbi_quote_literal()`.
The backend would indicate the maximum length of a query, large tables would be split into chunks automatically based on that size.

Similarly, a backend that supports asynchronous operations might use a default implementation for the corresponding synchronous operations: the user interface detects that async is supported and implements sync by forwarding to async and waiting.

This allows generics declared by {DBI} to remain frozen.
To extend or alter the signature of a generic, a new generic (e.g. with a numeric suffix) is conceived: `dbAppendTable1()`, `dbAppendTable2()`, etc..
All arguments in new generics could be declared explicitly, without relying on passing them through `...` as is done currently.

The new user interface is responsible for deciding which methods to call.
When a new version of a generic is declared, {DBI} documents and proposes an upgrade path for backend implementers.
In the long run this also allows transitioning to another object-oriented system such as S3 or [R7](https://github.com/RConsortium/OOP-WG/).

This approach also enables support of rich callbacks: each function in the facade notifies listeners before it is called and after results are complete.
For example, a call to `dbi_connect()` will notify interested parties that a new connection has been established, and a call to `dbi_query()` issues callback with the query and the result.
Use cases include:

- logging as in {dblog}
- the "Connections" pane in the RStudio IDE
- mocking as in {dittodb} (with hooks)

Similarly to methods, existing callbacks will remain frozen but new extended callbacks can be defined, e.g. with a numeric suffix.


## Acknowledgments

I'd like to thank Jeroen Ooms and Gábor Csárdi for providing crucial infrastructure to support this and many other projects in the R ecosystem.

Thanks the numerous contributors to the packages in the "Maintaining DBI" project in 2021:

- DBI: [&#x0040;bwohl](https://github.com/bwohl), [&#x0040;cboettig](https://github.com/cboettig), [&#x0040;dcassol](https://github.com/dcassol), [&#x0040;jawond](https://github.com/jawond), [&#x0040;krlmlr](https://github.com/krlmlr), [&#x0040;mmccarthy404](https://github.com/mmccarthy404), [&#x0040;pnacht](https://github.com/pnacht), [&#x0040;r2evans](https://github.com/r2evans), [&#x0040;vituri](https://github.com/vituri), and [&#x0040;zoushucai](https://github.com/zoushucai);
- RSQLite: [&#x0040;ablack3](https://github.com/ablack3), [&#x0040;billy34](https://github.com/billy34), [&#x0040;edwindj](https://github.com/edwindj), [&#x0040;gaborcsardi](https://github.com/gaborcsardi), [&#x0040;ggrothendieck](https://github.com/ggrothendieck), [&#x0040;giocomai](https://github.com/giocomai), [&#x0040;habilzare](https://github.com/habilzare), [&#x0040;honghh2018](https://github.com/honghh2018), [&#x0040;Jeff-Gui](https://github.com/Jeff-Gui), [&#x0040;kevinushey](https://github.com/kevinushey), [&#x0040;mgirlich](https://github.com/mgirlich), [&#x0040;plantton](https://github.com/plantton), [&#x0040;schuemie](https://github.com/schuemie), [&#x0040;Shicheng-Guo](https://github.com/Shicheng-Guo), and [&#x0040;tschoonj](https://github.com/tschoonj);
- RPostgres: [&#x0040;aleaficionado](https://github.com/aleaficionado), [&#x0040;armenic](https://github.com/armenic), [&#x0040;ateucher](https://github.com/ateucher), [&#x0040;baderstine](https://github.com/baderstine), [&#x0040;beralef](https://github.com/beralef), [&#x0040;carlganz](https://github.com/carlganz), [&#x0040;ColinFay](https://github.com/ColinFay), [&#x0040;dcaud](https://github.com/dcaud), [&#x0040;dpprdan](https://github.com/dpprdan), [&#x0040;f-ritter](https://github.com/f-ritter), [&#x0040;galachad](https://github.com/galachad), [&#x0040;GitHubGeniusOverlord](https://github.com/GitHubGeniusOverlord), [&#x0040;gontcharovd](https://github.com/gontcharovd), [&#x0040;gtm19](https://github.com/gtm19), [&#x0040;hadley](https://github.com/hadley), [&#x0040;jakob-r](https://github.com/jakob-r), [&#x0040;jeroen](https://github.com/jeroen), [&#x0040;jkylearmstrong](https://github.com/jkylearmstrong), [&#x0040;JSchoenbachler](https://github.com/JSchoenbachler), [&#x0040;matthewgson](https://github.com/matthewgson), [&#x0040;mgirlich](https://github.com/mgirlich), [&#x0040;mmuurr](https://github.com/mmuurr), [&#x0040;mskyttner](https://github.com/mskyttner), [&#x0040;phedinkus](https://github.com/phedinkus), [&#x0040;ppssphysics](https://github.com/ppssphysics), [&#x0040;RakeshG1](https://github.com/RakeshG1), [&#x0040;rickbpdq](https://github.com/rickbpdq), [&#x0040;samiaab1990](https://github.com/samiaab1990), [&#x0040;sawnaanwas](https://github.com/sawnaanwas), [&#x0040;tomasjanikds](https://github.com/tomasjanikds), [&#x0040;vspinu](https://github.com/vspinu), [&#x0040;waynelapierre](https://github.com/waynelapierre), [&#x0040;zmbc](https://github.com/zmbc), and [&#x0040;zozlak](https://github.com/zozlak);
- RMariaDB: [&#x0040;bakiunal](https://github.com/bakiunal), [&#x0040;dirkschumacher](https://github.com/dirkschumacher), [&#x0040;hadley](https://github.com/hadley), [&#x0040;jeroen](https://github.com/jeroen), [&#x0040;Mosk915](https://github.com/Mosk915), [&#x0040;noamross](https://github.com/noamross), [&#x0040;paulmaunders](https://github.com/paulmaunders), [&#x0040;retowyss](https://github.com/retowyss), [&#x0040;rorynolan](https://github.com/rorynolan), [&#x0040;twentytitus](https://github.com/twentytitus), [&#x0040;verajosemanuel](https://github.com/verajosemanuel), [&#x0040;wiligl](https://github.com/wiligl), [&#x0040;Woosah](https://github.com/Woosah), and [&#x0040;zoushucai](https://github.com/zoushucai).
- DBItest: [&#x0040;adamsma](https://github.com/adamsma), and [&#x0040;michaelquinn32](https://github.com/michaelquinn32);
