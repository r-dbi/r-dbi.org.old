+++
author = "Kirill Müller"
date = "2018-04-30"
draft = false
weight = 180
title = "Done “Establishing DBI”!?"
description = "Summary of the “Establishing DBI” project"
+++

The "Establishing DBI" project, funded by the R consortium, started
about a year ago. It includes the completion of two new backends,
*RPostgres* and *RMariaDB*, and a quite a few interface extensions and
specifications. This blog post showcases only the visible changes, a
substantial amount of work went into extending the DBI specification and
making the three open-source database backends compliant to it. Learn
more about DBI, R's database interface, on <https://r-dbi.org>.

Release of *RPostgres* and *RMariaDB*
-------------------------------------

The DBI specification has been formulated in the preceding R consortium
project, "Improving DBI". It is both an automated test suite and a
human-readable description of behavior, implemented in the *DBItest*
package. For this project, I extended this specification and could also
use it to implement *RPostgres* and *RMariaDB*: for once, test-driven
development was pure pleasure, because the tests were already there!

I took over maintenance of the *RPostgres* and *RMariaDB* packages,
which are complete rewrites of the *RPostgreSQL* and *RMySQL* packages,
respectively. These packages use C++ (with *Rcpp*) as glue between R and
the native database libraries. A reimplementation and release under a
different name has made it much easier to fully conform to the DBI
specification: only listing temporary tables and casting to blob or
character is not supported by *RMariaDB* (due to a limitation of the
DBMS), all other parts of the specification are fully covered.

Projects that use *RPostgreSQL* or *RMySQL* can continue to do so, or
switch to the new backends at their own pace (which likely requires some
changes to the code). For new projects I recommend *RPostgres* or
*RMariaDB* to take advantage of the thorougly tested codebases and of
the consistency across backends.

Schema support
--------------

Consistent access of tables in database schemas was planned for the
"Improving DBI" project already, but I have implemented it only
recently. It felt safer to see how the interface works on three
backends, as opposed to implementing it for just *RSQLite* and then
perhaps having to adapt it.

The new `Id()` function constructs identifiers. All arguments must be
named, yet *DBI* doesn't specify the argument names, because DBMS have
an inconsistent notion of namespaces. The objects returned by `Id()` are
"dumb", they gain meaning only when used in methods such as
`dbQuoteIdentifier()` or `dbWriteTable()`.

For listing database objects in schemas, the new `dbListObjects()`
generic can be used. It returns a data frame that contains identifiers
(like those created by the `Id()` function) and a flag that indicates if
the identifier is complete (i.e., pointing to a table or view) or a
prefix. Incomplete identifiers can be passed to `dbListObjects()` again,
which allows traversing the tree of database objects.

The following example assumes a schema `my_schema`. A table named
`my_table` is created in this schema, objects are listed, and the table
is read again.

    library(RPostgres)
    pg_conn <- dbConnect(Postgres())

    table_name <- Id(schema = "my_schema", table = "my_table")
    table_name

    ## <Id> schema = my_schema, table = my_table

    data <- data.frame(a = 1:3, b = letters[1:3])
    dbWriteTable(pg_conn, table_name, data)

    dbListObjects(pg_conn)

    ##                               table is_prefix
    ## 1    <Id> table = geography_columns     FALSE
    ## 2     <Id> table = geometry_columns     FALSE
    ## 3      <Id> table = spatial_ref_sys     FALSE
    ## 4       <Id> table = raster_columns     FALSE
    ## 5     <Id> table = raster_overviews     FALSE
    ## 6             <Id> table = topology     FALSE
    ## 7                <Id> table = layer     FALSE
    ## 8                 <Id> table = temp     FALSE
    ## 9            <Id> schema = topology      TRUE
    ## 10          <Id> schema = my_schema      TRUE
    ## 11 <Id> schema = information_schema      TRUE
    ## 12         <Id> schema = pg_catalog      TRUE
    ## 13             <Id> schema = public      TRUE

    dbListObjects(
      pg_conn,
      prefix = Id(schema = "my_schema")
    )

    ##                                       table is_prefix
    ## 1 <Id> schema = my_schema, table = my_table     FALSE

    dbReadTable(pg_conn, table_name)

    ##   a b
    ## 1 1 a
    ## 2 2 b
    ## 3 3 c

In addition to `dbReadTable()` and `dbWriteTable()`, also
`dbExistsTable()` and `dbRemoveTable()` and the new `dbCreateTable()`
and `dbAppendTable()` (see below) support an `Id()` object as table
name. The `dbQuoteIdentifier()` method converts these objects to SQL
strings. Some operations (e.g. checking if a table exists) require the
inverse, the new `dbUnquoteIdentifier()` generic takes care of
converting valid SQL identifiers to (a list of) `Id()` objects:

    quoted <- dbQuoteIdentifier(pg_conn, table_name)
    quoted

    ## <SQL> "my_schema"."my_table"

    dbUnquoteIdentifier(pg_conn, quoted)

    ## [[1]]
    ## <Id> schema = my_schema, table = my_table

