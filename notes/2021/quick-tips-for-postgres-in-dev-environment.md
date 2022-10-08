---
[meta]
title = "Quick tips for postgres in dev environment"
date = 2021-12-03
tags = ["postgres", "ubuntu", "linux", "relational database", "psql", "postgresql", "dev"]
description = ""
---

While developing tools that use PostgresSQL as a database, I often tend to 
forget about syntax and some commands for basic operations. This post is a note 
to myself, but feel free to find your answer here.


***1) Install postgres in Ubuntu***

```commandline 
sudo apt install postgres postgresql-client
```

***2) Install postgres in Mac***

    Navigate to https://postgresapp.com/

***3) Connect to local postgres from Ubuntu***

```commandline 
sudo su - postgres
psql
```

***4) Setup a new dev database***

```commandline 
CREATE DATABASE <database name>
CREATE USER <user> WITH ENCRYPTED PASSWORD '<password>';
GRANT ALL PRIVILEGES ON DATABASE <database name> TO <user>;
```

***5) Backup a databasse from ubuntu***

```commandline
pg_dump -U username database -f dump.sql
```

***6) Restore a database from ubuntu***

```commandline
psql -U username -d database -f dump.sql
```

***7) Restore a database in Mac*** (_From psql shell opened by Postgress.app._)

```commandline
\i <path_to_.dump_file>
```
