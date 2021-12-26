+++
author = "Kirill Müller"
date = "2021-12-21"
draft = false
weight = 180
title = "Maintaining DBI, 4/4"
description = "Summarizing the progress of 2021"
+++

<!-- how do you want to pluralize abbreviations? DBMSes, DBMS's DBMSs? -->

## What is DBI?

The {DBI} package (**d**ata**b**ase **i**nterface) provides an abstraction for communication between R and database management systems (DBMSes) by specifying a common application programming interface (API).
Actual connectivity to DBMSes is established via database specific [backend packages](https://github.com/r-dbi/backends#readme), implementing this interface.
Examples for such backends include [RPostgres](https://rpostgres.r-dbi.org/), [RMariaDB](https://rmariadb.r-dbi.org/), and [RSQLite](https://rsqlite.r-dbi.org/).
For users that are new to DBI, the [introductory tutorial](https://dbi.r-dbi.org/articles/dbi) provides a good entry point for getting acquainted with some key concepts.

This blog post summarizes recent developments in {DBI} and related packages and concludes with an outlook on potential future directions.
Similar articles are available from previous years, reporting on earlier states of the {DBI} ecosystem:

- [3/4: January 2021](https://www.r-dbi.org/blog/dbi-3-3/)
- [2/4: December 2019](https://www.r-dbi.org/blog/dbi-3-2/)
- [1/4: December 2018](https://www.r-dbi.org/blog/dbi-3-1/)

## Recent developments

Several packages associated with DBI have been updated since [early 2021](https://www.r-dbi.org/blog/dbi-3-3/):

- DBI 1.1.1 -> 1.1.2 ([NEWS](https://dbi.r-dbi.org/news/))
- RMariaDB 1.1.0 -> 1.2.1 ([NEWS](https://rmariadb.r-dbi.org/news/))
- RPostgres 1.3.1 -> 1.4.3 ([NEWS](https://rpostgres.r-dbi.org/news/))
- RSQLite 2.2.2 -> 2.2.9 ([NEWS](https://rsqlite.r-dbi.org/news/))
- DBItest 1.7.0 -> 1.7.2 ([NEWS](https://dbitest.r-dbi.org/news/))

And the following sections elaborate on some of the noteworthy changes and improvements contained in these updates, both user-visible and internal.

### Clickable method documentation

The DBI method reference on <https://dbi.r-dbi.org/reference/> has been updated to include clickable links to known DBI backends.
This makes documentation specific to certain backends more accessible, as optional function arguments used by some backend implementations are only documented by the respective packages.

### Full support for AWS Redshift

Redshift support has been greatly improved by Adam Foryś as part of the RPostgres package and both databases now pass all applicable tests offered by DBItest. The BLOB data type is currently not supported by Redshift and consequently, related tests are skipped.
For connecting to a Redshift cluster, the RPostgres package exports `Redshift()` (to be used over `Postgres()`).

### Faster table imports

Previous versions of {RMariaDB} and {RPostgres} relied `dbBind()` for writing tables, using a prepared `INSERT INTO ... VALUES (...)` statement with placeholders.
Contrary to the expectation, this was very inefficient, because each row requires a communication roundtrip to the server.
To improve the situation, {RMariaDB} now uses `LOAD DATA LOCAL INFILE` to load data from a temporary CSV file.
Recent MySQL server versions disable this capability by default, and it must therefore be explicitly enabled by passing `load_data_local_infile = TRUE` to `dbConnect()`.
For {RPostgres}, `dbAppendTable()` has been updated to use the same optimization as `dbWriteTable()` when writing data.

### Windows compatibility

{RMariaDB} can now use the `caching_sha2_password` plugin on Windows which was permanently disabled on previous versions.
This is important for connecting to recent versions of MySQL which require this plugin.

### Extended data types for SQLite

Thanks to Eric Anderson, {RSQLite} now returns typed data for columns declared with `DATE`, `TIME` and `TIMESTAMP` data types.
To enable this feature, `extended_types = TRUE` has to be passed in `dbConnect()`.

### Interrupt handling

The `check_interrupts = TRUE` argument to `dbConnect()` in {RPostgres} now correctly cancels the query and returns to the user as soon as an interrupt is signalled (by pressing Ctrl+C or Escape in RStudio).
Thanks to Mateusz Żółtak for tests and discussion.

### Automation

{RMariaDB} is tested against all combinations of major MariaDB and MySQL client/server releases, while {RPostgres} is tested against all versions of PostgreSQL &ge; 10 using GitHub Actions.
This guarantees compatibility with a broader range of database instances for both backends and for future updates to the corresponding packages.
All tests are run daily, thereby ensuring that upstream updates remain compatible with backend implementations.
The database servers are installed on GitHub Actions using dedicated actions for installing [MariaDB](https://github.com/ankane/setup-mariadb/), [MySQL](https://github.com/ankane/setup-mysql) and [Postgres](https://github.com/ankane/setup-postgres), maintained by Andrew Kane.

Thanks to the automated monitoring of SQLite3 releases, the vendored code can be updated continuously with minimal delay over upstream releases.
{RSQLite} now uses SQLite3 3.37.0 and became available from CRAN only 10 days after the upstream release.

### Simpler upgrade path for DBItest

By making it possible for backends to specify the supported version of {DBItest}, using `tweaks(dbitest_version = "x.y.z")`, it is now simpler to update {DBItest} on CRAN.
Newly added tests in {DBItest} are skipped if the declared version is too low.
Skipped tests are reported in the test results and can be fixed independently of the {DBItest} releases.

### Inlined Boost headers

The {BH} package is a C++ header-only package containing in excess of 10,000 individual files and installation has proven challenging for some systems, such as Amazon's Elastic File System.
By vendoring only the required files into {RSQLite}, {RMariaDB} and {RPostgres}, it is no longer necessary to install {BH} to use these packages, and therefore the total number of files required to build these packages is greatly reduced.
Thanks to RStudio for supporting this change.

### Reorganized structure of the R code

{DBI} uses S4 generic functions to specify the interface to be implemented by backends, using database specific subclasses.
Class specific generic implementations are consequently declared with `setMethod()`, using the following convention:

```r
setMethod("foo", c("myclass", "character"), function(x, char_arg, ...) {
  ...
})
```

Instead of passing an anonymous function as `definition` argument to `setMethod()`, this has been changed to a semantic equivalent which is more explicit:

```r
foo_myclass_character <- function(x, char_arg, ...) {
  ...
}

setMethod("foo", c("myclass", "character"), foo_myclass_mycharacter)
```

Reasons for this transformation were to make the respective implementations more accessible, as function definitions now can be displayed more easily via `mypkg:::foo_myclass_character`, and to generally make the code-base easier to read and navigate.
In similar spirit, each such generic implementation is now defined in its own file with file names constructed as `foo_myclass_mycharacter.R`.
This makes it immediately clear, exactly which methods are implemented by a package, simply from the list of associated files.
Code transformation was carried out in semi-automated fashion, with the help of a [script](https://github.com/pre-processing-r/rpp/blob/main/script/un-s4.R) that uses infrastructure from the "Pre-processing R code" project.

## Future work

The {DBI} package provides a low-level interface for database connectivity with a [narrow scope](https://www.r-dbi.org/blog/dbi-3-3/).
Data query and manipulation tasks that are not in-scope for {DBI} are currently left to auxiliary packages, including [dbplyr](https://dbplyr.tidyverse.org/), [dm](https://cynkra.github.io/dm/), [dbx](https://github.com/ankane/dbx) and [rquery](https://winvector.github.io/rquery/).
{DBI} uses S4, one of several systems for object-oriented programming in R.
While S4 offers several advantages over its predecessor S3, including increased strictness and multiple dispatch, it also is more rigid compared to S3.
<!-- I don't see how this is specific to S4, but then again, I haven't really used S4 myself, so I'm no expert; it might still be worthwhile to elaborate a bit here -->
<!-- also, what is the intention of discussing S4 here? is it to justify a move away from S4? if so I feel a need for fleshing out this argument more -->
Consequently, once the definition of a generic is published, it is difficult to make changes without breaking downstream dependencies.

The first release of {DBI} dates back roughly 20 years and since, the package has been widely adopted by others, both for accessing databases or providing [backends](https://github.com/r-dbi/backends#readme) to DBMSes.
Its success, combined with the rigidity imposed by S4, has made it difficult to extend the interface beyond what is currently offered.
When considering new additions, there is pressure to get it right in the first attempt, thereby holding back less essential improvements.

The DBI specification aims to standardize the feature set of {DBI}-compliant backends, while the {DBItest} package provides a test suite against which conformity of an implementation can be verified.
Due to differences in design of individual DBMSes, not all features of the DBI specification and therefore not all tests provided by {DBItest} are supported by all backend packages.
Using a newly introduced mechanism, backends can declare, by means of *tweaks*, which tests to run in what way.
This addresses some of the problems associated with implementing a test suite that can be re-used for several backends.
General-purpose clients however, can only make guesses as to the exact feature set supported by a given backend or connection. There currently is no formal way to declare certain capabilities as missing (or available).

Based on these observations, for extending DBI, it may be worthwhile to address the following two issues:

1. Formal declaration of capabilities.
1. Decoupling the user from the backend interface.

The new [dbi3 repository](https://github.com/r-dbi/dbi3) contains a collection of issues, some of which will be easier to address after these changes are in place.

### Capabilities

A mechanism is introduced by which backends can declare explicitly which features of DBI a particular connection supports.
Examples for existing functionality that varies over backends include:

- Support for BLOBs (not available e.g. in Redshift).
- Support for logical columns (not available e.g. in SQL Server or SQLite).
- Support for named or nested transactions (not standardized).
- Placeholder character to use in parameterized queries (different across databases).

In the future, backends may also indicate:

- Whether asynchronous queries are supported (important for web development).
- Whether the database supports SQL or a different query language (DBI currently assumes SQL).

The list of possible capabilities will be maintained by DBI and {DBItest} will rely solely on these capabilities, foregoing the current *tweaking* mechanism.
Users can in turn query these capabilities and act accordingly.

### Separate user interface

As an evolution of the current approach, where users of DBI will often directly call methods that are mostly implemented by backends themselves, introduction of a separate user-facing API may be worthwhile.
Based on plain functions and essentially providing a [facade](https://en.wikipedia.org/wiki/Facade_pattern), this user interface would be sufficient for the overwhelming majority of use cases.
At the same time, such an approach should contribute to simpler code with less duplication in backend packages.

The new user interface performs tasks that are common to all database backends (e.g. validation of arguments), and calls methods provided by the backends, in some cases dependent on declared capabilities.
Overall, this should lead to less code that needs to be reimplemented across backends and the decoupling of interfaces could help with iterative improvements, while guaranteeing stability for users.
As an example, a `dbi_write_table()` function that optionally creates and writes data to a database table might encompass the following functionality:

- If the backend supports transactions:
  - Call `dbi_is_transacting()` to determine if the statement is occurring as part of a transaction.
  - Call `dbi_begin_transaction()` if it is not already part a transaction.
  - Use `dbi_begin_transaction(name = "...")` if the backend supports named transactions.
- Call `dbi_remove_table()` and/or `dbi_create_table()` if necessary.
- Call `dbi_append_table()` otherwise.
- Call `dbi_commit()` on success or `dbi_rollback()` on failure whenever transactions are supported.

For appending rows to a table, `dbi_append_table()` might check if the backend supports streaming uploads or if SQL should be created for inserting rows.
In the latter case, the SQL statement (or multiple statements for large tables) could be constructed using quoted literals obtained from `dbi_quote_literal()`.
The backend could indicate the maximum supported length of a statement, so that splitting of large tables into multiple chunks can happen automatically.

As a final example, a backend supporting asynchronous operations might rely entirely on DBI for providing the corresponding blocking operations.
The asynchronous procedure provided by the backend could automatically be wrapped by a DBI function that only returns upon completion.

Such a split API would allow for generics declared by {DBI} for interfacing with backends to remain frozen.
To extend or alter the signature of a generic, a new generic can be added, using some form of versioning (e.g. with a numeric suffix, such as `dbAppendTable1()`, `dbAppendTable2()`, etc.).
With such an architecture, arguments in generics could be declared explicitly, without relying on forwarding via `...`, as is done currently.

Furthermore, while presenting the user with a stable API, {DBI} internally decides which method versions to call.
When a new version of a generic is introduced, {DBI} documents and proposes an upgrade path for backend implementers.
In the long run this would also allow for transitioning to another object-oriented system such as S3 or [R7](https://github.com/RConsortium/OOP-WG/) without introducing user-facing breaking changes.

This approach also enables support of rich callbacks: each function in the facade can notify listeners before when called and before returning.
For example, a call to `dbi_connect()` would notify interested parties that a new connection has been established, and a call to `dbi_query()` issues callback with the query and the result.
Potential use cases include:

- Logging as in {dblog}.
- The *Connections* pane in RStudio.
- Mocking (with hooks) as in {dittodb}.

A versioning scheme could also be implemented for callbacks, keeping existing callbacks frozen while allowing for addition of new features that alter callback signatures.

## Acknowledgments

I'd like to thank Jeroen Ooms and Gábor Csárdi for providing crucial infrastructure to support this and many other projects in the R ecosystem.

Thanks the numerous contributors to the packages in the "Maintaining DBI" project in 2021:

- DBI: [&#x0040;bwohl](https://github.com/bwohl), [&#x0040;cboettig](https://github.com/cboettig), [&#x0040;dcassol](https://github.com/dcassol), [&#x0040;jawond](https://github.com/jawond), [&#x0040;krlmlr](https://github.com/krlmlr), [&#x0040;mmccarthy404](https://github.com/mmccarthy404), [&#x0040;pnacht](https://github.com/pnacht), [&#x0040;r2evans](https://github.com/r2evans), [&#x0040;vituri](https://github.com/vituri), and [&#x0040;zoushucai](https://github.com/zoushucai);
- RSQLite: [&#x0040;ablack3](https://github.com/ablack3), [&#x0040;billy34](https://github.com/billy34), [&#x0040;edwindj](https://github.com/edwindj), [&#x0040;gaborcsardi](https://github.com/gaborcsardi), [&#x0040;ggrothendieck](https://github.com/ggrothendieck), [&#x0040;giocomai](https://github.com/giocomai), [&#x0040;habilzare](https://github.com/habilzare), [&#x0040;honghh2018](https://github.com/honghh2018), [&#x0040;Jeff-Gui](https://github.com/Jeff-Gui), [&#x0040;kevinushey](https://github.com/kevinushey), [&#x0040;mgirlich](https://github.com/mgirlich), [&#x0040;plantton](https://github.com/plantton), [&#x0040;schuemie](https://github.com/schuemie), [&#x0040;Shicheng-Guo](https://github.com/Shicheng-Guo), and [&#x0040;tschoonj](https://github.com/tschoonj);
- RPostgres: [&#x0040;aleaficionado](https://github.com/aleaficionado), [&#x0040;armenic](https://github.com/armenic), [&#x0040;ateucher](https://github.com/ateucher), [&#x0040;baderstine](https://github.com/baderstine), [&#x0040;beralef](https://github.com/beralef), [&#x0040;carlganz](https://github.com/carlganz), [&#x0040;ColinFay](https://github.com/ColinFay), [&#x0040;dcaud](https://github.com/dcaud), [&#x0040;dpprdan](https://github.com/dpprdan), [&#x0040;f-ritter](https://github.com/f-ritter), [&#x0040;galachad](https://github.com/galachad), [&#x0040;GitHubGeniusOverlord](https://github.com/GitHubGeniusOverlord), [&#x0040;gontcharovd](https://github.com/gontcharovd), [&#x0040;gtm19](https://github.com/gtm19), [&#x0040;hadley](https://github.com/hadley), [&#x0040;jakob-r](https://github.com/jakob-r), [&#x0040;jeroen](https://github.com/jeroen), [&#x0040;jkylearmstrong](https://github.com/jkylearmstrong), [&#x0040;JSchoenbachler](https://github.com/JSchoenbachler), [&#x0040;matthewgson](https://github.com/matthewgson), [&#x0040;mgirlich](https://github.com/mgirlich), [&#x0040;mmuurr](https://github.com/mmuurr), [&#x0040;mskyttner](https://github.com/mskyttner), [&#x0040;phedinkus](https://github.com/phedinkus), [&#x0040;ppssphysics](https://github.com/ppssphysics), [&#x0040;RakeshG1](https://github.com/RakeshG1), [&#x0040;rickbpdq](https://github.com/rickbpdq), [&#x0040;samiaab1990](https://github.com/samiaab1990), [&#x0040;sawnaanwas](https://github.com/sawnaanwas), [&#x0040;tomasjanikds](https://github.com/tomasjanikds), [&#x0040;vspinu](https://github.com/vspinu), [&#x0040;waynelapierre](https://github.com/waynelapierre), [&#x0040;zmbc](https://github.com/zmbc), and [&#x0040;zozlak](https://github.com/zozlak);
- RMariaDB: [&#x0040;bakiunal](https://github.com/bakiunal), [&#x0040;dirkschumacher](https://github.com/dirkschumacher), [&#x0040;hadley](https://github.com/hadley), [&#x0040;jeroen](https://github.com/jeroen), [&#x0040;Mosk915](https://github.com/Mosk915), [&#x0040;noamross](https://github.com/noamross), [&#x0040;paulmaunders](https://github.com/paulmaunders), [&#x0040;retowyss](https://github.com/retowyss), [&#x0040;rorynolan](https://github.com/rorynolan), [&#x0040;twentytitus](https://github.com/twentytitus), [&#x0040;verajosemanuel](https://github.com/verajosemanuel), [&#x0040;wiligl](https://github.com/wiligl), [&#x0040;Woosah](https://github.com/Woosah), and [&#x0040;zoushucai](https://github.com/zoushucai).
- DBItest: [&#x0040;adamsma](https://github.com/adamsma), and [&#x0040;michaelquinn32](https://github.com/michaelquinn32);