The new methods work consistently across backends, only *RSQLite* is
currently restricted to the default schema. (Schemas in *RSQLite* are
created by attaching another database, this use case seemed rather
exotic but can be supported with the new infrastructure.)

Quoting literal values
----------------------

When working on the database backends, it has become apparent that
quoting strings and identifiers isn't quite enough. Now there is a way
to quote arbitrary values, i.e. convert them to a string that can be
pasted into an SQL query:

    library(RSQLite)
    sqlite_conn <- dbConnect(SQLite())

    library(RMariaDB)
    mariadb_conn <- dbConnect(MariaDB(), dbname = "test")

    dbQuoteLiteral(sqlite_conn, 1.5)

    ## <SQL> 1.5

    dbQuoteLiteral(mariadb_conn, 1.5)

    ## <SQL> 1.5

    dbQuoteLiteral(pg_conn, 1.5)

    ## <SQL> 1.5::float8

    dbQuoteLiteral(mariadb_conn, Sys.time())

    ## <SQL> '20180501012806'

    dbQuoteLiteral(pg_conn, Sys.time())

    ## <SQL> '2018-05-01 03:28:06'::timestamp

The default implementation works for ANSI SQL compliant DBMS, the method
for *RPostgres* takes advantage of the `::` casting operator as seen in
the examples.

More fine-grained creation of tables
------------------------------------

*DBI* supports storing data frames as tables in the database via
`dbWriteTable()`. This operation consists of multiple steps:

-   Checking if a table of this name exists, if yes:
    -   If `overwrite = TRUE`, removing the table
    -   If not, throwing an error
-   Creating the table with the correct field structure
-   Preparing the data for writing
-   Writing the data

To reduce complexity and allow for more options without cluttering the
argument list of `dbWriteTable()`, *DBI* now provides generics for the
individual steps:

-   The existing `dbRemoveTable()` generic has been extended with
    `temporary` and `fail_if_missing` arguments. Setting
    `temporary = TRUE` makes sure that only temporaries are removed. By
    default, trying to remove a table that doesn't exist fails, setting
    `fail_if_missing = FALSE` changes this behavior to a silent success.

-   The new `dbCreateTable()` generic accepts a data frame or a
    character vector of DBMS data types and creates a table in the
    database. It builds upon the existing `sqlCreateTable()` generic and
    also supports the `temporary` argument. If a table by that name
    already exists, an error is raised.

-   The new `dbAppendTable()` generic uses a prepared statement (created
    via `sqlAppendTableTemplate()`) to efficiently insert rows into the
    database. This avoids the internal overhead of converting values to
    SQL literals.

The following example shows the creation and population of a table with
the new methods.

    table_name

    ## <Id> schema = my_schema, table = my_table

    dbRemoveTable(pg_conn, table_name, fail_if_missing = FALSE)

    dbCreateTable(pg_conn, table_name, c(a = "int8", b = "float8"))

    dbAppendTable(pg_conn, table_name, data.frame(a = 1:3, b = 1:3))

    ## [1] 3

    str(dbReadTable(pg_conn, table_name))

    ## 'data.frame':    3 obs. of  2 variables:
    ##  $ a:integer64 1 2 3 
    ##  $ b: num  1 2 3

The `dbWriteTable()` methods in the three backends have been adapted to
use the new methods.

Support for 64-bit integers
---------------------------

As seen in the previous example, 64-bit integers can be read from the
database. The three backends *RSQLite*, *RPostgres* and *RMariaDB* now
also support writing 64-bit integers via the *bit64* package:

    data <- data.frame(a = bit64::as.integer64(4:6), b = 4:6)
    dbAppendTable(pg_conn, table_name, data)

    ## [1] 3

    str(dbReadTable(pg_conn, table_name))

    ## 'data.frame':    6 obs. of  2 variables:
    ##  $ a:integer64 1 2 3 4 5 6 
    ##  $ b: num  1 2 3 4 5 6

Because R still lacks support for native 64-bit integers, the *bit64*
package feels like the best compromise: the returned values can be
computed on, or coerced to `integer`, `numeric` or even `character`
depending on the application. In some cases, it may be useful to always
coerce. This is where the new `bigint` argument to `dbConnect()` helps:

    pg_conn_int <- dbConnect(Postgres(), bigint = "integer")
    str(dbReadTable(pg_conn_int, table_name))

    ## 'data.frame':    6 obs. of  2 variables:
    ##  $ a: int  1 2 3 4 5 6
    ##  $ b: num  1 2 3 4 5 6

    pg_conn_num <- dbConnect(Postgres(), bigint = "numeric")
    str(dbReadTable(pg_conn_num, table_name))

    ## 'data.frame':    6 obs. of  2 variables:
    ##  $ a: num  1 2 3 4 5 6
    ##  $ b: num  1 2 3 4 5 6

    pg_conn_chr <- dbConnect(Postgres(), bigint = "character")
    str(dbReadTable(pg_conn_chr, table_name))

    ## 'data.frame':    6 obs. of  2 variables:
    ##  $ a: chr  "1" "2" "3" "4" ...
    ##  $ b: num  1 2 3 4 5 6

