# osm
Steps to install Open Street Maps on Centos 7 (with aim to install on Red Hat 7 also)
Summarised & updated from: https://knowledgebase.hyperlearning.ai/en/articles/centos-7-open-street-map-tile-server

## Summary

We require the following software: 
 * [Mapnik](https://github.com/mapnik/mapnik) mapping toolkit used by mod_tile and renderd to render tiles: 
   * boost
 * [Open Street Maps Carto](https://github.com/gravitystorm/openstreetmap-carto), provides stylesheets for mapping layers in OSM 
 * apache web server - providing the web service to server web pages etc
 * [mod_tile and renderd](https://github.com/openstreetmap/mod_tile) - efficiently render and serve raster map tiles 
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

# Build and install mod_tile
cd ~/mod_tile-0.5
make
sudo make install
sudo make install-mod_tile
sudo ldconfig
```


## Carto Install With Stylesheet
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

# Python3 Setup
Unknown if this is required yet - SKIP THIS PYTHON STEP FOR NOW! As apparently the openstreetmap-carto script is only required if we want to make changes to the map style.
``` bash
sudo pip3 install pyyaml requests psycopg2-binary

# note that "requests" is dependent on:
# idna, chardet, certifi, urllib3, requestsidna, chardet, certifi, urllib3, requests

```


```

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

#MC Note - I think this next step is only required if we want to update the map styles.  This exact reference is out of date also, think the new process is to call "get-external-data.py
scripts/get-shapefiles.py
``` 

## Renderd and mod_tile configuration
Note you have to update the path to the mapnik.xml file below

https://github.com/openstreetmap/mod_tile


``` bash
# Configure Renderd
sudo cp renderd.conf renderd.conf.orig
sudo vi /usr/local/etc/renderd.conf

    # Edit where your paths and number of threads differ
    socketname=/var/run/renderd/renderd.sock
    num_threads=4
    plugins_dir=/usr/local/lib/mapnik/input
    font_dir=/usr/local/lib/mapnik/fonts
    XML=/home/renderaccount/openstreetmap-carto-5.3.1/mapnik.xml # See OSM Carto Stylesheet section
    HOST=127.0.0.1

# Configure mod_tile
vi /etc/httpd/conf.d/mod_tile.conf

    # Edit the ServerName and ServerAlias to suit your server
    # Also update LoadTileConfigFile and ModTileRenderdSocketName if this differs on your server
    LoadModule tile_module /etc/httpd/modules/mod_tile.so
    <VirtualHost *:80>
        ServerName map1.earth.dev.hyperlearning.ai
        ServerAlias a.map1.earth.dev.hyperlearning.ai b.map1.earth.dev.hyperlearning.ai c.map1.earth.dev.hyperlearning.ai d.map1.earth.dev.hyperlearning.ai
        DocumentRoot /var/www/html

        # Specify the default base storage path for where tiles live. A number of different storage backends
        # are available, that can be used for storing tiles.  Currently these are a file based storage, a memcached
        # based storage and a RADOS based storage.
        # The file based storage uses a simple file path as its storage path ( /path/to/tiledir )
        # The RADOS based storage takes a location to the rados config file and a pool name ( rados://poolname/path/to/ceph.conf )
        # The memcached based storage currently has no configuration options and always connects to memcached on localhost ( memcached:// )
        #
        # The storage path can be overwritten on a style by style basis from the style TileConfigFile
        ModTileTileDir /var/lib/mod_tile

        # You can either manually configure each tile set with the default png extension and mimetype
        #    AddTileConfig /folder/ TileSetName
        # or manually configure each tile set, specifying the file extension
        #    AddTileMimeConfig /folder/ TileSetName js
        # or load all the tile sets defined in the configuration file into this virtual host.
        # Some tile set specific configuration parameters can only be specified via the configuration file option
        LoadTileConfigFile /usr/local/etc/renderd.conf

        # Specify if mod_tile should keep tile delivery stats, which can be accessed from the URL /mod_tile
        # The default is On. As keeping stats needs to take a lock, this might have some performance impact,
        # but for nearly all intents and purposes this should be negligable ans so it is safe to keep this turned on.
        ModTileEnableStats On

        # Turns on bulk mode. In bulk mode, mod_tile does not request any dirty tiles to be rerendered. Missing tiles
        # are always requested in the lowest priority. The default is Off.
        ModTileBulkMode Off
        ModTileRequestTimeout 3

        # Timeout before giving up for a tile to be rendered that is otherwise missing
        ModTileMissingRequestTimeout 10

        # If tile is out of date, don't re-render it if past this load threshold (users gets old tile)
        ModTileMaxLoadOld 16

        # If tile is missing, don't render it if past this load threshold (user gets 404 error)
        ModTileMaxLoadMissing 50

        # Sets how old an expired tile has to be to be considered very old and therefore get elevated priority in rendering
        ModTileVeryOldThreshold 31536000000000

        # Unix domain socket where we connect to the rendering daemon
        ModTileRenderdSocketName /var/run/renderd/renderd.sock

        # Alternatively you can use a TCP socket to connect to renderd. The first part
        # is the location of the renderd server and the second is the port to connect to.
        #   ModTileRenderdSocketAddr renderd.mydomain.com 7653

        ##
        ## Options controlling the cache proxy expiry headers. All values are in seconds.
        ##
        ## Caching is both important to reduce the load and bandwidth of the server, as
        ## well as reduce the load time for the user. The site loads fastest if tiles can be
        ## taken from the users browser cache and no round trip through the internet is needed.
        ## With minutely or hourly updates, however there is a trade-off between cacheability
        ## and freshness. As one can't predict the future, these are only heuristics, that
        ## need tuning.
        ## If there is a known update schedule such as only using weekly planet dumps to update the db,
        ## this can also be taken into account through the constant PLANET_INTERVAL in render_config.h
        ## but requires a recompile of mod_tile

        ## The values in this sample configuration are not the same as the defaults
        ## that apply if the config settings are left out. The defaults are more conservative
        ## and disable most of the heuristics.

        ##
        ## Caching is always a trade-off between being up to date and reducing server load or
        ## client side latency and bandwidth requirements. Under some conditions, like poor
        ## network conditions it might be more important to have good caching rather than the latest tiles.
        ## Therefor the following config options allow to set a special hostheader for which the caching
        ## behaviour is different to the normal heuristics
        ##
        ## The CacheExtended parameters overwrite all other caching parameters (including CacheDurationMax)
        ## for tiles being requested via the hostname CacheExtendedHostname
        #ModTileCacheExtendedHostname cache.tile.openstreetmap.org
        #ModTileCacheExtendedDuration 2592000

        # Upper bound on the length a tile will be set cacheable, which takes
        # precedence over other settings of cacheing
        ModTileCacheDurationMax 604800

        # Sets the time tiles can be cached for that are known to by outdated and have been
        # sent to renderd to be rerendered. This should be set to a value corresponding
        # roughly to how long it will take renderd to get through its queue. There is an additional
        # fuzz factor on top of this to not have all tiles expire at the same time
        ModTileCacheDurationDirty 900

        # Specify the minimum time mod_tile will set the cache expiry to for fresh tiles. There
        # is an additional fuzz factor of between 0 and 3 hours on top of this.
        ModTileCacheDurationMinimum 10800

        # Lower zoom levels are less likely to change noticeable, so these could be cached for longer
        # without users noticing much.
        # The heuristic offers three levels of zoom, Low, Medium and High, for which different minimum
        # cacheing times can be specified.

        #Specify the zoom level below  which Medium starts and the time in seconds for which they can be cached
        ModTileCacheDurationMediumZoom 13 86400

        #Specify the zoom level below which Low starts and the time in seconds for which they can be cached
        ModTileCacheDurationLowZoom 9 518400

        # A further heuristic to determine cacheing times is when was the last time a tile has changed.
        # If it hasn't changed for a while, it is less likely to change in the immediate future, so the
        # tiles can be cached for longer.
        # For example, if the factor is 0.20 and the tile hasn't changed in the last 5 days, it can be cached
        # for up to one day without having to re-validate.
        ModTileCacheLastModifiedFactor 0.20

        ## Tile Throttling
        ## Tile scrappers can often download large numbers of tiles and overly straining tileserver resources
        ## mod_tile therefore offers the ability to automatically throttle requests from ip addresses that have
        ## requested a lot of tiles.
        ## The mechanism uses a token bucket approach to shape traffic. I.e. there is an initial pool of n tiles
        ## per ip that can be requested arbitrarily fast. After that this pool gets filled up at a constant rate
        ## The algorithm has two metrics. One based on overall tiles served to an ip address and a second one based on
        ## the number of requests to renderd / tirex to render a new tile.

        ## Overall enable or disable tile throttling
        ModTileEnableTileThrottling Off
        # Specify if you want to use the connecting IP for throtteling, or use the X-Forwarded-For header to determin the
        # IP address to be used for tile throttling. This can be useful if you have a reverse proxy / http accellerator
        # in front of your tile server.
        # 0 - don't use X-Forward-For and allways use the IP that apache sees
        # 1 - use the client IP address, i.e. the first entry in the X-Forwarded-For list. This works through a cascade of proxies.
        #     However, as the X-Forwarded-For is written by the client this is open to manipulation and can be used to circumvent the throttling
        # 2 - use the last specified IP in the X-Forwarded-For list. If you know all requests come through a reverse proxy
        #     that adds an X-Forwarded-For header, you can trust this IP to be the IP the reverse proxy saw for the request
        ModTileEnableTileThrottlingXForward 0
        ## Parameters (poolsize in tiles and topup rate in tiles per second) for throttling tile serving.
        ModTileThrottlingTiles 10000 1
        ## Parameters (poolsize in tiles and topup rate in tiles per second) for throttling render requests.
        ModTileThrottlingRenders 128 0.2

        ###
        ###
        # increase the log level for more detailed information
        LogLevel debug

    </VirtualHost>
```

# Import Map Data into PostgreSQL

TODO

# Test

TODO

# Useful references

Once again, the main source: https://knowledgebase.hyperlearning.ai/en/articles/centos-7-open-street-map-tile-server
An alternate source: https://gist.github.com/coder4web/39e981adbc859db62576dc6840a4023b
https://ircama.github.io/osm-carto-tutorials/tile-server-ubuntu/
Setup postgres: https://www.symmcom.com/docs/how-tos/databases/how-to-install-postgresql-11-x-on-centos-7
Setup PostGIS: https://computingforgeeks.com/how-to-install-postgis-on-centos-7/
General good counter reference for mapnik, boost, node: https://gist.github.com/davidheyman/5417b515b421a99360ca

