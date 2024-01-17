---
title: "Large file listing diffs with postgres"
tags:
  - postgres
  - docker
---

I recently needed to know what files were missing from a backup (~2 million files). Directory structure aside, I simply needed to know what files were missing from the destination. Once I got a recursive file listing of both source and destination directories, I imported them into unique postgres tables and did a `LEFT OUTER JOIN` query to see what was present in the source but absent in the destination.

## docker postgres

Spin up postgres in Docker leveraging env_file with credentials and an initial database named `sample_db`.

```plaintext
# postgres.env
POSTGRES_USER="post_user"
POSTGRES_PASSWORD="post_pass"
POSTGRES_DB="sample_db" # default Postgres database
```

```yaml
# docker-compose.yaml
version: '3'

services:
  postgres:
    container_name: postgres
    image: postgres:14.1
    restart: always
    ports:
      - 5432:5432
    env_file:
      - ./postgres.env # contains POSTGRES_USER, POSTGRES_PASSWORD, and POSTGRES_DB
    volumes:
      - ./postgresql_data:/var/lib/postgresql/data
```

`docker-compose up -d`

### connect to postgres

Export the following environment variables for `psql`.

```plaintext
# .envrc
export PGHOSTADDR=127.0.0.1
export PGDATABASE=sample_db
export PGUSER=post_user
export PGPASSWORD=post_pass
```

Connect with `psql`. You'll be dropped into the default Postgres database.

```shell
➜ psql
psql (16.1, server 14.1 (Debian 14.1-1.pgdg110+1))
Type "help" for help.

sample_db=#
```

## create tables

From the `sample_db=#` prompt create two tables with rows named `file_name`.

```sql
CREATE TABLE destination_files (
  file_name TEXT
);
CREATE TABLE source_files (
  file_name TEXT
);
```

List the tables.

```plaintext
\dt
               List of relations
 Schema |       Name        | Type  |   Owner
--------+-------------------+-------+-----------
 public | destination_files | table | post_user
 public | source_files      | table | post_user
(2 rows)
```

## import data

### sample data snippets

Below are snippets from `destination_files.txt` and `source_files.txt`.

```plaintext
# source_files.txt
36719.zip
36720.zip
36721.zip
36722.zip
36723.zip
36724.zip
36725.zip
36726.zip
```

```plaintext
# destination_files.txt
893145.zip
893146.zip
893147.zip
893148.zip
893149.zip
893150.zip
893151.zip
893152.zip
```

### use `psql` to copy the data

Exit from the `psql` `sample_db=#` prompt. Use `psql -c` to copy the data into their respective tables.

```shell
➜ psql -c "\COPY source_files (file_name) FROM './source_files.txt'"
COPY 2131354
```

```shell
➜ psql -c "\COPY destination_files (file_name) FROM './destination_files.txt'"
COPY 2131350
```

## diff the data

Connect with `psql` again. From the `sample_db=#` prompt run the following query.

```sql
SELECT *
FROM source_files
LEFT OUTER JOIN destination_files ON source_files.file_name = destination_files.file_name
WHERE destination_files.file_name IS NULL;
```

From the output we can see the four files present in the source but absent in the destination.

```plaintext
 file_name | file_name
-----------+-----------
 1344167.zip |
 1310270.zip |
 1420005.zip |
 2033203.zip |
(4 rows)
```
