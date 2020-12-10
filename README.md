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

## Koji components
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

## Prerequisites
Koji primarily supports Kerberos and SSL Certificate authentication.
We will be using SSL certificates for the xmlrpc server, for the various koji components, and one for the admin user will need to be setup. Let's get started setting up the Koji Root CA with the basic guide below.
https://github.com/eunseong/koji-ssl-certificate

After setting the guide above, proceed to the next
```shell
cd /etc/pki/koji/
./gen_cert.sh kojiadmin   #FQDN=kojiadmin
./gen_cert.sh kojiweb     #OUN=kojiweb
./gen_cert.sh kojihub     #OUN=kojihub
```

## Koji - set up guide
### [PostgreSQL Server]
koji can use postgresql as a DB.

#### Configuration files
* /var/lib/pgsql/data/pg_hba.conf
* /var/lib/pgsql/data/postgresql.conf


Let's install and configure a PostgreSQL server and prepare the database which will be used to hold the koji users
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

and then restart the database service
```shell
root@localhost$ systemctl enable postgresql --now
root@localhost$ systemctl restart postgresql
```

Now we need to add the initial admin user to the DB:
```shell
root@localhost$ su - koji;
koji@localhost$ psql

psql
Type "help" for help.
koji=> insert into users (name, status, usertype) values ('$ADMIN', 0, 0);
koji=> insert into user_perms (user_id, perm_id, creator_id) values (1, 1, 1);
```


### [koji-hub]
Install the koji-hub package along with mod_ssl
```shell
root@localhost$ yum install koji-hub httpd mod_ssl
```

#### Configuration files
* /etc/koji-hub/hub.conf
* /etc/koji-hub/hub.conf.d/*
* /etc/httpd/conf/httpd.conf
* /etc/httpd/conf.d/kojihub.conf
* /etc/httpd/conf.d/ssl.conf


#### /etc/koji-hub/hub.conf
This file contains the configuration information for the hub. You will need to edit this configuration to point Koji Hub to the database you are using and to setup Koji Hub to utilize the authentication scheme you selected in the beginning.
```shell
## Basic options ##
DBName = koji
DBUser = koji
DBHost = #FQDN=prolinux-koji.tk
#DBPass = example_password
KojiDir = /mnt/koji
......
DNUsernameComponent = CN
ProxyDNs = /C=KO/ST=Gyeonggi/O=TmaxA&C/OU=kojiweb/CN=#FQDN
......
LoginCreatesUser = On
KojiWebURL = http://#FQDN/koji
```
#### /etc/httpd/conf/httpd.conf
You’ll need to make /mnt/koji/ web-accessible, either here, on the hub, or on another web server altogether. Add "/mnt/koji" directory configuration
```shell
#
# Relax access to content within /var/www.
#
<Directory "/var/www">
    AllowOverride None
    # Allow open access:
    Require all granted
</Directory>
...
<Directory "/mnt/koji">
    AllowOverride None
    # Allow open access:
    Order allow,deny
    Allow from all
    Require all granted
</Directory>
```

#### /etc/httpd/conf.d/kojihub.conf
Add below configuration
```shell
<Directory "/usr/share/koji-hub">
    Options ExecCGI
    SetHandler wsgi-script
    WSGIApplicationGroup %{GLOBAL}
    <IfVersion < 2.4>
        Order allow,deny
        Allow from all
    </IfVersion>
    <IfVersion >= 2.4>
        Require all granted
    </IfVersion>
</Directory>

# Also serve /mnt/koji
Alias /kojifiles "/mnt/koji/"

<Directory "/mnt/koji">
    Options Indexes SymLinksIfOwnerMatch
    AllowOverride None
    Require all granted
</Directory>
```

#### /etc/httpd/conf.d/ssl.conf
The CA certificate you configure in SSLCACertificateFile here should be the same one that you use to issue user certificates.
```shell
SSLCertificateFile /etc/pki/koji/certs/kojihub.crt
SSLCertificateKeyFile /etc/pki/koji/certs/kojihub.key
SSLCertificateChainFile /etc/pki/koji/koji_ca_cert.crt
SSLCACertificateFile /etc/pki/koji/koji_ca_cert.crt
SSLVerifyClient require
SSLVerifyDepth 10
```
#### Preparing the koji filesystem skeleton
We need to create a skeleton filesystem structure for koji as well as make the file area owned by apache so that the xmlrpc interface can write to it as needed.
```shell
mkdir -p /mnt/koji/{packages,repos,work,scratch}
chown -R apache:apache /mnt/koji/
systemctl restart httpd
```
The root of the koji build directory(/mnt/koji) must be mounted on the builder. A Read-Only NFS mount is the easiest way to handle this.
```shell
dnf install nfs-utils libnfsidmap
firewall-cmd --permanent --add-port=80/tcp
firewall-cmd --permanent --add-port=111/tcp
firewall-cmd --permanent --add-port=443/tcp
firewall-cmd --permanent --add-port=875/tcp
firewall-cmd --permanent --add-port=2049/tcp
firewall-cmd --permanent --add-port=5432/tcp
firewall-cmd --permanent --add-port=8069/tcp
firewall-cmd --permanent --add-port=20048/tcp
firewall-cmd --permanent --add-port=42955/tcp
firewall-cmd --permanent --add-port=46666/tcp
firewall-cmd --permanent --add-port=51660/tcp
firewall-cmd --permanent --add-port=54302/tcp
firewall-cmd --reload && firewall-cmd --list-all

mount -t nfs $FQDN:/mnt/koji /mnt/koji
```


## koji CLI
## koji-web
## koji builder
## kojira
