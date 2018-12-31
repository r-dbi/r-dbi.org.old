+++
author = "Kirill Müller"
date = "2018-12-31"
draft = false
weight = 180
title = "Maintaining DBI, 1/4"
description = "Summarizing the progress of 2018"
+++

Much earlier this year my proposal for the third R Consortium project
for working on DBI has been accepted. DBI is a set of virtual functions
declared in the *DBI* package. Communication with the database is
implemented by *DBI backends*, packages that import *DBI* and implement
its methods. A common interface is helpful for both users and backend
implementers. Users, including package developers for DBMS-agnostic
packages, need to memoize only one set of functions. Backend developers
can focus on functionality instead of design decisions, and can benefit
from a large base of potential users right from the start.

I'm grateful for the trust, and will do my best to make the "Maintaining
DBI" project a success. For this round, the main goals are: maintain,
enhance, disseminate. The project is delayed mostly becase I grossly
underestimated how much time and energy it would take to set up
[cynkra](http://cynkra.com). The new joint venture with Christoph Sax
consults businesses and organizations on matters related to R,
statistics, data, and software. We are strongly committed to R and
open-source software, and more priority will be given the "Maintaining
DBI" project next year.

This blog post, much later than planned, summarizes the efforts of the
past year: presentations at meetups, the "Core Infrastructure
Initiative" badge, and activity in the various repositories of the
[r-dbi](https://github.com/r-dbi) GitHub organization. I'll repeat the
big picture issues from the proposal and present plans for future
development.

Presentations at meetups
------------------------

I presented DBI at the [Berlin R user
group](https://www.meetup.com/Berlin-R-Users-Group/), at the
[amstRdays](https://amsterdam2018.satrdays.org/), and at the [Zurich R
meetup](https://zurich-r-user-group.github.io/index.html). The
presentation in Berlin made me realize that a progress report isn't that
helpful for a general audience. The Zurich version of the presentation
featured a DBI intro also suitable for new users, merely highlighting
recent developments. [These slides](https://bit.ly/2wFJix3), and the
[intro at db.rstudio.com](https://db.rstudio.com/dbi/) seem to be the
most recent general-purpose introduction materials available. I think an
entry-level tutorial would be a good fit for a DBI vignette.

CII badge
---------

From
<a href="https://bestpractices.coreinfrastructure.org/en" class="uri">https://bestpractices.coreinfrastructure.org/en</a>:

> The Linux Foundation (LF) Core Infrastructure Initiative (CII) Best
> Practices badge is a way for Free/Libre and Open Source Software
> (FLOSS) projects to show that they follow best practices. Projects can
> voluntarily self-certify, at no cost, by using this web application to
> explain how they follow each best practice. The CII Best Practices
> Badge is inspired by the many badges available to projects on GitHub.
> Consumers of the badge can quickly assess which FLOSS projects are
> following best practices and as a result are more likely to produce
> higher-quality secure software.

The CII badge can be obtained after a self-certification process that
comprises ~70 soft and hard questions about the project around the
following topics:

-   Basics: Project URLs, license, documentation
-   Change Control: Version control and version numbers
-   Reporting: Tracking issues and vulnerabilities
-   Quality: Build and test system, best practices
-   Security (software)
-   Analysis (static and dynamic)

After completing the process, projects are entitled to wear a badge like
the one below for the DBI project:

[![CII Best
Practices](https://bestpractices.coreinfrastructure.org/projects/1882/badge)](https://bestpractices.coreinfrastructure.org/projects/1882)

A click on the badge takes you to the detailed assessment. In addition
to the badge, completing the self-certification allows the maintainer to
rethink if workflows and practices can be improved.

DBI is currently has the "passing" status, the backend packages and
*DBItest* will follow. I have compiled the changes that were necessary
to obtain that status below. It appears to be much more difficult but
not impossible to obtain the "silver" status.

### Necessary changes to the DBI package

Several files had to be added or updated:

1.  [`CONTRIBUTING.md`](https://github.com/r-dbi/DBI/blob/master/.github/CONTRIBUTING.md):
    This file describes how to contribute to the project. A link to this
    file is available when you open a new issue. The function
    `usethis::use_tidy_contributing()` created the files which I tweaked
    a bit.
2.  [`LICENSE.md`](https://github.com/r-dbi/DBI/blob/master/LICENSE.md):
    The full license terms need to be available as part of the project.
    To create the file, I
    [contributed](https://github.com/r-lib/usethis/pull/472)
    `usethis::use_lgpl_2.1_license()`. Because CRAN discourages
    redistribution of copies of standard license texts in packages, the
    file has been added to `.Rbuildignore`. This makes the file
    available in the GitHub repository, but not package file on CRAN.
3.  [`README.md`](https://github.com/r-dbi/DBI#readme): Added missing
    installation instructions.

Though not on the CII badge checklist, I also added:

1.  [`ISSUE_TEMPLATE.md`](https://github.com/r-dbi/DBI/blob/master/.github/ISSUE_TEMPLATE.md):
    pre-populates the issue description when opening a new issue. This
    is a tweaked version of the file provided by
    `usethis::use_tidy_issue_template()`.

2.  [`CODE_OF_CONDUCT.md`](https://github.com/r-dbi/DBI/blob/master/.github/CODE_OF_CONDUCT.md):
    The default file as added by `usethis::use_code_of_conduct()`.

One badge still had an HTTP image source, after changing it to HTTPS the
criterion that the website needs to use TLS was satisfied.

Establishing a process for reporting code vulnerabilities was perhaps
the most challenging part. It seems unclear if it applies to R packages
at all, in particular to an interface-only package such as DBI. The
solution was to add a link with text to the project page
<a href="https://dbi.r-dbi.org" class="uri">https://dbi.r-dbi.org</a>,
asking to send an e-mail and await further instructions.

Future development
------------------

The principal roadmap for future development has been outlined in the
project proposal. There are both "hard" and "soft" issues to solve,
repeated below, with comments based on experience from the past year.

### "Hard" issues

1.  The test suite for the DBI specificaton in `DBItest` is currently
    designed to run as part of the package checks. The next step is to
    support running the test suite against a particular R + DBMS
    installation, to ensure that code interoperating with that DBMS in
    that environment runs as expected.

    -   Shouldn't be too hard, but need to keep the second "soft" issue
        in mind.

2.  Users expect the hard disk or the DBMS to be the limiting factor for
    loading data, but DBI still lacks a consistent interface for fast
    data import.

    -   The new [*arkdb*
        package](https://cran.r-project.org/package=arkdb) offers a
        dedicated interface for importing data, I still think this
        functionality should better live there (or elsewhere).

3.  The syntax for query placeholders currently depends on the DBMS. A
    consistent interface would be useful, in particular for implementers
    of packages that compute on the database.

    -   This has already caused some confusion. Shouldn't be too hard
        either, but requires a compatibility mode so that existing code
        doesn't break.

4.  The `RPostgres` package now has special handling for geometry data.
    A generic extension to arbitrary data types via hooks would allow
    e.g. returning JSON data directly as a `"json"` class without
    user-initiated manual conversion.

    -   This seems to be a bigger problem, requiring some thought and
        design.

### "Soft" issues

1.  Some users reported installation problems on specific architectures,
    or connectivity problems with certain databases, or other specific
    issues. Making the new backends accessible for various combinations
    of OS/hardware, software, and configuration, will help the adoption
    of the new packages.

    -   I remember seeing many SSL and timezone issues, as well as
        genuine bugs like the representation of times before 1970 on
        Windows. Expect some progress for the second blog post.

2.  The internal architecture of the DBI specification in `DBItest`
    requires a bit of reworking. Currently, it is difficult to
    understand a test failure without inspecting the source code of
    `DBItest`. It is difficult to locate the source of a failure in the
    specification and in the code. Ideally, each test failure would come
    with a precise link to the part of the specification that is
    violated, and with a simple sequence of DBI method calls that allow
    replicating the failure externally.

    -   That code I wrote 1-2 years ago requires some attention...

3.  The communication related to the projects has been rather terse so
    far. The new website
    <a href="https://r-dbi.org" class="uri">https://r-dbi.org</a> can
    host blog posts highlighting different aspects of DBI, and serve as
    a resource for advice on connecting R with databases and computing
    on the database. This includes coordination and support for
    developments around DBI like *sqlr*, an interface for data
    definition statements on top of DBI.

    -   Together with the new [Databases CRAN Task
        View](https://github.com/terrytangyuan/ctv-databases) maintained
        by Yuan (Terry) Tang and
        <a href="https://db.rstudio.com/" class="uri">https://db.rstudio.com/</a>,
        the
        <a href="https://r-dbi.org" class="uri">https://r-dbi.org</a>
        should become a viable resource for new and experienced users
        alike. New users should be directed to tutorials and
        introductory material, whereas experienced users should expect
        to find pointers to solve the most common problems. The role of
        each of these websites remains to be shaped, some overlap may be
        desired.

4.  All operations on DBI currently block until a result is available or
    the DBMS has indicated completion. Asynchronous operations allow
    parallel processing of multiple queries or statements, however some
    research is necessary to understand to what extent this can be
    supported realistically in DBI and for the existing backends.

    -   Additional arguments to `dbConnect()`, like the new
        `check_interrupts` argument [contributed to *RPostgres* by
        Mateusz Żółtak](https://github.com/r-dbi/RPostgres/pull/193),
        are an option to experiment with asynchronous processing without
        disrupting existing code.

These lists are not comprehensive, new issues may surface over time, or
the importance of issues mentioned above may fade.

Outlook: next blog post
-----------------------

The "Maintaining DBI" project is driven by blog posts. I promised four
blog posts, describing the ongoing maintenance and development. Each
blog post will get a corresponding GitHub project, like the project for
[this](https://github.com/orgs/r-dbi/projects/2) and for the
[next](https://github.com/orgs/r-dbi/projects/3) blog post.

For the next iteration, I plan to improve documentation, do a release
round for all packages, furnish more packages with a CII badge, review
several new packages that build on top of *DBI*, and improve my
responsiveness.

A Walkthrough for first-time DBI users seems to be the highest priority,
perhaps accompanied by an online course. Other documentation
improvements mostly will address
<a href="https://r-dbi.org" class="uri">https://r-dbi.org</a>.

The following is an excerpt of changes in the forthcoming CRAN releases
of the DBI packages:

-   *RSQLite*: window functions!
-   *RMariaDB*: better handling for time zones
-   *DBItest*: minor improvements

New packages worth reviewing include:

-   [*arkdb*](https://cran.r-project.org/package=arkdb): Consistent (and
    fast?) import and export
-   [*flobr*](https://cran.r-project.org/package=flobr): Converting
    files to blobs and back
-   [*sqlr*](https://github.com/nbenn/sqlr): SQLAlchemy-like DSL for
    data definition, work in progress
-   [*dbplot*](https://cran.r-project.org/package=dbplot): Plotting from
    the database
-   and many others.

Adding CII badges for the backend packages and *DBItest* will give a
more consistent appearance of the entire project.

As a New Year's resolution, I pledge to do a better job as package
maintainer for *DBI* and related packages. I reserved a few hours each
Monday to respond to issues raised on GitHub and other channels
([SO](https://stackoverflow.com/), [RStudio
Community](https://community.rstudio.com/), Twitter, and the somewhat
underappreciated [R-SIG-DB mailing
list](https://stat.ethz.ch/mailman/listinfo/r-sig-db)). CI builds on
Travis and AppVeyor also require occasional intervention. The remaining
time will be spent on resolving known problems.

Acknowledgments
---------------

Thanks to all contributors to *DBI* and the other projects in the [r-dbi
organization](https://github.com/r-dbi)!

![DBI
contributors](https://krlmlr.github.io/dbi-slides/2018-09-zurich-meetup/img/contributors.svg)
