# osm
Steps to install Open Street Maps on Centos 7 (with aim to install on Red Hat 7 also)
Summarised & updated from: https://knowledgebase.hyperlearning.ai/en/articles/centos-7-open-street-map-tile-server

## Summary

We require the following software: 
 * [Mapnik](https://github.com/mapnik/mapnik) to render tiles, which also requires: 
   * boost
 * Open Street Maps Carto, provides stylesheets for mapping layers in OSM https://github.com/gravitystorm/openstreetmap-carto
 * apache web server - providing the web service to server web pages etc
 * mod_tile - cache of tiles
 * PostgreSQL with PostGIS extensions

Optionally, if we want to edit the stylesheets we could consider installing
 * [Kosmtik](https://github.com/kosmtik)

# Pre-Reqs

## Pre-Downloads

``` bash
cd ~
wget https://downloads.sourceforge.net/boost/boost_1_76_0.tar.bz2
wget https://github.com/mapnik/mapnik/archive/refs/heads/master.zip
```


## Create local user 
Steps not in the linked article
``` bash
sudo su -
	adduser renderaccount
	passwd renderaccount
  usermod -aG wheel renderaccount
```

## CentOS 7   Dependencies
There are some pre-requisite packages that need to be installed on your CentOS 7 server before we can get started installing the dependencies listed above. This guide assumes that you have installed the minimal ISO for CentOS 7 (command-line only). If not, or depending on other packages you may have installed historically, some of the pre-requisitie packages listed below may already be installed on your server.

``` bash
# Dependencies
#Adjust accordingly for red-hat
sudo yum -y install epel-release

sudo yum install libpng libtiff libjpeg freetype gdal cairo pycairo sqlite geos boost curl libcurl libicu bzip2-devel libpng-devel libtiff-devel zlib-devel libjpeg-devel libxml2-devel python-setuptools proj-devel proj proj-epsg proj-nad freetype-devel libicu-devel gdal-devel sqlite-devel libcurl-devel cairo-devel pycairo-devel geos-devel protobuf-devel protobuf-c-devel lua-devel cmake proj boost-thread proj-devel autoconf automake libtool pkgconfig ragel gtk-doc glib2 glib2-devel libpng libpng-devel libwebp libtool-ltdl-devel python-devel harfbuzz harfbuzz-devel harfbuzz-icu boost-devel cabextract xorg-x11-font-utils fontconfig perl-DBD-Pg mesa-libGLU-devel

# GCC++ 14 standards are required for Mapnik so we shall install the Dev Toolset from the CentOS Software Collections
#in red hat would instead be something like:   yum-config-manager --enable rhel-server-rhscl-7-rpms
sudo yum install centos-release-scl
sudo yum install devtoolset-7
scl enable devtoolset-7 bash

# Carto can be installed via NodeJS so we will install NodeJS too
sudo yum install nodejs
npm -v

# Git Version Control is required to clone relevant dependency source code repositories
sudo yum install git
```

# Installation

## Postgres and PostGIS

```
# Install Postgres
sudo rpm -Uvh https://yum.postgresql.org/11/redhat/rhel-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum -y install postgresql11-server postgresql11

#init db and basic setup
sudo /usr/pgsql-11/bin/postgresql-11-setup initdb
cd /var/lib/pgsql/11
cp data/postgresql.conf data/postgresql.conf.orig
vi data/postgresql.conf
    # Add the IP addresses on which the server should listen for connections NOT REQUIRED AT THIS STAGE
    listen_addresses = 'localhost,192.168.1.1'

sudo systemctl enable postgresql-11.service
sudo systemctl start postgresql-11.service

#Setup Postgres User/s: 
sudo su - postgres
  psql -c "alter user postgres with password 'password'"
    ALTER ROLE
exit

# Basic Authentication Methods
cd /var/lib/pgsql/11
cp data/pg_hba.conf data/pg_hba.conf.orig
vi data/pg_hba.conf

    # Update your client authentication methods as appropriate
    local   all all                 md5
    host    all all 127.0.0.1/32    md5
    host    all all ::1/128         md5
    #host    all all 192.168.1.0/24  md5 #Only required if listening outside host I think

# Tile Server OSM Processing Performance
vi data/postgresql.conf

    # Update to suit your server capabilities (MC CHANGED NOTHING IN ORIG SETUP)
    shared_buffers = 128MB
    checkpoint_segments = 20
    maintenance_work_mem = 256MB
    autovacuum = off

# Restart PostgreSQL Server
sudo systemctl restart postgresql-11.service
sudo systemctl status postgresql-11.service

# Check that PostgreSQL is listening on port 5432 by default
netstat -an | grep 5432

#install PostGIS
sudo yum install postgis25_11

# Create the GIS database (Basic Setup)
sudo su - postgres
psql
    CREATE DATABASE gis WITH ENCODING = 'UTF8';
    \q

# Execute PostGIS SQL Installation Files (still as the postgres user)
export PATH=$PATH:/usr/pgsql-11/bin
psql gis < /usr/pgsql-11/share/contrib/postgis-2.5/postgis.sql
psql gis < /usr/pgsql-11/share/contrib/postgis-2.5/spatial_ref_sys.sql
createuser osm -W # No to all questions
createuser apache -W # No to all questions
echo "grant all on geometry_columns to apache;" | psql gis
echo "grant all on spatial_ref_sys to apache;" | psql gis
echo "grant all on geometry_columns to osm;" | psql gis
echo "grant all on spatial_ref_sys to osm;" | psql gis
exit

# Kernel Configuration change for PostgreSQL OSM Processing Performance
sudo vi /etc/sysctl.conf
    kernel.shmmax=268435456
    #TODO - confirm with someone this persists
sudo sysctl -p
sudo sysctl kernel.shmmax

```

### Todo
Configure and optimise postgres and storage under postgres


## osm2pgsql
Note this installed the following packages: 
* geos-3.4.2-2.el7.x86_64
* proj-4.8.0-4.el7.x86_64
* osm2pgsql-0.92.0-1.el7.

``` bash
sudo yum install osm2pgsql
```

## Apache Web (HTTP) Server

``` bash
# Basic Installation with default configuration
 yum install httpd
```



## Install Boost
``` bash
#The boost versions in centos7 are not high enough for mapnik, need to build from source
# SO WHY ARE THE FOLLOWING PACKAGES INSTALLED IN THE ABOVE YUM INSTALL??: boost boost-thread boost-devel

# Boostrap and install recent boost (get later version as required)
cd ~
JOBS=`grep -c ^processor /proc/cpuinfo`
tar xf boost_1_76_0.tar.bz2
cd boost_1_76_0
./bootstrap.sh
./b2 -d1 -j${JOBS} --with-thread --with-filesystem --with-python --with-regex -sHAVE_ICU=1 --with-program_options --with-system link=shared release toolset=gcc stage
sudo ./b2 -d1 -j${JOBS} --with-thread --with-filesystem --with-python --with-regex -sHAVE_ICU=1 --with-program_options --with-system link=shared release toolset=gcc install

# set up support for libraries installed in /usr/local/lib
sudo bash -c "echo '/usr/local/lib' > /etc/ld.so.conf.d/boost.conf"
sudo ldconfig

```


## Install Mapnik
``` bash
# Clone and Bootstrap
vi /etc/profile.d/pgsql.sh
    $ export PATH=$PATH:/usr/pgsql-11/bin:/usr/pgsql-11/lib:/usr/local/lib
source /etc/profile.d/pgsql.sh
#git clone git://github.com/mapnik/mapnik
cd ~
tar xf master.zip
mv master mapnik
cd mapnik
./bootstrap.sh
./configure

# Handle mapbox/variant.hpp: no such file or directory error - https://github.com/mapnik/mapnik/issues/3246
git submodule sync
git submodule update --init deps/mapbox/variant

# Build and install
make && sudo make install
sudo ldconfig

``` 

# Useful references

https://ircama.github.io/osm-carto-tutorials/tile-server-ubuntu/
Setup postgres: https://www.symmcom.com/docs/how-tos/databases/how-to-install-postgresql-11-x-on-centos-7
Setup PostGIS: https://computingforgeeks.com/how-to-install-postgis-on-centos-7/
	