The `bigint` argument works consistently across the three backends
*RSQLite*, *RPostgres* and *RMariaDB*, the DBI specification contains a
test for and a description of the requirements.

Geometry columns
----------------

PostgreSQL has support for user-defined data types, this is used e.g. by
PostGIS to store spatial data. Before, user-defined data types were
returned as character values, with a warning. Thanks to a contribution
by Etienne B. Racine:

-   the warnings are gone,
-   the user-defined data type is now stored in an attribute of the
    column in the data frame,
-   details on columns with user-defined data types are available in
    `dbColumnInfo()`.

<!-- -->

    dbCreateTable(
      pg_conn,
      "geom_test",
      c(id = "int4", geom = "geometry(Point, 4326)")
    )

    data <- data.frame(
      id = 1,
      geom = "SRID=4326;POINT(-71.060316 48.432044)",
      stringsAsFactors = FALSE
    )
    dbAppendTable(pg_conn, "geom_test", data)

    ## [1] 1

    str(dbReadTable(pg_conn, "geom_test"))

    ## 'data.frame':    1 obs. of  2 variables:
    ##  $ id  : int 1
    ##  $ geom:Class 'pq_geometry'  chr "0101000020E61000003CDBA337DCC351C06D37C1374D374840"

    res <- dbSendQuery(pg_conn, "SELECT * FROM geom_test")
    dbColumnInfo(res)

    ##   name      type   .oid .known .typname
    ## 1   id   integer     23   TRUE     int4
    ## 2 geom character 101529  FALSE geometry

    dbClearResult(res)

Special support for geometry columns is currently available only in
*RPostgres*.

Duplicate column names
----------------------

The specification has been extended to disallow duplicate, empty or `NA`
column names. The deduplication used by our three backends is similar to
that used by `tibble::set_tidy_names()`, but the DBI specification does
not require any particular deduplication mechanism. Syntactic names
aren't required either:

    dbGetQuery(sqlite_conn, "SELECT 1, 2, 3")

    ##   1 2 3
    ## 1 1 2 3

    dbGetQuery(sqlite_conn, "SELECT 1 AS a, 2 AS a, 3 AS `a..2`")

    ##   a a..2 a..3
    ## 1 1    2    3

    dbGetQuery(mariadb_conn, "SELECT 1, 2, 3")

    ##   1 2 3
    ## 1 1 2 3

    dbGetQuery(mariadb_conn, "SELECT 1 AS a, 2 AS a, 3 AS `a..2`")

    ##   a a..2 a..3
    ## 1 1    2    3

    dbGetQuery(pg_conn, "SELECT 1, 2, 3")

    ##   ?column? ?column?..2 ?column?..3
    ## 1        1           2           3

    dbGetQuery(pg_conn, 'SELECT 1 AS a, 2 AS a, 3 AS "a..2"')

    ##   a a..2 a..3
    ## 1 1    2    3

Helpers
-------

Two little helper generics have been added.

The new `dbIsReadOnly()` generic (contributed by Anh Le) should return
`TRUE` for a read-only connection. This is not part of the specification
yet.

The `dbCanConnect()` tests a set of connection parameters. The default
implementation simply connects and then disconnects upon success. For
DBMS that can provide more efficient methods of checking connectivity, a
lighter-weight implementation of this method may give a better
experience.

None of the three backends currently provide specialized implementations
for these generics.

Code reuse
----------

I have made some efforts to extract common C++ classes for assembling
data frames and prepare them for reuse. The C++ source code for the
three backends contains files prefixed with `Db`, these are almost
identical across the backends. The planned packaging into the *RKazam*
package had to yield to higher-priority features described above.

The situation in the R code is similar: I have found myself copy-pasting
code from one backend into another because I didn't feel it's ready (or
standardized enough) to be included in the *DBI* package.

For both use cases, a code reuse strategy based on copying/updating
template files or reconciling files may be more robust than the
traditional importing mechanisms offered by R.

Outlook
-------

The upcoming CRAN release of *DBI*, *DBItest* and the three backends
*RSQLite*, *RMariaDB* and *RPostgres* are an important milestone.
Stability is important when more and more users and projects use the new
backends. Nevertheless, I see quite a few potential improvements that so
far were out of scope of the "Improving DBI" and "Establishing DBI"
projects:

1.  Support running the test suite locally, to validate adherence to DBI
    for a particular installation.

2.  Consistent fast data import.

3.  Consistent query placeholders (currently `$1` for *RPostgres* and
    `?` for many other backends).

4.  Support for arbitrary data types via hooks.

5.  Assistance with installation problems on specific architectures, or
    connectivity problems with certain databases, or other specific
    issues.

6.  Rework the internal architecture of *DBItest* to simplify locating
    test failures.

7.  Improve the <https://r-dbi.org> website.

8.  Non-blocking queries.

I have submitted another proposal to the R Consortium, hoping to receive
support with these and other issues.

Acknowledgments
---------------

I'd like to thank the R Consortium for their generous financial support.
Many thanks to the numerous contributors who helped make the past two
projects a success.
