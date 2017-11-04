# PHP-FPM 7.3 Docker Image

[![Build Status](https://travis-ci.org/devilbox/docker-php-fpm-7.3.svg?branch=master)](https://travis-ci.org/devilbox/docker-php-fpm-7.3)
[![Join the chat at https://gitter.im/devilbox/Lobby](https://badges.gitter.im/devilbox/Lobby.svg)](https://gitter.im/devilbox/Lobby?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge&utm_content=badge)
[![](https://images.microbadger.com/badges/license/devilbox/php-fpm-7.3.svg)](https://microbadger.com/images/devilbox/php-fpm-7.3 "php-fpm-7.3")
[![Github](https://img.shields.io/badge/github-docker--php--fpm--7.3-red.svg)](https://github.com/devilbox/docker-php-fpm-7.3)

**[devilbox/docker-php-fpm-7.3](https://github.com/devilbox/docker-php-fpm-7.3)**

This repository will provide you fully functional PHP-FPM 7.3 Docker images in different flavours and packed with different types of integrated PHP modules. It also solves the problem of [syncronizing file permissions](#unsynchronized-permissions) of mounted volumes between the host and the container.

Choose between [Alpine](https://www.alpinelinux.org) or [Debian](https://www.debian.org) and select an image type for extending, use in production or use for local development. Images are always guaranteed to be fresh and up-to-date as they are automatically built every night by [travis-ci](https://travis-ci.org/devilbox/docker-php-fpm-7.3) and pushed to [Docker hub](https://hub.docker.com/r/devilbox/php-fpm-7.3/).


| Docker Hub | Upstream Project |
|------------|------------------|
| <a href="https://hub.docker.com/r/devilbox/php-fpm-7.3"><img height="82px" src="http://dockeri.co/image/devilbox/php-fpm-7.3" /></a> | <a href="https://github.com/cytopia/devilbox" ><img height="82px" src="https://raw.githubusercontent.com/devilbox/artwork/master/submissions_banner/cytopia/01/png/banner_256_trans.png" /></a> |

**Select different version**

**[PHP 5.4](https://github.com/cytopia/docker-php-fpm-5.4)** |
**[PHP 5.5](https://github.com/cytopia/docker-php-fpm-5.5)** |
**[PHP 5.6](https://github.com/cytopia/docker-php-fpm-5.6)** |
**[PHP 7.0](https://github.com/cytopia/docker-php-fpm-7.0)** |
**[PHP 7.1](https://github.com/cytopia/docker-php-fpm-7.1)** |
**[PHP 7.2](https://github.com/cytopia/docker-php-fpm-7.2)** |
**PHP 7.3** |
**[HHVM latest](https://github.com/cytopia/docker-hhvm-latest)**

---

#### Table of Contents

1. **[Motivation](#motivation)**
  1. [Unsynchronized permissions](#unsynchronized-permissions)
  2. [It gets even worse](#it-gets-even-worse)
  3. [The solution](#the-solution)
2. **[PHP-FPM 7.3 Flavours](#php-fpm-7.3-flavours)**
  1. [Assembly](#assembly)
  2. [Available Images](#available-images)
  3. [Tagging](#tagging)
  4. [PHP Modules](#php-modules)
3. **[PHP-FPM 7.3 Features](#php-fpm-7.3-features)**
  1. [Image: base](#image-base)
  2. [Image: mods](#image-mods)
  3. [Image: prod](#image-prod)
  4. [Image: work](#image-work)
4. **[PHP-FPM 7.3 Options](#php-fpm-7.3-options)**
  1. [Environment variables](#environment-variables)
  2. [Volumes](#volumes)
  3. [Ports](#ports)
5. **[Examples](#examples)**
6. **[Automated builds](#automated-builds)**
7. **[Contributing](#contributing)**
8. **[Credits](#credits)**
9. **[License](#license)**

----

<h2><img id="motivation" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> Motivation</h2>

One main problem with a running Docker container is to **synchronize the ownership of files in a mounted volume** in order to preserve security (Not having to use `chmod 0777`).


#### Unsynchronized permissions

Consider the following directory structure of a mounted volume. Your hosts computer uid/gid are `1000` which does not have a corresponding user/group within the container. Fortunately the `tmp/` directory allows everybody to create new files in it. 

```
                  [Host]                   |             [Container]
------------------------------------------------------------------------------------------
 $ ls -l                                   | $ ls -l
 -rw-r--r-- user group index.php           | -rw-r--r-- 1000 1000 index.php
 drwxrwxrwx user group tmp/                | drwxrwxrwx 1000 1000 tmp/
```

Your web application might now have created some temporary files (via the PHP-FPM process) inside the `tmp/` directory:

```
                  [Host]                   |             [Container]
------------------------------------------------------------------------------------------
 $ ls -l tmp/                              | $ ls -l tmp/
 -rw-r--r-- 96 96 _tmp_cache01.php         | -rw-r--r-- www www _tmp_cache01.php
 -rw-r--r-- 96 96 _tmp_cache02.php         | -rw-r--r-- www www _tmp_cache01.php
```

On the Docker container side everything is still fine, but on your host computers side, those files now show a user id and group id of `96`, which is in fact the uid/gid of the PHP-FPM process running inside the container. On the host side you will now have to use `sudo` in order to delete/edit those files.

#### It gets even worse

Consider your had created the `tmp/` directory on your host only with `0775` permissions:

```
                  [Host]                   |             [Container]
------------------------------------------------------------------------------------------
 $ ls -l                                   | $ ls -l
 -rw-r--r-- user group index.php           | -rw-r--r-- 1000 1000 index.php
 drwxrwxr-x user group tmp/                | drwxrwxr-x 1000 1000 tmp/
```

If your web application now wants to create some temporary files (via the PHP-FPM process) inside the `tmp/` directory, it will fail due to lacking permissions.

#### The solution

To overcome this problem, it must be made sure that the PHP-FPM process inside the container runs under the same uid/gid as your local user that mouns the volumes and also wants to work on those files locally. However, you never know during Image build time what user id this would be. Therefore it must be something that can be changed during startup of the container.

This is achieved by two environment variables that can be provided during startup in order to change the uid/gid of the PHP-FPM user prior starting up PHP-FPM.

```
$ docker run -e NEW_UID=1000 -e NEW_GID=1000 -it devilbox/php-fpm-7.3:base-alpine
[INFO] Changing user 'devilbox' uid to: 1000
root $ usermod -u 1000 devilbox
[INFO] Changing group 'devilbox' gid to: 1000
root $ groupmod -g 1000 devilbox
[INFO] Starting PHP 7.3.0-dev (fpm-fcgi) (built: Oct 30 2017 12:05:19)
```

When **`NEW_UID`** and **`NEW_GID`** are provided to the startup command, the container will do a `usermod` and `groupmod` prior starting up in order to assign new uid/gid to the PHP-FPM user. When the PHP-FPM process finally starts up it actually runs with your local system user and making sure permissions will be in sync from now on.

At a minimum those two environment variables are offered by all flavours and types of the here provided PHP-FPM images.

**Note:**

To tackle this on the PHP-FPM side is only half a solution to the problem. The same applies to a web server Docker container when you offer **file uploads**. They will be uploaded and created by the web servers uid/gid. Therefore the web server itself must also provide the same kind of solution. See the following Web server Docker images for how this is done:

**[Apache 2.2](https://github.com/devilbox/docker-apache-2.2)** |
**[Apache 2.4](https://github.com/devilbox/docker-apache-2.4)** |
**[Nginx stable](https://github.com/devilbox/docker-nginx-stable)** |
**[Nginx mainline](https://github.com/devilbox/docker-nginx-mainline)**


<h2><img id="php-fpm-7.3-flavours" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> PHP-FPM 7.3 Flavours</h2>

#### Assembly

The provided Docker images heavily rely on inheritance to guarantee smallest possible image size. Each of them provide a working PHP-FPM server and you must decide what version works best for you. Look at the sketch below to get an overview about the two provided flavours and each of their different types.

```
      [Alpine]               [Debian]          # Base FROM image
         |                      |              #
         |                      |              #
    [base-alpine]          [base-debian]       # Clones PHP git repository, compiles
         |                      |              # and installs it
         |                      |              #
    [mods-alpine]          [mods-debian]       # Installs additional PHP modules
         |                      |              # via pecl
         |                      |              #
    [prod-alpine]          [prod-debian]       # Devilbox flavour for production
         |                      |              # (locales, postifx, socat and injectables)
         |                      |              #
    [work-alpine]          [work-debian]       # Devilbox flavour for local development
                                               # (includes development tools)
```

#### Available Images

The following table shows a more complete overview about the offered Docker images and what they should be used for.

<table>
 <thead>
  <tr>
   <th width="80">Type</th>
   <th width="310">Docker Image</th>
   <th>Description</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td rowspan="2">**base**</td>
   <td>`devilbox/php-fpm-7.3:base-alpine`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:base-alpine.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:base-alpine.svg" /></td>
  </tr>
  <tr>
   <td>`devilbox/php-fpm-7.3:base-debian`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:base-debian.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:base-debian.svg" /></td>
  </tr>
  <tr>
   <td colspan="3"></td>
  </tr>
  <tr>
   <td rowspan="2">**mods**</td>
   <td>`devilbox/php-fpm-7.3:mods-alpine`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:mods-alpine.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:mods-alpine.svg" /></td>
  </tr>
  <tr>
   <td>`devilbox/php-fpm-7.3:mods-debian`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:mods-debian.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:mods-debian.svg" /></td>
  </tr>
  <tr>
   <td colspan="3"></td>
  </tr>
  <tr>
   <td rowspan="2">**prod**</td>
   <td>`devilbox/php-fpm-7.3:prod-alpine`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:prod-alpine.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:prod-alpine.svg" /></td>
  </tr>
  <tr>
   <td>`devilbox/php-fpm-7.3:prod-debian`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:prod-debian.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:prod-debian.svg" /></td>
  </tr>
  <tr>
   <td colspan="3"></td>
  </tr>
  <tr>
   <td rowspan="2">**work**</td>
   <td>`devilbox/php-fpm-7.3:work-alpine`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:work-alpine.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:work-alpine.svg" /></td>
  </tr>
  <tr>
   <td>`devilbox/php-fpm-7.3:work-debian`</td>
   <td><img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:work-debian.svg" /> <img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:work-debian.svg" /></td>
  </tr>
 </tbody>
</table>

#### Tagging

This repository uses Docker tags to refer to different flavours and types of the PHP-FPM 7.3 Docker image. Therefore `:latest` and `:<git-branch-name>` as well as `:<git-tag-name>` must be presented differently. Refer to the following table to see how tagged Docker images are produced at Docker hub:

<table>
 <thead>
  <tr>
   <th width="190">Meant Tag</th>
   <th width="300">Actual Tag</th>
   <th>Comment</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td>`:latest`</td>
   <td><sub>`:base-alpine`<br/>`:base-debian`<br/>`:mods-alpine`<br/>`:mods-debian`<br/>`:prod-alpine`<br/>`:prod-debian`<br/>`:work-alpine`<br/>`:work-debian`</sub></td>
   <td>Stable<br/><sub>(rolling)</sub><br/><br/>These tags are produced by the master branch of this repository.</td>
  </tr>
  <tr>
   <td>`:<git-tag-name>`</td>
   <td><sub>`:base-alpine-<git-tag-name>`<br/>`:base-debian-<git-tag-name>`<br/>`:mods-alpine-<git-tag-name>`<br/>`:mods-debian-<git-tag-name>`<br/>`:prod-alpine-<git-tag-name>`<br/>`:prod-debian-<git-tag-name>`<br/>`:work-alpine-<git-tag-name>`<br/>`:work-debian-<git-tag-name>`</sub></td>
   <td>Stable<br/><sub>(fixed)</sub><br/><br/>Every git tag will produce and preserve these Docker tags.</td>
  </tr>
  <tr>
   <td>`:<git-branch-name>`</td>
   <td><sub>`:base-alpine-<git-branch-name>`<br/>`:base-debian-<git-branch-name>`<br/>`:mods-alpine-<git-branch-name>`<br/>`:mods-debian-<git-branch-name>`<br/>`:prod-alpine-<git-branch-name>`<br/>`:prod-debian-<git-branch-name>`<br/>`:work-alpine-<git-branch-name>`<br/>`:work-debian-<git-branch-name>`</sub></td>
   <td>Feature<br/><sub>(for testing)</sub><br/><br/>Tags produced by unmerged branches. Do not rely on them as they might come and go.</td>
  </tr>
 </tbody>
</table>


#### PHP Modules

Check out this table to see which Docker image provides what PHP modules.

<table>
 <thead>
  <tr>
   <th></th>
   <th width="45%">Alpine</th>
   <th width="45%">Debian</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <th>base</th>
   <td id="mod-base-alpine">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, imap, intl, json, ldap, libxml, mbstring, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
   <td id="mod-base-debian">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, imap, interbase, intl, json, ldap, libxml, mbstring, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, PDO_Firebird, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
  </tr>
  <tr>
   <th>mods</th>
   <td id="mod-mods-alpine">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, igbinary, imap, intl, json, ldap, libxml, mbstring, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, redis, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
   <td id="mod-mods-debian">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, igbinary, imagick, imap, interbase, intl, json, ldap, libxml, mbstring, mongodb, msgpack, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, PDO_Firebird, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, redis, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
  </tr>
  <tr>
   <th>prod</th>
   <td id="mod-prod-alpine">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, igbinary, imap, intl, json, ldap, libxml, mbstring, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, redis, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
   <td id="mod-prod-debian">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, igbinary, imagick, imap, interbase, intl, json, ldap, libxml, mbstring, mongodb, msgpack, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, PDO_Firebird, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, redis, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
  </tr>
  <tr>
   <th>work</th>
   <td id="mod-work-alpine">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, igbinary, imap, intl, json, ldap, libxml, mbstring, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, redis, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
   <td id="mod-work-debian">bcmath, bz2, calendar, Core, ctype, curl, date, dom, enchant, exif, fileinfo, filter, ftp, gd, gettext, gmp, hash, iconv, igbinary, imagick, imap, interbase, intl, json, ldap, libxml, mbstring, mongodb, msgpack, mysqli, mysqlnd, openssl, pcntl, pcre, PDO, PDO_Firebird, pdo_mysql, pdo_pgsql, pdo_sqlite, pgsql, Phar, posix, pspell, readline, redis, Reflection, session, shmop, SimpleXML, snmp, soap, sockets, SPL, sqlite3, standard, sysvmsg, sysvsem, sysvshm, tidy, tokenizer, wddx, xml, xmlreader, xmlrpc, xmlwriter, xsl, zip, zlib</td>
  </tr>
 </tbody>
</table>


<h2><img id="php-fpm-7.3-features" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> PHP-FPM 7.3 Features</h2>

#### Image: base
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:base-alpine.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:base-alpine.svg" /><br/>
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:base-debian.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:base-debian.svg" /> 
```
docker pull devilbox/php-fpm-7.3:base-alpine
docker pull devilbox/php-fpm-7.3:base-debian
```

Generic PHP-FPM base image. Use it to derive your own php-fpm docker image from it and add more extensions, tools and injectables.<br/><br/><sub>(Does not offer any environment variables except for `NEW_UID` and `NEW_GID`)</sub>

#### Image: mods
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:mods-alpine.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:mods-alpine.svg" /><br/>
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:mods-debian.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:mods-debian.svg" /> 
```
docker pull devilbox/php-fpm-7.3:mods-alpine
docker pull devilbox/php-fpm-7.3:mods-debian
```

Generic PHP-FPM image with fully loaded extensions. Use it to derive your own php-fpm docker image from it and add more extensions, tools and injectables.<br/><br/><sub>(Does not offer any environment variables except for `NEW_UID` and `NEW_GID`)</sub></td>

#### Image: prod
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:prod-alpine.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:prod-alpine.svg" /><br/>
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:prod-debian.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:prod-debian.svg" />
```
docker pull devilbox/php-fpm-7.3:prod-alpine
docker pull devilbox/php-fpm-7.3:prod-debian
```

Devilbox production image. This Docker image comes with many injectables, port-forwardings, mail-catch-all and user/group rewriting.

#### Image: work
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:work-alpine.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:work-alpine.svg" /><br/>
<img src="https://images.microbadger.com/badges/version/devilbox/php-fpm-7.3:work-debian.svg" /> <img src="https://images.microbadger.com/badges/image/devilbox/php-fpm-7.3:work-debian.svg" /> 
```
docker pull devilbox/php-fpm-7.3:work-alpine
docker pull devilbox/php-fpm-7.3:work-debian
```

Devilbox development image. Same as prod (alpine), but comes with lots of locally installed tools to make development inside the container as convenient as possible.


<h2><img id="php-fpm-7.3-options" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> PHP-FPM 7.3 Options</h2>

#### Environment variables

Have a look at the following table to see all supported environment variables for each Docker image flavour.

<table>
 <thead>
  <tr>
   <th>Image</th>
   <th>Env Variable</th>
   <th>Type</th>
   <th>Default</th>
   <th>Description</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td rowspan="3">**base**<br/><br/>**mods**<br/><br/>**prod**<br/><br/>**work**</td>
   <td><sub>`DEBUG_ENTRYPOINT`</sub></td>
   <td>int</td>
   <td>`0`</td>
   <td>Set debug level for startup.<br/><sub>`0` Only warnings and errors are shown.<br/>`1` All log messages are shown<br/>`2` All log messages and executed commands are shown.</sub></td>
  </tr>
  <tr>
   <td><sub>`NEW_UID`</sub></td>
   <td>int</td>
   <td>`1000`</td>
   <td>Assign the PHP-FPM user a new `uid` in order to syncronize file system permissions with your host computer and the Docker container. You should use a value that matches your host systems local user.<br/><sub>(Type `id` for your uid).</sub></td>
  </tr>
  <tr>
   <td><sub>`NEW_GID`</sub></td>
   <td>int</td>
   <td>`1000`</td>
   <td>Assign the PHP-FPM group a new `gid` in order to syncronize file system permissions with your host computer and the Docker container. You should use a value that matches your host systems local group.<br/><sub>(Type `id` for your gid).</sub></td>
  </tr>
  <tr>
   <td colspan="5"></td>
  </tr>
  <tr>
   <td rowspan="4">**prod**<br/><br/>**work**</td>
   <td><sub>`TIMEZONE`</sub></td>
   <td>string</td>
   <td>`UTC`</td>
   <td>Set docker OS timezone as well as PHP timezone.<br/>(Example: `Europe/Berlin`)</td>
  </tr>
  <tr>
   <td><sub>`DOCKER_LOGS`</sub></td>
   <td>bool</td>
   <td>`1`</td>
   <td>By default all Docker images are configured to output their PHP-FPM access and error logs to stdout and stderr. Those which support it can change the behaviour to log into files inside the container. Their respective directories are available as volumes that can be mounted to the host computer. This feature might help developer who are more comfortable with tailing or searching through actual files instead of using docker logs.<br/><br/>Set this variable to `0` in order to enable logging to files. Log files are avilable under `/var/log/php/` which is also a docker volume that can be mounted locally.</td>
  </tr>
  <tr>
   <td><sub>`ENABLE_MAIL`</sub></td>
   <td>bool</td>
   <td>`0`</td>
   <td>Enable local email catch-all.<br/>Postfix will be configured for local delivery and all mails sent (even to real domains) will be catched locally. No email will ever go out. They will all be stored in a local devilbox account.<br/>Value: `0` or `1`</td>
  </tr>
  <tr>
   <td><sub>`FORWARD_PORTS_TO_LOCALHOST`</sub></td>
   <td>string</td>
   <td></td>
   <td>List of remote ports to forward to 127.0.0.1.<br/>**Format:**<br/><sub>`<local-port>:<remote-host>:<remote-port>`</sub><br/>You can separate multiple entries by comma.<br/>**Example:**<br/><sub>`3306:mysqlhost:3306, 6379:192.0.1.1:6379`</sub></td>
  </tr>
 </tbody>
</table>

#### Volumes

Have a look at the following table to see all offered volumes for each Docker image flavour.

<table>
 <thead>
  <tr>
   <th>Image</th>
   <th width="200">Volumes</th>
   <th>Description</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td rowspan="4">**prod**<br/><br/>**work**</td>
   <td><sub>`/etc/php-custom.d`</sub></td>
   <td>Mount this directory into your host computer and add custom `*.ini` files in order to alter php behaviour.</td>
  </tr>
  <tr>
   <td><sub>`/etc/php-modules.d`</sub></td>
   <td>Mount this directory into your host computer and add custo `*.so` files in order to add your php modules.<br/><br/>**Note:**Your should then also provide a custom `*.ini` file in order to actually load your custom provided module.</td>
  </tr>
  <tr>
   <td><sub>`/var/log/php`</sub></td>
   <td>When setting environment variable `DOCKER_LOGS` to `0`, log files will be available under this directory.</td>
  </tr>
  <tr>
   <td><sub>`/var/mail`</sub></td>
   <td>Emails caught be the postfix catch-all (`ENABLE_MAIL=1`) will be available in this directory.</td>
  </tr>
 </tbody>
</table>


#### Ports

Have a look at the following table to see all offered exposed ports for each Docker image flavour.

<table>
 <thead>
  <tr>
   <th>Image</th>
   <th width="200">Port</th>
   <th>Description</th>
  </tr>
 </thead>
 <tbody>
  <tr>
   <td rowspan="1">**base**<br/>**mods**<br/>**prod**<br/>**work**</td>
   <td><sub>`9000`</sub></td>
   <td>PHP-FPM listening port</td>
  </tr>
 </tbody>
</table>


<h2><img id="examples" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> Examples</h2>

#### Provide PHP-FPM port to host
```
$ docker run -i \
    -p 127.0.0.1:9000:9000 \
    -t devilbox/php-fpm-7.3	
```

#### Custom PHP configuration
```
# Create config directory to be mounted with dummy configuration
$ mkdir config
$ echo "xdebug.enable = 1" > config/xdebug.ini

# Run container and mount it
$ docker run -i \
    -p 127.0.0.1:9000:9000 \
	-v config:/etc/php-custom.d \
    -t devilbox/php-fpm-7.3	
```

#### Custom PHP modules
```
# Create module directory and place module into it
$ mkdir modules
$ cp /my/module/phalcon.so modules/

# Custom php config to load this module
$ mkdir config
$ echo "extension=phalcon.so" > config/phalcon.ini

# Run container and mount it
$ docker run -i \
    -p 127.0.0.1:9000:9000 \
	-v config:/etc/php-custom.d \
	-v modules:/etc/php-modules.d \
    -t devilbox/php-fpm-7.3	
```

#### Webserver and PHP-FPM
```
# Start myself
$ docker run -d \
    -v ~/my-host-www:/var/www/default/htdocs \
    --name php \
    -t devilbox/php-fpm-7.3

# Start webserver and link into myself
$ docker run -d \
    -p 80:80 \
    -v ~/my-host-www:/var/www/default/htdocs \
    -e PHP_FPM_ENABLE=1 \
    -e PHP_FPM_SERVER_ADDR=php \
    -e PHP_FPM_SERVER_PORT=9000 \
    --link php \
    -t devilbox/nginx-mainline
```


<h2><img id="automated-builds" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> Automated builds</h2>

[![Build Status](https://travis-ci.org/devilbox/docker-php-fpm-7.3.svg?branch=master)](https://travis-ci.org/devilbox/docker-php-fpm-7.3)

Docker images are built and tested every night by **[travis-ci](https://travis-ci.org/devilbox/docker-php-fpm-7.3)** and pushed to **[Docker hub](https://hub.docker.com/r/devilbox/php-fpm-7.3/)** on success. This is all done automatically to ensure that sources as well as base images are always fresh and in case of security updates always have the latest patches.

<h2><img id="contributing" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> Contributing</h2>

Contributors are welcome. Feel free to star and clone this repository and submit issues and pull-requests. Add examples and show what you have created with the provided images.

<h2><img id="credits" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> Credits</h2>

None yet.

<h2><img id="license" width="20" src="https://github.com/devilbox/artwork/raw/master/submissions_logo/cytopia/01/png/logo_64_trans.png"> License</h2>

**[MIT License](LICENSE.md)**

Copyright (c) 2017 [cytopia](https://github.com/cytopia)

