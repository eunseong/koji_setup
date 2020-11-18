# koji Build System
This project provides a simple guide for setting up the Koji cluster. It's tested on EL(Enterprise Linux)7 and EL8<br/>
Koji build system is the software that builds RPM packages for the Fedora project. It uses Mock to create chroot environments to perform builds.

## Terminology
In Koji, it is sometimes necessary to distinguish between the a package in general, a specific build of a package, and the various rpm files created by a build. When precision is needed, these terms should be interpreted as follows:

* Package
  * The name of a source rpm. This refers to the package in general and not any particular build or subpackage. For example: kernel, glibc, etc.
* Build
  * A particular build of a package. This refers to the entire build: all arches and subpackages. For example: kernel-2.6.9-34.EL, glibc-2.3.4-2.19.
* RPM
  * A particular rpm. A specific arch and subpackage of a build. For example: kernel-2.6.9-34.EL.x86_64, kernel-devel-2.6.9-34.EL.s390, glibc-2.3.4-2.19.i686, glibc-common-2.3.4-2.19.ia64

## koji components
Koji is comprised of several components:
* koji-hub
  * koji-hub is the center of all Koji operations. It is an XML-RPC server running under mod_wsgi in Apache. koji-hub is passive in that it only receives XML-RPC calls and relies upon the build daemons and other components to initiate communication. koji-hub is the only component that has direct access to the database and is one of the two components that have write access to the file system.
* koji (CLI)
  * koji CLI written in Python that provides many hooks into Koji. It allows the user to query much of the data as well as perform actions such as build initiation.
* koji-web
  * koji-web is a set of scripts that run in mod_wsgi and use the Cheetah templating engine to provide an web interface to Koji. koji-web exposes a lot of information and also provides a means for certain operations, such as cancelling builds.
* kojid (builder)
  * kojid is the build daemon that runs on each of the build machines. Its primary responsibility is polling for incoming build requests and handling them accordingly. Koji also has support for tasks other than building. Creating install images is one example. kojid is responsible for handling these tasks as well.
* kojira
  * kojira is a daemon that keeps the build root repodata updated.
## Koji - set up guide
### koji-hub


#### DB setup
koji can use postgresql as a DB. Let's install and configure a PostgreSQL server and prime the database which will be used to hold the koji users
```shell
root@localhost$ yum install postgresql-server koji
root@localhost$ postgresql-setup initdb && systemctl start postgresql

# on EL7
root@localhost$ useradd koji;passwd -d koji

# on EL8
root@localhost$ postgresql-setup --initdb --unit postgresql
```
then create the koji user and database within PostgreSQL and import koji schema using the provided file from the `koji` package
```shell
root@localhost$ su postgres
postgres@localhost$ createuser koji; createdb -O koji koji
postgres@localhost$ su koji
koji@localhost$ psql koji koji < /usr/share/doc/koji*/docs/schema.sql
```

1. Edit `/var/lib/pgsql/data/pg_hba.conf` to have the following contents
```shell
#TYPE   DATABASE  USER  CIDR-ADDRESS  METHOD
# "local" is for Unix domain socket connections only
local     koji    apache              trust
local     koji    koji                trust
local     all     all                 ident
# IPv4 local connections:
host      koji    koji  127.0.0.1/32  trust
host      all     all   127.0.0.1/32  ident
# IPv6 local connections:
host      all     all   ::1/128       ident
```
* `TYPE` - The local connection type means the postgres connection uses a local Unix socket, so PostgreSQL is not exposed over TCP/IP at all.
* `DATABASE` - The local koji user should only have access to the koji database. The local postgres user will have access to everything.

2. Edit `/var/lib/pgsql/data/postgresql.conf` and set `listen_addresses` to prevent TCP/IP access entirely
```shell
# - Connection Settings - 

listen_addresses = '*'
...
#port = 5432
```

Last, restart the database service
```shell
root@localhost$ systemctl enable postgresql --now
root@localhost$ systemctl restart postgresql
```

## koji CLI
## koji-web
## koji builder
## kojira
