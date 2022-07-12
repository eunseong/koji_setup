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
./gen_cert.sh $ADMIN      #commonName=$ADMIN
./gen_cert.sh kojiweb     #OrganizationUnitName=kojiweb
./gen_cert.sh kojihub     #OrganizationUnitName=kojihub
./gen_cert.sh kojid       #commonName=kojid (buildhost filesystem)
./gen_cert.sh kojira      #OrganizationUnitName=kojira
```

## Koji - set up guide
### [PostgreSQL Server]
koji can use postgresql as a DB.

#### Configuration files
* /var/lib/pgsql/data/pg_hba.conf
* /var/lib/pgsql/data/postgresql.conf


Let's install and configure a PostgreSQL server and prepare the database which will be used to hold the koji users
```shell
root@kojidomain.tk$ yum install postgresql-server koji
root@kojidomain.tk$ useradd koji;passwd -d koji

# on EL7
root@kojidomain.tk$ postgresql-setup initdb && systemctl start postgresql

# on EL8
root@kojidomain.tk$ postgresql-setup --initdb --unit postgresql
```
then create the koji user and database within PostgreSQL and import koji schema using the provided file from the `koji` package
```shell
root@kojidomain.tk$ su postgres
postgres@kojidomain.tk$ createuser koji && createdb -O koji koji
root@kojidomain.tk$ su koji
koji@kojidomain.tk$ psql koji koji < /usr/share/doc/koji*/docs/schema.sql
koji@kojidomain.tk$ exit
```

1. Edit `/var/lib/pgsql/data/pg_hba.conf` to have the following contents
```shell
#TYPE   DATABASE  USER  CIDR-ADDRESS  METHOD
# "local" is for Unix domain socket connections only
local     koji    apache              trust
local     koji    koji                trust
local     all     all                 ident
# IPv4 local connections:
host      koji    koji  0.0.0.0/0     trust
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
root@kojidomain.tk$ systemctl enable postgresql --now
root@kojidomain.tk$ systemctl restart postgresql
```

Now we need to add the initial admin user to the DB:
```shell
root@kojidomain.tk$ su - koji;
koji@kojidomain.tk$ psql

psql
Type "help" for help.
koji=> insert into users (name, status, usertype) values ('$ADMIN', 0, 0);
koji=> insert into user_perms (user_id, perm_id, creator_id) values (1, 1, 1);
```


### [koji-hub]
Install the koji-hub package along with mod_ssl
```shell
root@kojidomain.tk$ yum install koji-hub httpd mod_ssl
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
You’ll need to make `/mnt/koji/` web-accessible, either here, on the hub, or on another web server altogether. Add `/mnt/koji` directory configuration
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

# uncomment this to enable authentication via SSL client certificates
<Location /kojihub/ssllogin>
    SSLVerifyClient require
    SSLVerifyDepth  10
    SSLOptions +StdEnvVars
</Location>
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
root@kojidomain.tk$ mkdir -p /mnt/koji/{packages,repos,work,scratch}
root@kojidomain.tk$ chown -R apache:apache /mnt/koji/
root@kojidomain.tk$ systemctl restart httpd
```
The root of the koji build directory(`/mnt/koji`) must be mounted on the builder. A Read-Only NFS mount is the easiest way to handle this.
```shell
root@{kojidomain.tk,kojibuilder.tk}$ dnf install nfs-utils libnfsidmap
root@{kojidomain.tk,kojibuilder.tk}$ firewall-cmd --permanent --add-port=80/tcp &&
> firewall-cmd --permanent --add-port=111/tcp &&
> firewall-cmd --permanent --add-port=443/tcp &&
> firewall-cmd --permanent --add-port=875/tcp &&
> firewall-cmd --permanent --add-port=2049/tcp &&
> firewall-cmd --permanent --add-port=5432/tcp &&
> firewall-cmd --permanent --add-port=8069/tcp &&
> firewall-cmd --permanent --add-port=20048/tcp &&
> firewall-cmd --permanent --add-port=42955/tcp &&
> firewall-cmd --permanent --add-port=46666/tcp &&
> firewall-cmd --permanent --add-port=51660/tcp &&
> firewall-cmd --permanent --add-port=54302/tcp
root@{kojidomain.tk,kojibuilder.tk}$ firewall-cmd --reload && firewall-cmd --list-all

root@kojibuilder.tk$ mount -t nfs $FQDN:/mnt/koji /mnt/koji
```

### [koji-CLI]
The koji cli is the standard client. It can perform most tasks and is essential to the successful use of any koji environment.
The system-wide koji client configuration file is `/etc/koji.conf`.
```shell
[koji]
;configuration for koji cli tool

