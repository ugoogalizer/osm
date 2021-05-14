# osm
Steps to install Open Street Maps on Centos 7 (with aim to install on Red Hat 7 also), ultimately in a non-internet connected environment.
Summarised & updated from: https://knowledgebase.hyperlearning.ai/en/articles/centos-7-open-street-map-tile-server

## Summary

We require the following software: 
* Map Rendering Software:
  * [Mapnik](https://github.com/mapnik/mapnik) mapping toolkit used by mod_tile and renderd to render tiles: 
    * boost
  * [mod_tile and renderd](https://github.com/openstreetmap/mod_tile) - efficiently render and serve raster map tiles 
* Map Stylesheet Defining (and editing) software - THIS APPEARS OPTIONAL IF YOU CREATE STYLESHEETS SEPARATELY AND IMPORT THEM INTO PRODUCTION:
  * [Open Street Maps Carto](https://github.com/gravitystorm/openstreetmap-carto), provides stylesheets for mapping layers in OSM 
  * [Carto](https://www.npmjs.com/package/carto)
* Interfaces:
  * "httpd" apache web server - providing the web service to server web pages etc
* Storage:
  * PostgreSQL with PostGIS extensions (v11 chosen here)

Optionally, if we want to edit the stylesheets we could consider installing
 * [Kosmtik](https://github.com/kosmtik)

Ignored for now: 
 * Harfbuzz - no idea if we require it...

# Pre-Reqs

## Pre-Downloads
For offline installation, the following downloads are required ahead:
``` bash
cd ~
wget https://downloads.sourceforge.net/boost/boost_1_76_0.tar.bz2
wget https://github.com/mapnik/mapnik/releases/download/v3.1.0/mapnik-v3.1.0.tar.bz2
wget https://github.com/openstreetmap/mod_tile/archive/refs/tags/0.5.tar.gz
wget https://github.com/gravitystorm/openstreetmap-carto/archive/refs/tags/v5.3.1.tar.gz # MIGHT BE OPTIONAL FOR PROD
wget https://noto-website-2.storage.googleapis.com/pkgs/Noto-unhinted.zip # Google fonts for map rendering
wget https://github.com/mapbox/mason/archive/refs/tags/v0.23.0.tar.gz

```

## OSM "External Data"

OSM has a set of fixed data on things like water bodies, continent boundaries, icesheets etc.  This information is normally downloaded using the openstreetmap-carto-5.3.1/scripts/get-external-data.py script.  In order to run this offline, this data needs to be hosted elsewhere.  Once openstreetmap-carto-5.3.1 is extracted, have a look at the configuration file: `external-data.yml` to get the latest file names, but in summary the following need to be cached:

```
wget https://osmdata.openstreetmap.de/download/simplified-water-polygons-split-3857.zip
wget https://osmdata.openstreetmap.de/download/water-polygons-split-3857.zip
wget https://osmdata.openstreetmap.de/download/antarctica-icesheet-polygons-3857.zip
wget https://osmdata.openstreetmap.de/download/antarctica-icesheet-outlines-3857.zip
wget https://naciscdn.org/naturalearth/110m/cultural/ne_110m_admin_0_boundary_lines_land.zip

#Testing purposes (sample leaflet app)
wget https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/extra/sample_leaflet.html
```

And then the `external-data.yml` will need to be updated to point to the new location of these files.


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
sudo yum install nodejs # MIGHT BE OPTIONAL IN PROD
npm -v # MIGHT BE OPTIONAL IN PROD

# Git Version Control is required to clone relevant dependency source code repositories
sudo yum install git

# newer versions of openstreetmap-carto require ogr2ogr, so need to install gdal:
sudo yum install gdal
```

# Installation

## Postgres and PostGIS

```
# Install Postgres
sudo rpm -Uvh https://yum.postgresql.org/11/redhat/rhel-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm
sudo yum -y install postgresql11-server postgresql11 postgresql11-devel

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
sudo su - postgres
  cd /var/lib/pgsql/11
  cp data/pg_hba.conf data/pg_hba.conf.orig
  vi data/pg_hba.conf

    # Update your client authentication methods as appropriate
    local   all all                 ident # this allows local OS users to log in to postgres as themselves without further authentication requests
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
  exit

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
# sudo -u postgres psql gis --command='CREATE EXTENSION postgis;'
sudo -u postgres psql gis --command='CREATE EXTENSION hstore;'
#createuser osm -W # No to all questions
createuser apache -W # No to all questions
createuser renderaccount -W # No to all questions
echo "grant all on geometry_columns to apache;" | psql gis
echo "grant all on spatial_ref_sys to apache;" | psql gis
#echo "grant all on geometry_columns to osm;" | psql gis
#echo "grant all on spatial_ref_sys to osm;" | psql gis
echo "grant all on geometry_columns to renderaccount;" | psql gis
echo "grant all on spatial_ref_sys to renderaccount;" | psql gis
exit

#as postgres (sorry for order here...) 
sudo su - postgres
psql
    ALTER DATABASE gis OWNER TO renderaccount; # Needs to be done for the get-excternal-data.py step later
    ALTER TABLE geometry_columns OWNER TO renderaccount;
    ALTER TABLE spatial_ref_sys OWNER TO renderaccount;
    \q
    
# Kernel Configuration change for PostgreSQL OSM Processing Performance
sudo vi /etc/sysctl.conf
    kernel.shmmax=268435456
    #TODO - confirm with someone this persists
sudo sysctl -p
sudo sysctl kernel.shmmax

```

###

### PostGres and PostGIS summary: 

Database name: gis
Loader username: osm
Web reader username: apache


### Remaining Todos
* Configure and optimise postgres and storage under postgres. Appears to be some good suggestions here: https://osm2pgsql.org/doc/manual.html
* Ensure that the postgres database storage is on appropriate storage volume (as it's going to get big!)

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
 yum install httpd httpd-devel
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

#now have to manually extract mason in place, otherwide bootstrap.sh will error trying to get it itself
tar xf v0.23.0.tar.gz
mv mason-0.23.0 mapnik-v3.1.0/.mason

#Now bootstrap and install
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
tar xf 0.5.tar.gz
cd mod_tile-0.5
./autogen.sh
./configure

# From where you cloned the Mapnik source (see above) copy Mapnick libraries to /usr/include/mapnik
sudo mkdir /usr/include/mapnik
sudo cp -rf ~/mapnik-v3.1.0/include/mapnik/* /usr/include/mapnik
#NOTE MC ADJUSTED the below line to be all because box2d.hpp wasn't in that folder
#sudo cp ~/mapnik-v3.1.0/include/mapnik/geometry/box2d.hpp /usr/include/mapnik
sudo cp ~/mapnik-v3.1.0/include/mapnik/geometry/* /usr/include/mapnik
sudo find /usr/include/mapnik -type d - exec chmod 755 {} +
sudo find /usr/include/mapnik -type d - exec chmod 644 {} +


# Build and install mod_tile
cd ~/mod_tile-0.5
make
sudo make install
sudo make install-mod_tile
sudo ldconfig
```


## Carto Install With Stylesheet # N NOT REQUIRED IN PROD
Unknown how to achieve this offline yet...
``` bash
# Install Carto using NodeJS that we installed earlier
npm install -g carto

```
Note this then installed the following packages!

```
└─┬ carto@1.2.0
  ├── chroma-js@1.3.7
  ├── hsluv@0.0.3
  ├─┬ js-yaml@3.12.2
  │ ├─┬ argparse@1.0.10
  │ │ └── sprintf-js@1.0.3
  │ └── esprima@4.0.1
  ├── lodash@4.17.21
  ├── mapnik-reference@8.10.0
  ├── semver@5.6.0
  └─┬ yargs@12.0.5
    ├─┬ cliui@4.1.0
    │ ├─┬ strip-ansi@4.0.0
    │ │ └── ansi-regex@3.0.0
    │ └─┬ wrap-ansi@2.1.0
    │   ├─┬ string-width@1.0.2
    │   │ ├── code-point-at@1.1.0
    │   │ └─┬ is-fullwidth-code-point@1.0.0
    │   │   └── number-is-nan@1.0.1
    │   └─┬ strip-ansi@3.0.1
    │     └── ansi-regex@2.1.1
    ├── decamelize@1.2.0
    ├─┬ find-up@3.0.0
    │ └─┬ locate-path@3.0.0
    │   ├─┬ p-locate@3.0.0
    │   │ └─┬ p-limit@2.3.0
    │   │   └── p-try@2.2.0
    │   └── path-exists@3.0.0
    ├── get-caller-file@1.0.3
    ├─┬ os-locale@3.1.0
    │ ├─┬ execa@1.0.0
    │ │ ├─┬ cross-spawn@6.0.5
    │ │ │ ├── nice-try@1.0.5
    │ │ │ ├── path-key@2.0.1
    │ │ │ ├─┬ shebang-command@1.2.0
    │ │ │ │ └── shebang-regex@1.0.0
    │ │ │ └─┬ which@1.3.1
    │ │ │   └── isexe@2.0.0
    │ │ ├─┬ get-stream@4.1.0
    │ │ │ └─┬ pump@3.0.0
    │ │ │   ├── end-of-stream@1.4.4
    │ │ │   └─┬ once@1.4.0
    │ │ │     └── wrappy@1.0.2
    │ │ ├── is-stream@1.1.0
    │ │ ├── npm-run-path@2.0.2
    │ │ ├── p-finally@1.0.0
    │ │ ├── signal-exit@3.0.3
    │ │ └── strip-eof@1.0.0
    │ ├─┬ lcid@2.0.0
    │ │ └── invert-kv@2.0.0
    │ └─┬ mem@4.3.0
    │   ├─┬ map-age-cleaner@0.1.3
    │   │ └── p-defer@1.0.0
    │   ├── mimic-fn@2.1.0
    │   └── p-is-promise@2.1.0
    ├── require-directory@2.1.1
    ├── require-main-filename@1.0.1
    ├── set-blocking@2.0.0
    ├─┬ string-width@2.1.1
    │ └── is-fullwidth-code-point@2.0.0
    ├── which-module@2.0.0
    ├── y18n@4.0.3
    └─┬ yargs-parser@11.1.1
      └── camelcase@5.3.1

```

## Setup the Carto Stylesheet

``` bash

#OSM Carto Stylesheet
# Clone
#git clone git://github.com/gravitystorm/openstreetmap-carto.git
#should already be downloaded from above
cd ~
tar xf v5.3.1.tar.gz
cd openstreetmap-carto-5.3.1
#Skipping the following hoping it's fixed by now
#git checkout `git rev-list -n 1 --before="2016-12-04 00:00" master`

# Compile and download shape files, NOTE the first one threw quite a few Warnings for me
carto project.mml > mapnik.xml

#MC Note - I think this next step is only required if we want to update the map styles.  This exact reference is out of date also, think the new process is to call "get-external-data.py -- see below
scripts/get-shapefiles.py
``` 

## Get External Data

Get "external data", which I think is basically the barely changing boundaries of land masses (continents) 

First install python and some packages: 
```
sudo yum install python3
sudo pip3 install pyyaml requests psycopg2-binary
```

If you have external internet access:
``` bash
#as renderaccount
cd ~/openstreetmap-carto-5.3.1/
mkdir data
sudo chown renderaccount data
scripts/get-external-data.py
#Note this takes a very long time (10's of minutes) as it downloads large volumes of data, and you can see it slowly populate the "~/openstreetmap-carto-5.3.1/data" directory

```

If you have to be offline: 
``` bash
#change directories to whereever you have uploaded the external data zip files
cd ~
cp simplified-water-polygons-split-3857.zip water-polygons-split-3857.zip antarctica-icesheet-polygons-3857.zip antarctica-icesheet-outlines-3857.zip ne_110m_admin_0_boundary_lines_land.zip /var/www/html
cd /openstreetmap-carto-5.3.1/
mkdir data
sudo chown renderaccount data
cp external-data.yml external-data.yml.orig
vi external-data.yml
    #Change all the references to websites instead to http://127.0.0.1/
scripts/get-external-data.py
#Note this takes a very long time (10's of minutes) as it downloads large volumes of data, and you can see it slowly populate the "~/openstreetmap-carto-5.3.1/data" directory

```



## Alternate approach without installing carto (for PROD System - untested)
Copy across tghe following files from a system that has generated them: 
* indexes.sql
* mapnik.xml
Into the folder `~/openstreetmap-carto-5.3.1`

Then as renderaccount: 
``` bash 
cd ~/openstreetmap-carto-5.3.1
psql -d gis -f indexes.sql
```


## Renderd and mod_tile configuration
Note you have to update the path to the mapnik.xml file below

``` bash
# Configure Renderd
sudo cp /usr/local/etc/renderd.conf /usr/local/etc/renderd.conf.orig
sudo vi /usr/local/etc/renderd.conf

    # Edit where your paths and number of threads differ
    socketname=/var/run/renderd/renderd.sock
    num_threads=4
    plugins_dir=/usr/local/lib/mapnik/input
    TILEDIR={SOME LOCATION WITH LOTS OF STORAGE}
    font_dir=/usr/local/lib/mapnik/fonts
    XML=/home/renderaccount/openstreetmap-carto-5.3.1/mapnik.xml # See OSM Carto Stylesheet section
    HOST=127.0.0.1

# Configure mod_tile
sudo cp /home/renderaccount/mod_tile-0.5/debian/tileserver_site.conf /etc/httpd/conf.d/mod_tile.conf
sudo vi /etc/httpd/conf.d/mod_tile.conf

    #Add this line before the "<VirtualHost *:80>" line: 
    LoadModule tile_module /etc/httpd/modules/mod_tile.so
    
    #then make the following changes: 
    
    DocumentRoot /var/www/html
    LoadTileConfigFile /usr/local/etc/renderd.conf
```


Then configured renderd to run: 

``` bash
sudo mkdir /var/run/renderd
sudo mkdir /var/lib/mod_tile
sudo chown renderaccount:renderaccount /var/run/renderd
sudo chown renderaccount:renderaccount /var/lib/mod_tile
sudo chmod 755 /var/run/renderd
sudo chmod 755 /var/lib/mod_tile
```

## Install Google Fonts

```
cd ~
# https://noto-website-2.storage.googleapis.com/pkgs/Noto-unhinted.zip
mkdir noto
mv Noto-unhinted.zip noto/
cd noto/
unzip Noto-unhinted.zip
sudo mkdir -p /usr/local/lib/mapnik/fonts/noto
sudo mv *otf *otc *ttf /usr/local/lib/mapnik/fonts/noto

```

# Import Map Data into PostgreSQL


``` bash
# Download OSM Map Data
# In this example, I am downloading the Greater London OSM Map Data from geofabrik.de
export PATH=$PATH:/usr/pgsql-11/bin
cd ~
wget http://download.geofabrik.de/europe/great-britain/england/greater-london-latest.osm.pbf

# Process this OSM Map Data into the PostGIS-enabled PostgreSQL Database
#osm2pgsql --slim -d gis -C 1600 --number-process 4 -S /usr/local/share/osm2pgsql/default.style greater-london-latest.osm.pbf
#osm2pgsql --slim -d gis --hstore -C 16000 --number-process 4 -S /usr/share/osm2pgsql/default.style ~/greater-london-latest.osm.pbf
osm2pgsql --slim -d gis --hstore -C 16000 --number-process 4 -S /home/renderaccount/openstreetmap-carto-5.3.1/openstreetmap-carto.style --tag-transform-script /home/renderaccount/openstreetmap-carto-5.3.1/openstreetmap-carto.lua ~/greater-london-latest.osm.pbf

#Now ensure that indexes are created (as renderaccount):
cd ~/penstreetmap-carto-5.3.1
psql -d gis -f indexes.sql
```

## Autostarting renderd

```
# renderd.service
#AS ROOT
cat >/lib/systemd/system/renderd.service <<CMD_EOF
[Unit]
Description=Mapnik rendering daemon
After=syslog.target
After=network.target
[Service]
User=renderaccount
Environment="PATH=/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin"
ExecStart=/usr/local/bin/renderd -f -c /usr/local/etc/renderd.conf
#WorkingDirectory=
ExecReload=/bin/kill -HUP $MAINPID
KillMode=process
Restart=on-failure
[Install]
WantedBy=multi-user.target
CMD_EOF

exit

#as renderaccount:
sudo systemctl enable renderd
sudo systemctl start renderd
tail -f /var/log/messages |grep renderd
```



## Running renderd manually (testing)
As renderaccount
```
renderd -f -c /usr/local/etc/renderd.conf

```

## Spinning up a Test Leaflet Website
As renderaccount
``` bash
cd /var/www/html
sudo wget https://raw.githubusercontent.com/SomeoneElseOSM/mod_tile/switch2osm/extra/sample_leaflet.html
sudo vi sample_leaflet.html
    # Update the following line to point to your server & endpoint:
    # i.e. change the IP address to the IP address of your server
    L.tileLayer('http://127.0.0.1/osm_tiles/{z}/{x}/{y}.png', {
    
```
Then browse to http://127.0.0.1/sample_leaflet.html#10/-5.0355/122.5580

# Useful references

Once again, the main source: https://knowledgebase.hyperlearning.ai/en/articles/centos-7-open-street-map-tile-server
An alternate source: https://gist.github.com/coder4web/39e981adbc859db62576dc6840a4023b
https://ircama.github.io/osm-carto-tutorials/tile-server-ubuntu/
Setup postgres: https://www.symmcom.com/docs/how-tos/databases/how-to-install-postgresql-11-x-on-centos-7
Setup PostGIS: https://computingforgeeks.com/how-to-install-postgis-on-centos-7/
General good counter reference for mapnik, boost, node: https://gist.github.com/davidheyman/5417b515b421a99360ca
How to install Google Noto fonts: https://www.google.com/get/noto/help/install/

## Diagram 

Open StreetMaps overview sourced from [here](https://wiki.openstreetmap.org/wiki/Component_overview)

![image](https://user-images.githubusercontent.com/1974156/117956026-fe9c8b00-b35b-11eb-89fd-3300155470d7.png)
