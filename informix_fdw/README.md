README
======

Exploration of PostgreSQL Foreign Data Wrapper with an Informix databases.

Using <https://github.com/credativ/informix_fdw>

# PostgreSQL

PostgreSQL 15 was used for testing. This version worked even though the last stated
supported version of `informix_fdw` is PostgreSQL 13.

# IBM Client SDK Setup

## Compatibility

Works on CentOS 7 not on Rocky 8 (RHEL8) due to version of ncurses library needed.
Needs libncurses.5 provides EPL libncurses.6.

> what doesnt work
> i think it is the setup of the Informix Client SDK

> **NOTE**: An IBM account is needed to download the Client SDK.

## Installation

Client SDK version used

```
Informix Client SDK Developer Edition for Linux x86_64, 64-bit

clientsdk.4.10.FC12.LINUX.tar
```

* Download and run `sudo ./installclientsdk` .
* Select options 1-10
* Installation path: `/opt/IBM/Informix_Client-SDK`

# Informix Foreign Data Wrapper setup

* Clone `https://github.com/credativ/informix_fdw.git`

* Install the following packages
  * yum install libpq5{,-devel}
  * centos-release-scl
  * llvm-toolset-7
* command

* Modify `Makefile` with the path to `pg_config` which is specific to the PostgreSQL 15
  setup

  from
  ```
  PG_CONFIG=pg_config
  ```
  to
  ```
  PG_CONFIG=/usr/pgsql-15/bin/pg_config
  ```

This modification will ensure the libraries are installed in the appropriate locations
for PostgreSQL 15.

* Modify the `Makefile`


to move on `Makefile`  was edited

from

```
ESQL=esql
```

to

```
ESQL=${INFORMIXDIR}/bin/esql
```

`INFORMIXDIR` needs to be set to the location of the Client SDK.

This modification will ensure the error

```
Preprocessing Informix ESQL/C sources
## Only preprocessing, compilation will be performed later
esql -c ifx_connection.ec
make: esql: Command not found
make: *** [ifx_connection.c] Error 127
```

does not occur.

Run the installation

```
  # INFORMIXDIR=/opt/IBM/Informix_Client-SDK/ USE_PGXS=1 make install
```


### Post installation

There are Informix libraries that need to be properly located so that they can be found
by `ifx_fdw.so`.

Check the dependencies of the library

```
  # ldd /usr/pgsql-15/lib/ifx_fdw.so

    linux-vdso.so.1 =>
    libifsql.so => not found
    libifasf.so => not found
    libifgen.so => not found
    libifos.so => not found
    libifgls.so =>  not found
    libpthread.so.0 => /lib64/libpthread.so.0
    libm.so.6 => /lib64/libm.so.6
    libdl.so.2 => /lib64/libdl.so.2
    libcrypt.so.1 => /lib64/libcrypt.so.1
    libifglx.so => not found
    libc.so.6 => /lib64/libc.so.6
    /lib64/ld-linux-x86-64.so.2
    libfreebl3.so => /lib64/libfreebl3.so
```

Configure the missing libraries by creating symlinks into the `/usr/pgsql-15/lib` folder from the respective IBM Client SDK folders

For example

```
  # ln -s /opt/IBM/Informix_Client-SDK/lib/libifsql.so /usr/pgsql-15/lib
```

Running ldd should show that the library is found

```
  # ldd /usr/pgsql-15/lib/ifx_fdw.so
    linux-vdso.so.1 =>
    libifsql.so => /usr/pgsql-15/lib/libifsql.so
    ... truncated ...
```

After all the libraries are linked and found  you can create the extension in PostgreSQL

```
  psql> create extension informix_fdw;
```

# Creation of Foreign Database and Table

## Informix Database

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
