README
======

Exploration of PostgreSQL Foreign Data Wrapper with an Informix databases.

Using <https://github.com/credativ/informix_fdw>

# PostgreSQL

PostgreSQL 16 used for testing. This version worked even though the last stated
supported version of `informix_fdw` is PostgreSQL 13.

# IBM Client SDK Setup

## Prerequisites

* Java Runtime

```shell
# sudo dnf install java-21-openjdk
```

## Compatibility

SDK: IBM Informix Client-SDK 4.50.FC12W5 Linux x86 64 bit (ibm.csdk.4.50.12.5.Linux.64.x86_64.tar)

[Workaround for Rocky 9][rocky9] based on Informix version

```
[The] following system libraries are needed for IBM Informix version 14.10.FC9,
14.10.FC10, and 14.10.FC11 to run on Red Hat Linux 9. (This workaround is not
needed in 14.10.FC11W1 and later). Higher versions of same libraries are
installed on Red Hat Linux 9. As a workaround, symbolic links to those
libraries need to be created.

      Libraries needed for Informix:
          libncurses.so.5
          libtinfo.so.5

      Libraries installed on Red Hat Linux 9:
         libncurses.so.6
         libtinfo.so.6

      Workaround:
          Run following commands as root user from /usr/lib64 directory.
               ln -s libncurses.so.6 libncurses.so.5

               ln -s libtinfo.so.6 libtinfo.so.5
```

**NOTE**: An IBM account is needed to download the Client SDK.

## Installation

Informix Client SDK Developer Edition for Linux x86_64, 64-bit
Client SDK version used

Link: https://www.ibm.com/resources/mrs/assets/packageList?source=ifxdl&lang=en_US

* Download and
* Run `sudo ./installclientsdk` -i console
* Select features 1,2,4,6,7,8,10,12

* Installation path: `/opt/IBM/Informix.4.50.FC12W5`

# Informix Foreign Data Wrapper Setup

## Dependencies

Postgresql dependencies

```shell

$ sudo dnf install postgresql-server-devel

```

## Setup

* Clone `https://github.com/credativ/informix_fdw.git`

```
$ cd inforix_fdw
$ export PATH="/opt/IBM/Informix.4.50.FC12W5/bin:$PATH"
$ sudo -E INFORMIXDIR=/opt/IBM/Informix.4.50.FC12W5 USE_PGXS=1 make install
```

Library should be installed at `/usr/lib64/pgsql/ifx_fdw.so`.

## Configure  library files

Create file `/etc/ld.so.conf.d/informix.conf` with content

```conf
/opt/IBM/Informix.4.50.FC12W5/lib/
/opt/IBM/Informix.4.50.FC12W5/lib/esql/
```

Run

```shell
# ldconfig
```

Verify all libraries are found and `not found` is not returend for any libraries.

```shell
# ldd /usr/lib65/pgsql/ifx_fdw.so
```

# Using Foreign Data Wrapper

After verifying libraries are linked verify you can create the extension in PostgreSQL

```
  # su - postgres
  #  psql
  psql> create extension informix_fdw;
  CREATE EXTENSION
```

# Creation of Foreign Database and Table

## Informix Database

### Docker Container Setup


### Database Setup

The test database is called `sales_demo` with the table `customer`.


```
  # dbschema -d sales_demo
DBSCHEMA Schema Utility       INFORMIX-SQL Version 14.10.FC7W1
grant dba to "informix";

{ TABLE "informix".customer row size = 55 number of columns = 3 index size = 0 }

create table "informix".customer
(
customer_code integer,
customer_name char(31),
company_name char(20)
);

revoke all on "informix".customer from "public" as "informix";

grant select on "informix".customer to "public" as "informix";
grant update on "informix".customer to "public" as "informix";
grant insert on "informix".customer to "public" as "informix";
grant delete on "informix".customer to "public" as "informix";
grant index on "informix".customer to "public" as "informix";

revoke usage on language SPL from public ;
grant usage on language SPL to public ;
```

The database connection configuration `etc/sqlhosts`

> this file was referenced from the Clent SDK folder but would work similarly for
a regular server setup

```conf
# dbservername  nettype         hostname        servicename     options

informix        onsoctcp        ifx     9088
informix_dr     drsoctcp        ifx     9089
```

In this example there is an enty in `/etc/hosts` which resolves `ifx`.

## Postgres Setup

The following shows the setup of the foreign objects in PostgreSQL.

```
 postgres=# CREATE EXTENSION informix_fdw;

 postgres=# CREATE DATABASE sales_demo_fdw;
 postgres=# \c sales_demo_fdw;

 sales_demo_fdw=# CREATE SERVER informix_tcp
 > FOREIGN DATA WRAPPER informix_fdw
 > OPTIONS (
 > informixserver 'informix',
 > informixdir '/opt/IBM/Informix_Client-SDK/',
 > client_locale 'en_US.819',
 > database 'sales_demo'
 > );

 sales_demo_fdw=# CREATE USER MAPPING FOR CURRENT_USER
 > SERVER informix_tcp
 > OPTIONS (username '<username>', password '<password>');

 sales_demo_fdw=# create foreign table foo_customer (
 > customer_code integer,
 > customer_name varchar(31),
 > company_name varchar(20)
 > )
 > server informix
 > options (query 'SELECT * FROM customer',
 > database 'sales_demo');
```

The instructions in https://github.com/credativ/informix_fdw has a setting for
`informixdir` on the `CREATE FOREIGN TABLE` statement. However, this will throw an error
showing that `informixdir` is not a valid option in this context.


**Note**:   The `client_locale`  are properly set

The client_locale must match the `db_locale`. To find the `db_locale` use the following
query against the Informix database server

```sql
  > database SysMaster;
  > select * from systables where tabid=91;
  tabname           GL_CTYPE
  owner
  ... truncated ...
  flags            0
  site             en_US.819
  dbname
  ... truncated ...

```

## Execute Query

After all the configurations are properly done query the foreign table

```sql
  sales_demo_fdw=# SELECT * FROM foo_customer;
  WARNING:  opened informix connection with warnings
  DETAIL:  informix SQLSTATE 01I04: "Database selected "
  customer_code  |          customer_name          |     company_name
  ---------------+---------------------------------+----------------------
  10             | Carole Sadler                   | Sports Spot
  103            | John Doe                        | John Company
  (2 rows)
```

## Troubleshooting

* SQLCODE=-25596
  * Check `sqlhosts` and the name used in `CREATE SERVER` configuration to ensure the
  server name is consistent.

* SQLCODE=-23197
  * Check the `client_locale` and the locale on the database (instructions above)
  * To update the configuration
  ```sql
    ALTER SERVER  informix_tcp OPTIONS (set client_locale 'en_US.819');
  ```
[rocky9]: https://www.ibm.com/support/pages/informix-client-software-development-kit-client-sdk-and-informix-connect-system-requirements#linux