;url of XMLRPC server
server = http://prolinux-koji-el8.tk/kojihub

;url of web interface
weburl = http://prolinux-koji-el8.tk/koji

;url of package download site
topurl = http://prolinux-koji-el8.tk/kojifiles

;path to the koji top directory
topdir = /mnt/koji

; use the fast upload feature of koji by default
use_fast_upload = yes

;client certificate
cert = ~/.koji/client.crt

;certificate of the CA that issued the client certificate
ca = ~/.koji/clientca.crt

;certificate of the CA that issued the HTTP server certificate
serverca = ~/.koji/serverca.crt

;enabled plugins for CLI, runroot is enabled by deafult for fedora
;save_failed_tree is available
plugins = runroot
```
And the user-specific one is in ~/.koji/config.
```shell
root@kojidomain.tk$ useradd $ADMIN && su $ADMIN
ADMIN@kojidomain.tk$ mkdir ~/.koji
ADMIN@kojidomain.tk$ cp /etc/pki/koji/${ADMIN}.pem ~/.koji/client.crt
ADMIN@kojidomain.tk$ cp /etc/pki/koji/koji_ca_cert.crt ~/.koji/clientca.crt
ADMIN@kojidomain.tk$ cp /etc/pki/koji/koji_ca_cert.crt ~/.koji/serverca.crt
ADMIN@kojidomain.tk$ ln -s /etc/koji.conf ~/.koji/config
```
The following command will test your login to the hub:
```shell
ADMIN@kojidomain.tk$ koji hello
dobre dan, ADMIN!

You are using the hub at https://kojidomain.tk/kojihub
Authenticated via client certificate /home/ADMIN/.koji/client.crt
```

### [koji-web]
Install the koji-web package along with mod_ssl:
```shell
root@kojidomain$ yum install koji-web mod_ssl
```
You will need to edit the kojiweb configuration file `/etc/kojiweb/web.conf` to tell kojiweb which URLs it should use to access the hub, the koji packages and its own web interface. You will also need to tell kojiweb where it can find the SSL certificates for each of these components. If you are using SSL authentication, the “WebCert” line below must contain both the public and private key.
```shell
[web]
SiteName = EL8-ProLinux-koji
# KojiTheme = mytheme

# Key urls
KojiHubURL = http://kojidomain.tk/kojihub
KojiFilesURL = http://kojidomain.tk/kojifiles
kojiWebURL = http://kojidomain.tk/koji

# Kerberos authentication options
# WebPrincipal = koji/web@EXAMPLE.COM
# WebKeytab = /etc/httpd.keytab
# WebCCache = /var/tmp/kojiweb.ccache
# The service name of the principal being used by the hub
# KrbService = host

# SSL authentication options
WebCert = /etc/pki/koji/kojiweb.pem
ClientCA = /etc/pki/koji/koji_ca_cert.crt
KojiHubCA = /etc/pki/koji/koji_ca_cert.crt

LoginTimeout = 72

# This must be changed and uncommented before deployment
Secret = superS3cret#

LibPath = /usr/share/koji-web/lib
...
```
#### Making Koji-Web look nice (optional)
```
cat /etc/httpd/conf.d/kojitheme.conf
#Override a few static items with our theme
Alias /koji-static/images/koji.png "/usr/share/koji-themes/navercloud-koji/images/navercloud-koji.png"
Alias /koji-static/images/powered-by-koji.png "/usr/share/koji-themes/navercloud-koji/images/Powered-by-koji_button.png"
Alias /koji-static/images/koji.ico "/usr/share/koji-themes/navercloud-koji/images/navercloud-koji.ico"
Alias /koji-static/koji.css "/usr/share/koji-themes/navercloud-koji/koji.css"
Alias /koji-static/errors/unauthorized.html "/usr/share/koji-themes/navercloud-koji/errors/unauthorized.html"
Alias /koji-static/images/header-bg.png "/usr/share/koji-themes/navercloud-koji/images/header-bg.png"
Alias /koji-static/images/complete.png "/usr/share/koji-themes/navercloud-koji/images/tick.png"
Alias /koji-static/images/yes.png "/usr/share/koji-themes/navercloud-koji/images/tick.png"
Alias /koji-static/images/closed.png "/usr/share/koji-themes/navercloud-koji/images/tick.png"
Alias /koji-static/images/ready.png "/usr/share/koji-themes/navercloud-koji/images/tick.png"
Alias /koji-static/images/no.png "/usr/share/koji-themes/navercloud-koji/images/minus.png"
Alias /koji-static/images/failed.png "/usr/share/koji-themes/navercloud-koji/images/minus.png"
Alias /koji-static/images/building.png "/usr/share/koji-themes/navercloud-koji/images/spanner.png"
Alias /koji-static/images/deleted.png "/usr/share/koji-themes/navercloud-koji/images/rubbishbin.png"
Alias /koji-static/images/free.png "/usr/share/koji-themes/navercloud-koji/images/clock.png"
Alias /koji-static/images/waiting.png "/usr/share/koji-themes/navercloud-koji/images/clock.png"
Alias /koji-static/images/open.png "/usr/share/koji-themes/navercloud-koji/images/arrowcircle.png"

