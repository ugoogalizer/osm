# osm
Steps to install Open Street Maps on Centos 7 (with aim to install on Red Hat 7 also)
Summarised & updated from: https://knowledgebase.hyperlearning.ai/en/articles/centos-7-open-street-map-tile-server

## Summary

We require the following software: 
 * [Mapnik](https://github.com/mapnik/mapnik) to render tiles, which also requires: 
   * boost
 * [Open Street Maps Carto](https://github.com/gravitystorm/openstreetmap-carto), provides stylesheets for mapping layers in OSM 
 * apache web server - providing the web service to server web pages etc
 * mod_tile and rendrd - cache of tiles
 * PostgreSQL with PostGIS extensions

Optionally, if we want to edit the stylesheets we could consider installing
 * [Kosmtik](https://github.com/kosmtik)

Ignored for now: 
 * Harfbuzz - no idea if we require it...

# Pre-Reqs

## Pre-Downloads

``` bash
cd ~
wget https://downloads.sourceforge.net/boost/boost_1_76_0.tar.bz2
wget https://github.com/mapnik/mapnik/releases/download/v3.1.0/mapnik-v3.1.0.tar.bz2
wget https://github.com/openstreetmap/mod_tile/archive/refs/tags/0.5.tar.gz
wget https://github.com/gravitystorm/openstreetmap-carto/archive/refs/tags/v5.3.1.tar.gz
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
#should already be downloaded from above
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

Note if you have logged out and back in and are seeing g++ compiler issues around c++14, you need to rerun the following: `scl enable devtoolset-7 bash`


## Install Mapnik
``` bash
# Clone and Bootstrap
sudo vi /etc/profile.d/pgsql.sh
    export PATH=$PATH:/usr/pgsql-11/bin:/usr/pgsql-11/lib:/usr/local/lib
source /etc/profile.d/pgsql.sh
echo $PATH #test
#git clone git://github.com/mapnik/mapnik
#should already be downloaded from above
cd ~
tar xf mapnik-v3.1.0.tar.bz2
cd mapnik-v3.1.0
./bootstrap.sh
#did get a strange git error here...
./configure BOOST_LIBS=/opt/boost/lib BOOST_INCLUDES=/opt/boost/includes

#Got the following interesting feedback:
	Note: will build without these OPTIONAL dependencies:
	   - proj (Proj.4 C Projections library | configure with PROJ_LIBS & PROJ_INCLUDES | more info: http://trac.osgeo.org/proj/)
	   - webp (WEBP C library | configure with WEBP_LIBS & WEBP_INCLUDES)
	   - sqlite3 (SQLite3 C Library | configure with SQLITE_LIBS & SQLITE_INCLUDES | more info: https://github.com/mapnik/mapnik/wiki/SQLite)
	   - sqlite_rtree (The SQLite plugin requires libsqlite3 built with RTREE support (-DSQLITE_ENABLE_RTREE=1))
	   - gdal-config (gdal-config program | try setting GDAL_CONFIG SCons option)
	   - ogr (OGR-enabled GDAL C++ Library | configured using gdal-config program | try setting GDAL_CONFIG SCons option | more info: https://github.com/mapnik/mapnik/wiki/OGR)
	   - cairo (Cairo C library | configured using pkg-config | try setting PKG_CONFIG_PATH SCons option)


# Handle mapbox/variant.hpp: no such file or directory error - https://github.com/mapnik/mapnik/issues/3246
# MC SKIPPED THESE TWO STEPS WITH HOPE THE ISSUE HAS BEEN RESOLVED SINCE WRITTEN
#git submodule sync
#git submodule update --init deps/mapbox/variant

# Build and install - note that this takes some time...
make && sudo make install
sudo ldconfig

``` 

Note if you have logged out and back in and are seeing g++ compiler issues around c++14, you need to rerun the following: `scl enable devtoolset-7 bash`

## mod_tile install

``` bash
# Clone and configure
#git clone git://github.com/openstreetmap/mod_tile.git
#should already be downloaded from above
cd ~
cd ~ 
tar xf 0.5.tar.gz
cd mod_tile-0.5
./autogen.sh
./configure

# From where you cloned the Mapnik source (see above) copy Mapnick libraries to /usr/include/mapnik
cp -rf mapnik/include/mapnik/* /usr/include/mapnik
cp mapnik/include/mapnik/geometry/box2d.hpp /usr/include/mapnik

# Build and install mod_tile
make
sudo make install
sudo make install-mod_tile
sudo ldconfig
```


## Carto Install With Stylesheet
``` bash
# Install Carto using NodeJS that we installed earlier
npm install -g carto

#OSM Carto Stylesheet
# Clone
#git clone git://github.com/gravitystorm/openstreetmap-carto.git
#should already be downloaded from above
cd ~
tar xf v5.3.1.tar.gz
cd openstreetmap-carto-5.3.1
#Skipping the following hoping it's fixed by now
#git checkout `git rev-list -n 1 --before="2016-12-04 00:00" master`

# Compile and download shape files
carto project.mml > mapnik.xml
scripts/get-shapefiles.py
``` 

## Renderd and mod_tile configuration

TODO

# Import Map Data into PostgreSQL

TODO

# Test

TODO

# Useful references

https://ircama.github.io/osm-carto-tutorials/tile-server-ubuntu/
Setup postgres: https://www.symmcom.com/docs/how-tos/databases/how-to-install-postgresql-11-x-on-centos-7
Setup PostGIS: https://computingforgeeks.com/how-to-install-postgis-on-centos-7/
General good counter reference for mapnik, boost, node: https://gist.github.com/davidheyman/5417b515b421a99360ca

