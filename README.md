# osm

Summarised & updated from: https://knowledgebase.hyperlearning.ai/en/articles/centos-7-open-street-map-tile-server

## Create local user 
Steps not in the linked article
``` bash
sudo su -
	adduser renderaccount
	passwd renderaccount
  usermod -aG wheel renderaccount
```

## CentOS 7 Minimal ISO Dependencies
There are some pre-requisite packages that need to be installed on your CentOS 7 server before we can get started installing the dependencies listed above. This guide assumes that you have installed the minimal ISO for CentOS 7 (command-line only). If not, or depending on other packages you may have installed historically, some of the pre-requisitie packages listed below may already be installed on your server.

``` bash
# Dependencies
sudo yum install libpng libtiff libjpeg freetype gdal cairo pycairo sqlite geos boost curl libcurl libicu bzip2-devel libpng-devel libtiff-devel zlib-devel libjpeg-devel libxml2-devel python-setuptools proj-devel proj proj-epsg proj-nad freetype-devel libicu-devel gdal-devel sqlite-devel libcurl-devel cairo-devel pycairo-devel geos-devel protobuf-devel protobuf-c-devel lua-devel cmake proj boost-thread proj-devel autoconf automake libtool pkgconfig ragel gtk-doc glib2 glib2-devel libpng libpng-devel libwebp libtool-ltdl-devel python-devel harfbuzz harfbuzz-devel harfbuzz-icu boost-devel cabextract xorg-x11-font-utils fontconfig perl-DBD-Pg mesa-libGLU-devel

# GCC++ 14 standards are required for Mapnik so we shall install the Dev Toolset from the CentOS Software Collections
#in red hat would instead be something like:   yum-config-manager --enable rhel-server-rhscl-7-rpms
sudo yum install centos-release-scl
sudo yum install devtoolset-7
sudo scl enable devtoolset-7 bash

# Carto can be installed via NodeJS so we will install NodeJS too
sudo yum install nodejs
npm -v

# Git Version Control is required to clone relevant dependency source code repositories
sudo yum install git
```
