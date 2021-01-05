+++
author = "Kirill Müller"
date = "2021-01-04"
draft = false
weight = 180
title = "Maintaining DBI, 3/4"
description = "Summarizing the progress of 2020"
+++

What is DBI?
------------

DBI stands for **d**ata**b**ase **i**nterface.
The DBI package connects R to database management systems (DBMS).
The goal of DBI is to
provide a common interface for database access,
regardless of the
specific underlying DBMS.
DBI works with a variety of DBMS, such as Postgres, MariaDB, and SQLite, through dedicated [backend packages](https://github.com/r-dbi/backends#readme).
For first-time users I recommend the new [introductory tutorial](https://dbi.r-dbi.org/articles/dbi).

The current version of DBI is 1.1.1.
This blog post attempts to define the scope of the DBI project, summarizes recent developments in DBI and related packages, and showcases future work.


Scope of the DBI project
------------------------

The DBI package is perfect for anyone looking to connect to a database,
read/write entire tables, and/or execute SQL queries.
DBI gives a direct access to the database driver,
leaving more sophisticated data query and manipulation tasks to packages like [dbplyr](https://dbplyr.tidyverse.org/), [dbx](https://github.com/ankane/dbx) and [rquery](https://winvector.github.io/rquery/).

The core DBI project in R provides an interface for databases, specified in textual form and via automated tests.
The [DBI specification](https://dbi.r-dbi.org/articles/spec) contains a detailed description of the methods provided by DBI.
In summary, the interface covers:

- Discovery of tables, also in schemas

- Reading/writing/creating/removing tables

- Executing queries, fetching data (with parameters)

- Safe quoting: low-level composition of queries

- Transactions

DBI should provide a way to ingest data of any type into R, at least in serialized form (e.g. string or [blob](https://en.wikipedia.org/wiki/Binary_large_object)).
It should offer a robust reliable interface for dependent packages; anything beyond this scope should be left to packages that extend DBI:

- arkdb: archival of database data

- connection: integrate database connections with the RStudio IDE

- dbplyr and rquery: generation of SQL queries

- dbx: DBI extension for data manipulation

- dittodb: mocking for databases

- dm: relational data models (via dbplyr)

- pool: connection pooling

- sqlr: schema definition

and many more.


Recent developments in DBI
--------------------------

This section discusses:

- the new DBI tutorials,

- improvements for  datetime data,

- other notable changes,

- the move to GitHub Actions.

The first three items directly affect DBI users, the last item much less so: it is nevertheless an important investment in the stability of the DBI infrastructure.

### New tutorials

James Wondrasek substantially expanded the  "Introduction to DBI" article and added a second article.
DBI now features two tutorials.
The [introduction](https://dbi.r-dbi.org/articles/dbi) includes a walkthrough that describes connecting and querying a real database.
The ["Advanced DBI usage"](https://dbi.r-dbi.org/articles/dbi-advanced) tutorial shows more advanced examples of quoting and parameter binding.
The tutorials are an important first-hand resource for new users.


### Time zones

To date, it was only possible to work reliably with time zones when the database connection represented all times in UTC.
This poses a few problems in practice:

- Not all databases store timestamps as UTC or with time zone offset, often local time is assumed by the data model.

- Other systems often use the default setting for time zone, this harms interoperability of DBI in these cases.

- Conversion of timestamps to dates via the SQL function `DATE` is only correct when the session time zone set correctly.

RMariaDB 1.1.0 and RPostgres 1.3.0 gained more robust support for datetime values.
As proposed in the [previous blog post](https://www.r-dbi.org/blog/dbi-3-2/), new arguments `timezone` and `timezone_out` were added.
Both arguments should use Olson names such as `Europe/Berlin` or `America/New_York`, not time offsets like `+01:00`; the latter may change with daylight time savings season.
If `timezone` is set to `NULL`, an attempt is made to detect the correct time zone on the database.
Thanks to Philipp Schauberger for contributing the initial `timezone` argument for RMariaDB.

RSQLite does not natively support dates or times.
A promising [pull request](https://github.com/r-dbi/RSQLite/pull/333) is underway that implements support for treating numeric values as time offsets if the column type is declared in a specific way.


### Notable changes to DBI backends

The following package versions were sent to CRAN since the last blog post:

- DBI 1.1.0 -> 1.1.1 ([NEWS](https://dbi.r-dbi.org/news/))
- RMariaDB 1.0.8 -> 1.1.0 ([NEWS](https://rmariadb.r-dbi.org/news/))
- RPostgres 1.2.0 -> 1.3.0 ([NEWS](https://rpostgres.r-dbi.org/news/))
- RSQLite 2.1.5 -> 2.2.2 ([NEWS](https://rsqlite.r-dbi.org/news/))

Highlights are:

- DBI: Two new tutorials; minor improvements to `dbQuoteLiteral()`, this is relevant for backends that don't provide their own implementation.

- RMariaDB: Better handling of data types and character encoding; minor tweaks to `dbBind()` and `dbQuoteLiteral()`.

- RPostgres: The new `Redshift()` driver that allows downstream packages to distinguish between Postgres and Amazon RedShift (thanks Hadley Wickham); minor improvements for querying and passing date and time types, `postgresWaitForNotify()` contributed by Jamie Lentin.

- RSQLite: `dbAppendTable()` is faster, strings and blobs can have virtually unlimited size (limit 2 GB), embedded SQLite library is now in version 3.34.

- DBItest: understanding which tests failed is now simpler, also thanks to simpler backtraces; `test_some()` integrates with the dblog package and shows DBI methods called; established compatibility with testthat 3.0.0; better and more robust tests.

- RKazam: Is now a template repository

Thanks to Jeroen Ooms for maintaining Windows versions for the database libraries.


### QA and automation

Automated tests are a crucial part of modern software engineering.
These are often augmented with continuous integration (CI) services that run these tests regularly or with every change to the code.
When I started working on DBI, Travis CI offered excellent continuous integration services for open-source repositories.
Unfortunately, this is no longer the case: the free tier introduced a limit on CI build time, rendering it effectively unusable for DBI.

[GitHub Actions](https://github.com/features/actions) is a CI/CD platform tightly integrated with GitHub.
It is somewhat simpler to set up, also for creating workflows that e.g. open a pull request.
It is sufficient to add a YAML configuration file to a dedicated location in the repository.
Each build automatically obtains a token that can be used to interact with the GitHub API.
R support is provided by [dedicated workflows and actions](https://github.com/r-lib/actions) contributed by RStudio.
Check status is conveniently reported in detail with each pull request, and the checks run considerably faster due to higher concurrency.

Continuous integration for all packages in the project has moved to GitHub Actions.
Cross-platform checks for all backends on the major operating systems were a bit challenging, because the tests require a live database.
Thanks to Andrew Kane for providing GitHub actions that install database engines on all platforms, this greatly simplified the move.

Three more parts of the infrastructure were updated as part of the move:

1. The odbc and duckdb packages are now also checked when the DBItest package updates.
  This ensures that new or amended specifications do not break these packages.
  If you maintain a DBI backend that uses DBItest, get in touch for integrating your backend with these checks.

1. The [list of DBI backends](https://github.com/r-dbi/backends) is now continuously updated.
  Updates to backends are applied automatically.
  Every time a new backend is found, a pull request is opened.

1. A new [pull request](https://github.com/r-dbi/RSQLite/pull/337) is opened in RSQLite when a new version of the SQLite library is available.
  This makes it much easier to keep the bundled SQLite version up to date.


Future work
-----------

The last blog post already identified major milestones:

- query cancellation

- testing on remote databases

A triage of the contributed issues has identified the following additional major topics:

- `immediate` argument to `dbSendQuery()` and `dbSendStatement()` for RMariaDB and RPostgres

- performance of table import

- reconnect if a database connection is lost

Other minor issues include:

- SSL connections

- authentification plugins

- support for more data types: arrays, JSON, ...

I'm planning to resolve most of the remaining issues in a final sprint.
Some of these issues can be outsourced to other packages, according to the scope outlined in the previous sections, priority should be given to issues that must be resolved in the core packages.
Future work might shift towards providing or improving useful extensions.


Acknowledgments
---------------

I'd like to thank James Wondrasek for creating the DBI tutorials and for a review of this blog post, Angelica Becerra for reviewing the material, and the numerous contributors to the packages in the "Maintaining DBI" project (DBI¹, RSQLite², RPostgres³, RMariaDB⁴, and DBItest⁵):

[&#x0040;abalter](https://github.com/abalter)³, [&#x0040;alanpaulkwan](https://github.com/alanpaulkwan)², [&#x0040;AllenSuttonValocity](https://github.com/AllenSuttonValocity)³, [&#x0040;altay-oz](https://github.com/altay-oz)³, [&#x0040;anderic1](https://github.com/anderic1)², [&#x0040;andybeet](https://github.com/andybeet)¹, [&#x0040;arencambre](https://github.com/arencambre)⁴, [&#x0040;artemklevtsov](https://github.com/artemklevtsov)³, [&#x0040;bastianilso](https://github.com/bastianilso)⁴, [&#x0040;bczernecki](https://github.com/bczernecki)¹, [&#x0040;Byggvir](https://github.com/Byggvir)⁴, [&#x0040;Chrisjb](https://github.com/Chrisjb)¹, [&#x0040;clementbfeyt](https://github.com/clementbfeyt)⁴, [&#x0040;colearendt](https://github.com/colearendt)¹, [&#x0040;daattali](https://github.com/daattali)¹, [&#x0040;datawookie](https://github.com/datawookie)¹, [&#x0040;dpprdan](https://github.com/dpprdan)³⁵, [&#x0040;elfatherbrown](https://github.com/elfatherbrown)⁴, [&#x0040;EntwicklR](https://github.com/EntwicklR)², [&#x0040;ericemc3](https://github.com/ericemc3)⁴, [&#x0040;formix](https://github.com/formix)³, [&#x0040;fproske](https://github.com/fproske)¹, [&#x0040;georgevbsantiago](https://github.com/georgevbsantiago)¹, [&#x0040;github-actions[bot]](https://github.com/github-actions[bot])², [&#x0040;GitHunter0](https://github.com/GitHunter0)¹, [&#x0040;hadley](https://github.com/hadley)²³, [&#x0040;hmeleiro](https://github.com/hmeleiro)¹, [&#x0040;hpages](https://github.com/hpages)², [&#x0040;imlijunda](https://github.com/imlijunda)³, [&#x0040;inferiorhumanorgans](https://github.com/inferiorhumanorgans)³, [&#x0040;jarauh](https://github.com/jarauh)⁴, [&#x0040;jawond](https://github.com/jawond)¹, [&#x0040;jeroen](https://github.com/jeroen)³, [&#x0040;jimhester](https://github.com/jimhester)¹, [&#x0040;jjesusfilho](https://github.com/jjesusfilho)³, [&#x0040;jsilve24](https://github.com/jsilve24)², [&#x0040;kforner](https://github.com/kforner)⁴, [&#x0040;kmishra9](https://github.com/kmishra9)², [&#x0040;Kodiologist](https://github.com/Kodiologist)¹², [&#x0040;LaugeGregers](https://github.com/LaugeGregers)³, [&#x0040;luispuerto](https://github.com/luispuerto)², [&#x0040;martinstuder](https://github.com/martinstuder)⁵, [&#x0040;matteodelucchi](https://github.com/matteodelucchi)⁴, [&#x0040;MaximumV](https://github.com/MaximumV)¹, [&#x0040;mbannert](https://github.com/mbannert)³, [&#x0040;mbedward](https://github.com/mbedward)³, [&#x0040;mgirlich](https://github.com/mgirlich)², [&#x0040;mlamias](https://github.com/mlamias)¹, [&#x0040;mllg](https://github.com/mllg)¹, [&#x0040;mmuurr](https://github.com/mmuurr)³, [&#x0040;momeara](https://github.com/momeara)³, [&#x0040;MonteShaffer](https://github.com/MonteShaffer)⁴, [&#x0040;Mosk915](https://github.com/Mosk915)⁴, [&#x0040;nfultz](https://github.com/nfultz)², [&#x0040;norquanttech](https://github.com/norquanttech)³, [&#x0040;OMalytics](https://github.com/OMalytics)³, [&#x0040;oriolcmp](https://github.com/oriolcmp)⁴, [&#x0040;Osc2wall](https://github.com/Osc2wall)⁴, [&#x0040;psychobas](https://github.com/psychobas)², [&#x0040;randyzwitch](https://github.com/randyzwitch)⁵, [&#x0040;rcfree](https://github.com/rcfree)², [&#x0040;rnorberg](https://github.com/rnorberg)¹, [&#x0040;rodriguesk](https://github.com/rodriguesk)², [&#x0040;rossholmberg](https://github.com/rossholmberg)⁴, [&#x0040;Sahil308](https://github.com/Sahil308)⁴, [&#x0040;samuel-cs4](https://github.com/samuel-cs4)⁴, [&#x0040;schuemie](https://github.com/schuemie)², [&#x0040;shutinet](https://github.com/shutinet)², [&#x0040;splaisan](https://github.com/splaisan)², [&#x0040;Trowic](https://github.com/Trowic)⁴, [&#x0040;verajosemanuel](https://github.com/verajosemanuel)⁴, [&#x0040;VictorYammouni](https://github.com/VictorYammouni)¹, [&#x0040;vigyoyo](https://github.com/vigyoyo)⁴, [&#x0040;vikram-rawat](https://github.com/vikram-rawat)³, [&#x0040;vspinu](https://github.com/vspinu)³, [&#x0040;warnes](https://github.com/warnes)³, [&#x0040;wiligl](https://github.com/wiligl)², [&#x0040;ycphs](https://github.com/ycphs)⁴, and [&#x0040;zyxdef](https://github.com/zyxdef)¹.
