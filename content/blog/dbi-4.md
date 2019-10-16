+++
author = "Kirill Müller"
date = "2019-10-16"
draft = true
weight = 180
title = "Maintaining DBI, 2/4"
description = "Summarizing the progress of 2019"
+++

What is DBI?
------------------------

DBI is a package for connecting to database management systems (DBMS).
The goal is a common interface no matter the specific underlying DBMS.
Hence the name: DBI stands for **d**ata**b**ase **i**nterface.

DBI works with a variety of DBMS, like Postgres, MariaDB, Sqlite... .
Therefore, users can focus on the specifics of their project instead of setting up the infrastructure for data import and export.

What happend last year
------------------------

Over the last year, I was working on several bigger tasks...

Milestones
------------------------

Input from Kirill

Implement Logging
------------------------

In spring, I implemented logging for DBI and published it as a package called [{DBIlog}](https://github.com/krlmlr/DBIlog).

Especially when using applications in production, keeping logs is an invaluable part of a sound infrastructure.
DBIlog helps with troubeshooting and auditing queries that access a database.
The goal of DBIlog is to implement logging for arbitrary DBI backends, similarly to Perl’s DBI::Log.

Also, DBIlog can be used with packages like {dplyr} and {dbplyr}.

Oftentimes, DBI is used under the hood by other packages like [dplyr](), [dbplyr]() or [dm](). For example, functions like `dplyr::src_dbi()` or `dbplyr::src_dbi()` are working with underlying DBI operations. DBIlog works in these scenarios, too. 

DBIlog is designed to be as simple as possible.
All that needs to be done is initializing a logging driver with `LoggingDBI()` before connecting to a database management system.
After that, all calls to DBI methods are logged and by default printed to the console, of course it can be redirected to an file as well.

Keep in mind that the logging output is runnable R code.
So you can just copy, paste and execute it, when debugging.

## This and that from Done-column
------------------------

Input from Kirill