<Directory "/usr/share/koji-themes/navercloud-koji">
    AllowOverride None
    # Allow open access:
    Order allow,deny
    Allow from all
    Require all grant
```

And then

```
cd /usr/local/src
git clone http://fedorapeople.org/cgit/ausil/public_git/koji-theme-fedora.git
cp ./koji-theme-fedora/www/static/images/fedora-koji.png ./koji-theme-fedora/www/static/images/navercloud-koji.png 
cp ./koji-theme-fedora/www/static/images/fedora-koji.ico ./koji-theme-fedora/www/static/images/navercloud-koji.ico
mkdir -p /usr/share/koji-themes/navercloud-koji
cp -r ./koji-theme-fedora/www/static /usr/share/koji-themes/navercloud-koji
systemctl restart httpd
````

### [koji builder]
The kojid service uses mock for creating pristine build environments and creates a fresh one for every build, ensuring that artifacts of build processes cannot contaminate each other. All of kojid is written in Python and communicates with koji-hub via XML-RPC.

Install the koji-builder package
```shell
root@kojibuilder.tk$ yum install koji-builder
```
The configuration file `/etc/kojid/kojid.conf` for each koji builder must be edited so that the line below points to the URL for the koji hub. The user tag must also be edited to point to the username used to add the koji builder.
```shell
[kojid]
...
topdir=/mnt/koji
...
user=#buildhost domaion
...
workdir=/tmp/koji
...
; The vendor to use in rpm headers
vendor=Tmax A&C Co., Ltd.
...
; The packager to use in rpm headers
packager=Tmax A&C Co., Ltd. <https://technet.tmaxsoft.com/>
...
; The distribution to use in rpm headers
distribution=TmaxA&C
...
; The _host string to use in mock
mockhost=redhat-linux-gnu
...
; The URL for the xmlrpc server
server=http://kojidomain.tk/kojihub
...
;client certificate
cert = /etc/pki/koji/kojid.pem

;certificate of the CA that issued the HTTP server certificate
serverca = /etc/pki/koji/koji_ca_cert.crt
ca = /etc/pki/koji/koji_ca_cert.crt
...
```
#### Add the host entry for the koji builder to the database
You will now need to add the koji builder to the database so that they can be utilized by koji hub. Make sure you do this before you start kojid for the first time, or you’ll need to manually remove entries from the sessions and users table before it can be run successfully.
```shell
ADMIN@kojidomain.tk$ koji add-host buildhost.tk x86_64
ADMIN@kojidomain.tk$ koji add-host-to-channel buildhost.tk createrepo
root@kojibuildhost.tk$ systemctl restart kojid
```
If its shown as "not ready", then check your logs in /var/logs/kojid.log

### [kojira]
Install the koji-utils package
```shell
root@kojibuilder.tk$ yum install koji-utils
```

Set `/etc/kojira/kojira.conf`
```shell
...
; The URL for the koji hub server
server=http://kojidomain.tk/kojihub

; The directory containing the repos/ directory
topdir=/mnt/koji
...
;client certificate
cert = /etc/pki/koji/kojira.pem

;certificate of the CA that issued the client certificate
ca = /etc/pki/koji/koji_ca_cert.crt
...
```

The kojira user requires the repo permission to function.
```shell
ADMIN@kojidomain.tk$ koji add-user kojira
ADMIN@kojidomain.tk$ koji grant-permission repo kojira
```

## Koji - Cluster Migration
### DB migration
```
# Export DB
[root@src_hostname ~]# pg_dump --dbname=koji --host=127.0.0.1 --port=5432 --username=koji --password --format=p --file=./koji_DB.sql
# Import DB
[koji@target_hostname]$ psql "postgresql://koji:koji@127.0.0.1:5432/koji" -f ./kojiDB.sql
```
