+++
author = "Kirill Müller"
date = "2020-12-31"
draft = true
weight = 180
title = "Maintaining DBI, 3/4"
description = "Summarizing the progress of 2020"
+++

What is DBI?
------------

DBI stands for **d**ata**b**ase **i**nterface, and DBI is a package for
connecting to database management systems (DBMS). The goal of DBI is to
provide a common interface for accessing a database, regardless of the
specific underlying DBMS.

DBI works with a variety of DBMS, such as Postgres, MariaDB, and SQLite,
allowing users to focus on the specifics of their project instead of
setting up the infrastructure for data import and export.

The DBI package is perfect for anyone looking to connect to a database,
read/write entire tables, and/or execute SQL queries. DBI offers more
control to the user than packages such as
[{dbplyr}](https://dbplyr.tidyverse.org/).

The current version of DBI is 1.1.0. This blog post summarizes recent
developments in DBI and related packages.


- Focus: DBI tutorial. Original "Introduction to DBI" substantially expanded, including a walkthrough against a real database, by James Wondrasek.

- Focus: testing and continuous integration

- Focus: several RMariaDB releases

- Bindings for database libraries: thanks Jeroen Ooms


# Scope of DBI:

- Database interface for R

- Reading/writing tables

- Executing query, fetching data (with parameters)

- Transactions



# Problems identified: see e-mail

- query termination

- table import: performance

- immediate for RMariaDB and RPostgres

- time zones

- reconnect


? data types: arrays, JSON, ...


- SSL

- auth plugins


# Notable changes

- Move to GitHub Actions, thanks Andrew Kane for providing actions to install database engines on all platforms. Using GitHub Actions to automate search for DBI backend packages on GitHub

- DBI: Minor improvements to `dbQuoteLiteral()`, relevant for backends that don't provide their own implementation

- DBItest: Simplify understanding which tests went wrong, simpler backtraces. `test_some()` integrates with {dblog} and shows DBI methods called; needs more work for duckdb. Compatibility with testthat 3.0.0. Better and more robust tests.

- RKazam: Is now a template repository

- RMariaDB: Better handling of data types and character encoding, @ycphs contributed `timezone` argument to `dbConnect()`, minor tweaks with `dbBind()` and `dbQuoteLiteral()`

- RPostgres: new `Redshift()` driver that allows downstream packages to distinguish between Postgres and Amazon RedShift (thanks Hadley Wickham), minor improvements for date and time types, `postgresWaitForNotify()` contributed by `@lentinj`

- RSQLite: faster `dbAppendTable()`, strings and blobs virtually unlimited size (limit 2 GB)






Specification of `immediate` argument to `dbSendQuery()` and friends
--------------------------------------------------------------------

Tom Nolan raised an [issue on
GitHub](https://github.com/r-dbi/DBI/issues/268), requesting to specify
details of the behavior of query execution. It became apparent that the
DBI specification did not account for database drivers where the
execution path is substantially different for queries with or without
parameters. Recent version of DBI mandated the use of a prepared
statement or query for everything.

Similar problems have been noted in MariaDB, Postgres and SQL Server
(when accessed through {odbc}): some statements cannot be executed as
prepared statements, or [prepared statements are
disabled](https://github.com/r-dbi/RPostgres/issues/185). Over the
course of several months, the details of the required extension of this
API were fleshed out.

The `dbSendQuery()`, `dbGetQuery()`, `dbSendStatement()` and
`dbExecute()` methods gain a new `immediate` argument. By setting this
argument to `TRUE`, a direct query is created, allowing to execute
queries that could not be run previously. Arguably, this is one of those
obscure features that are not noted until they are missed.

It is up to the individual backends to add support for this argument.
The default value should be made backward-compatible with the previous
version of DBI 1.0.0. It has already been implemented in the {odbc}
package. Plans to implement this feature in both {RMariaDB} and
{RPostgres} are underway.

### Examples using `immediate`

    library(DBI)
    con <- dbConnect(odbc::odbc(), dsn = "SQLServerConnection")

    # Create local temporary tables:
    # Did not work before, temporary table was removed immediately.
    dbExecute(con, "CREATE TABLE #temp (a integer)", immediate = TRUE)
    dbExecute(con, "INSERT INTO #temp VALUES (1)", immediate = TRUE)

    # Show execution plan:
    # Did not work before, execution plan was never shown
    dbExecute(con, "SET SHOWPLAN_TEXT ON", immediate = TRUE)
    dbGetQuery(con, "SELECT * FROM #temp WHERE a > 0")
    dbExecute(con, "SET SHOWPLAN_TEXT OFF", immediate = TRUE)

Connector objects
-----------------

The existing method in DBI has been to create the driver object and then
call `dbConnect()` with the connection arguments. However there are
times when a user may need to do the following:

-   Separate connection arguments from establishing a connection
-   Serialize the connector to file in order to establish the same
    connection later
-   Maintain multiple connectors in a list for testing different DBMS

In order to address these use cases, users now have the ability to
create a "connector object" that combines the driver and connection
arguments, allowing the user to call `dbConnect()` without additional
arguments. This feature is implemented in {DBI}, and works out of the
box for all DBI backends.

### Example

    library(DBI)

    # Old way:
    drv <- RSQLite::SQLite()
    con <- dbConnect(drv, dbname = ":memory:")
    dbDisconnect(con)

    # New connector object:
    cnr <- new("DBIConnector",
      .drv = RSQLite::SQLite(),
      .conn_args = list(dbname = ":memory:")
    )
    cnr

    ## <DBIConnector><SQLiteDriver>
    ## Arguments:
    ## $dbname
    ## [1] ":memory:"

    con <- dbConnect(cnr)
    dbDisconnect(con)

In addition, arguments can be functions, a useful feature for passwords
and other sensitive connection data.

Logging with the {dblog} package
--------------------------------

When using applications in production, keeping logs is an invaluable
part of a sound infrastructure. The new
[{dblog}](https://github.com/r-dbi/dblog) package is designed to be as
simple as possible. It can be used as a standalone package or in
conjunction with packages like {dbplyr}.

{dblog} helps both with troubleshooting as well as auditing the queries
that that are used to access a database. Similar to Perl's DBI::log, the
goal of {dblog} is to implement logging for arbitrary DBI backends.

{dblog} is straightforward in its use. Start by initializing a logging
driver using `dblog()` prior to connecting to a database management
system. All calls to DBI methods are logged and by default printed to
the console (or redirected to a file). The entirety of the logging
output is runnable R code, so users can copy, paste, and execute the
logging code for debugging.

### Using `dblog()` to connect to SQLite

    library(dblog)

    drv <- dblog(RSQLite::SQLite())
    #> drv1 <- RSQLite::SQLite()

### All calls to DBI methods are logged, by default to the console

    conn <- dbConnect(drv, file = ":memory:")
    #> conn1 <- dbConnect(drv1, file = ":memory:")

    dbWriteTable(conn, "iris", iris[1:3, ])
    #> dbWriteTable(conn1, name = "iris", value = structure(list(Sepal.Length = c(5.1, 4.9,
    #> 4.7), Sepal.Width = c(3.5, 3, 3.2), Petal.Length = c(1.4, 1.4, 1.3), Petal.Width = c(0.2,
    #> 0.2, 0.2), Species = structure(c(1L, 1L, 1L), .Label = c("setosa", "versicolor",
    #> "virginica"), class = "factor")), row.names = c(NA, 3L), class = "data.frame"), overwrite = FALSE,
    #>     append = FALSE)

    data <- dbGetQuery(conn, "SELECT * FROM iris")
    #> dbGetQuery(conn1, "SELECT * FROM iris")
    #> ##   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    #> ## 1          5.1         3.5          1.4         0.2  setosa
    #> ## 2          4.9         3.0          1.4         0.2  setosa
    #> ## 3          4.7         3.2          1.3         0.2  setosa

    dbDisconnect(conn)
    #> dbDisconnect(conn1)

    data
    #>   Sepal.Length Sepal.Width Petal.Length Petal.Width Species
    #> 1          5.1         3.5          1.4         0.2  setosa
    #> 2          4.9         3.0          1.4         0.2  setosa
    #> 3          4.7         3.2          1.3         0.2  setosa

This also works in scenarios where DBI is used under the hood by other
packages like [dbplyr](https://dbplyr.tidyverse.org/) or
[tidypredict](https://tidymodels.github.io/tidypredict/). The log will
represent the DBI operations issued, which allows for a better
understanding of the internals.

Testing your infrastructure for DBI compatibility
-------------------------------------------------

DBItest (on CRAN in version 1.7.0) is currently geared towards usage as
part of a package's test suite. With some effort it is possible to test
a database backend against a custom database. This can help verify that
your database installation gives expected results when accessed with DBI
with specific connection arguments. The [DBItest
article](https://dbitest.r-dbi.org/articles/dbitest#external-testing)
contains a new section that describes how to achieve this, including a
primer on using {dblog} to understand the cause of test failures.

A list of DBI backends
----------------------

The new [backends](https://github.com/r-dbi/backends) repository lists
all known DBI backends, as retrieved via a code search on GitHub. The
list is available in the README, and as a static web API for
programmatic processing.

Better handling of time zones
-----------------------------

Time zones are used to convert between *absolute* time and *civil* time,
where absolute time exists independent of human-created measures such as
calendars, days, and dates, whereas civil time is comprised of years,
months, days, hours, minutes, and seconds. For a more in-depth reading
on absolute time, civil time, and time zones, please read this excerpt
from the [ODBC
README](https://github.com/r-dbi/odbc/blob/7b35549f9df935e1d132f6221860f87a6eb64ef6/src/cctz/README.md).

For programming and data analysis, accurate handling time zones is
crucial. {odbc} has set an example for how to handle time zones through
the inclusion of `timezone` and `timezone_out` arguments to
`dbConnect()`. The `timezone` argument controls the server time zone,
which may be different from UTC. The `timezone_out` argument specifies
the time zone to use for displaying times.

This strategy gives the user control over datetime information passed on
to and retrieved from the database. Both arguments in combination should
be able to support a broad variety of use cases and server setups.
{RMariaDB} and {RPostgres} will incorporate this strategy with their
next CRAN release. {RPostgres} already has gained a `timezone` argument
in its `dbConnect()` method.

Window function support in {RSQLite}
------------------------------------

RSQLite 2.1.4 and later includes sqlite &gt;= 3.29.0, which introduces
support for window functions.

    library(tidyverse)
    library(dbplyr)

    tbl <- memdb_frame(a = rep(1:2, 5), b = 1:10)

    tbl %>%
      group_by(a) %>%
      window_order(b) %>%
      mutate(c = cumsum(b)) %>%
      ungroup()

    ## # Source:     lazy query [?? x 3]
    ## # Database:   sqlite 3.30.1 [:memory:]
    ## # Ordered by: b
    ##        a     b     c
    ##    <int> <int> <int>
    ##  1     1     1     1
    ##  2     1     3     4
    ##  3     1     5     9
    ##  4     1     7    16
    ##  5     1     9    25
    ##  6     2     2     2
    ##  7     2     4     6
    ##  8     2     6    12
    ##  9     2     8    20
    ## 10     2    10    30

CII "best practices" badges for all repos
-----------------------------------------

CII "best practices" have been implemented for {DBItest}, {RMariaDB},
{RPostgres} and {RSQLite}. The {DBItest} repository has a brand-new
badge, the badges for the other repositories will follow suit.

Package updates
---------------

The following package versions were sent to CRAN in conjunction with
this blog post:

-   DBI 1.1.0 ([NEWS](https://dbi.r-dbi.org/news/))
-   DBItest 1.7.0 ([NEWS](https://dbitest.r-dbi.org/news/))
-   RMariaDB 1.0.8 ([NEWS](https://rmariadb.r-dbi.org/news/))
-   RPostgres 1.2.0 ([NEWS](https://rpostgres.r-dbi.org/news/))
-   RSQLite 2.1.5 ([NEWS](https://rsqlite.r-dbi.org/news/))

Before that, minor updates of the database backend packages were
necessary to comply with stricter CRAN checks and toolchain updates.

Future work
-----------

The remainder of the blog post discusses future directions for DBI and
the backend packages.

### DBI tutorials

Improving documentation is a priority. DBI is still lacking an
up-to-date tutorial with a low entry bar that helps users connect to
their database and execute queries. The updated
[README](https://dbi.r-dbi.org/) is a little step forward, but a
slightly more comprehensive versions with link to more detailed
information would be helpful.

### Terraforming databases

Now that using DBItest to test backends against custom infrastructure is
understood, it becomes easier to enhance tests so that not only pristine
setups are tested, but also databases with nonstandard settings for time
zone, character encoding or collation.
[Terraform](https://www.terraform.io/) helps automating the setup of
databases of different flavors on cloud providers such as Azure or
Google Cloud. The desired state of computing infrastructure is specified
in a declarative way. This allows testing a much broader variety of
databases and configurations, without maintaining expensive
infrastructure: databases can be spun up when needed and torn down when
done.

It would be helpful to have a selection of open-source and commercial
databases in different configuration settings ready for testing.

### Query cancellation

Currently, {odbc} and many other backends freeze while a query is
executed. It is easy to underestimate the runtime of a query, or to
accidentally execute a query that is running too long. This severely
hampers interactive workflows: a frozen R session means forcibly
restarting R, or worse, the development environment.

Mateusz Żółtak has contributed a [pull
request](https://github.com/r-dbi/RPostgres/pull/193) that implements
support for graceful query cancellation in {RPostgres}. [Initial
research](https://github.com/r-dbi/odbc/issues/312) suggests that for
{odbc} it may be possible to implement this in a similar fashion. It
remains to be seen if an implementation is viable, and if the database
libraries used by {RMariaDB} and {RSQLite} support this mode of
operation.

Acknowledgments
---------------

I'd like to thank Katharina Brunner and Jesse Mostipak for help with
composing this blog post.
