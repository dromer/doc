# App packaging <img src="https://yunohost.org/images/yunohost_package.png" width=100/>

This document aimed to learn you how to package an application for YunoHost.

### Requirements
To package an application, here are the requirements:
* An account on a git server as [GitHub](https://github.com/) to publish the application;
* Control a minimum `git`, Shell and other programming stuffs;
* A testing [virtual machine or a distant server](/install_en) to package and test the package.

### Content
A YunoHost package is composed of:

* A `manifest.json` file
* A `scripts` directory, which contains five Shell scripts: `install`, `remove`, `upgrade`, `backup` and `restore`
* Optional directories, containing `sources` or `conf` files
* A `LICENSE` file containing the license of the package
* A presentation page of your package in a `README.md` file

**[A basic package](https://github.com/YunoHost/example_ynh)**: feel free to use it as a framework.

## Manifest
<a class="btn btn-lg btn-default" href="packaging_apps_manifest_en">Manifest</a>

## Scripts
For now, a YunoHost package must contain five Shell scripts: `install`, `remove`, `upgrade`, `backup` and `restore`.
These scripts will be executed as `admin` on the YunoHost instances.

Here is an example:
```bash
# Retrieve arguments
domain=$1
path=$2

# Check domain/path availability
sudo yunohost app checkurl $domain$path -a roundcube
if [[ ! $? -eq 0 ]]; then
    exit 1
fi

# Generate random DES key & password
deskey=$(dd if=/dev/urandom bs=1 count=200 2> /dev/null | tr -c -d '[A-Za-z0-9]' | sed -n 's/\(.\{24\}\).*/\1/p')
db_pwd=$(dd if=/dev/urandom bs=1 count=200 2> /dev/null | tr -c -d '[A-Za-z0-9]' | sed -n 's/\(.\{24\}\).*/\1/p')

# Use 'roundcube' as database name and user
db_user=roundcube

# Initialize database and store mysql password for upgrade
sudo yunohost app initdb $db_user -p $db_pwd -s $(readlink -e ../sources/SQL/mysql.initial.sql)
sudo yunohost app setting roundcube mysqlpwd -v $db_pwd

# Copy files to the right place
final_path=/var/www/roundcube
sudo mkdir -p $final_path
sudo cp -a ../sources/* $final_path
sudo cp ../conf/main.inc.php $final_path/config/
sudo cp ../conf/db.inc.php $final_path/config/
sudo mv $final_path/plugins/managesieve/config.inc.php.dist $final_path/plugins/managesieve/config.inc.php

# Change variables in Roundcube configuration
sudo sed -i "s/rcmail-ynhDESkeyTOchange/$deskey/g" $final_path/config/main.inc.php
sudo sed -i "s/yunouser/$db_user/g" $final_path/config/db.inc.php
sudo sed -i "s/yunopass/$db_pwd/g" $final_path/config/db.inc.php
sudo sed -i "s/yunobase/$db_user/g" $final_path/config/db.inc.php

# Set permissions to roundcube directory
sudo chown -R www-data: $final_path

# Modify Nginx configuration file and copy it to Nginx conf directory
sed -i "s@PATHTOCHANGE@$path@g" ../conf/nginx.conf
sed -i "s@ALIASTOCHANGE@$final_path/@g" ../conf/nginx.conf
sudo cp ../conf/nginx.conf /etc/nginx/conf.d/$domain.d/roundcube.conf

# Reload nginx and regenerate SSOwat conf
sudo service nginx reload
sudo yunohost app ssowatconf
```

### Usage
You have to put everything in the `install` script in order to get the app to install without issue. It means that you have to install dependencies, create required repositories, initialize potential databases, copy sources and configure everything in the single `install` script (and of course do the reverse process in the `remove` script).

**Be careful**: for security reasons, the script is executed as the **admin** user in YunoHost. Be sure to test it as **admin** and prepend `sudo` to commands that require it.

### Architecture and arguments
Since YunoHost has a unified architecture, you will be able to guess most of the settings you need. But if you need variable ones, like the domain or web path, you will have to ask the administrator at installation (see `arguments` section in the manifest above).

<a class="btn btn-lg btn-default" href="packaging_apps_arguments_management_en">Arguments management</a>

### Nginx configuration
<a class="btn btn-lg btn-default" href="packaging_apps_nginx_conf_en">Nginx configuration</a>

### Hooks
YunoHost provides a hook system, which is accessible via the packager's script callbacks in moulinette (CLI).
The scripts have to be placed in the `hooks` repository at the root of the YunoHost package, and must be named `priority-hook_name`, for example: `hooks/50-post_user_create` will be executed after each user creation.

**Note**: `priority` is optional, default is `50`.

Take a look at the [ownCloud package](https://github.com/Kloadut/owncloud_ynh) for a working example.

### Helpers
The CLI [moulinette](/moulinette) provides a few tools to make the packager's work easier:

<br>

```bash
sudo yunohost app checkport <port>
```
<blockquote>
This helper checks the port and returns an error if the port is already in use.
</blockquote>

<br>

```bash
sudo yunohost app setting <id> <key> [ -v <value> ]
```
<blockquote>
This is the most important helper of YunoHost. It allows you to store some settings for a specific app, in order to be either reused afterward or used for YunoHost configuration (**e.g.** for the SSO).
<br><br>
It sets the value if you append ```-v <value>```, and gets it otherwise.
<br><br>

** Some useful settings **<br><br>
```skipped_uris```<br><br>
Remove the protection on the uris list provided separated by commas.<br><br>

```protected_uris```<br><br>
Protects the uris list provided separated by commas. Only logged in users will have access.<br><br>

There are also `skipped_regex`, `protected_regex`, `unprotected_uris`, `unprotected_regex`.<br><br>

**Be careful** : you must run `yunohost app ssowatconf` to apply the effect. URIs will be converted into URLs and written to the file /etc/ssowat/conf.json.<br><br>

Example:<br>
```yunohost app setting myapp unprotected_urls -v "/"```<br>
```yunohost app ssowatconf```<br>
These commands will disable the SSO on the root of the aplication like domain.tld/myapp This is useful for public application.
</blockquote>

<br>

```bash
sudo yunohost app checkurl <domain><path> -a <id>
```
<blockquote>
This helper is useful for web apps and allows you to be sure that the web path is not taken by another app. If not, it "reserves" the path.
<br><br>
**Note**: do not prepend `http://` or `https://` to the `<domain><path>`.
</blockquote>

<br>

```bash
sudo yunohost app initdb <db_user> [ -p <db_pwd> ] [ -s <SQL_file> ]
```
<blockquote>
This helper creates a MySQL database. If you do not append a password, it generates one and returns it. If you append a SQL file, it initializes your database with the SQL statements inside.
</blockquote>

<br>

```bash
sudo yunohost app ssowatconf
```
<blockquote>
This helper reloads the SSO configuration. You have to call it at the end of the script when you are packaging a web app.
</blockquote>

### Test it!
In order to test your package, you can execute your script standalone as `admin` (do not forget to append required arguments):
```bash
su - admin -c "/bin/bash /path/to/my/script my_arg1 my_arg2"
```

Or you can use [moulinette](/moulinette_en):
```bash
yunohost app install /path/to/my/app/package
```
Note that it also works with a Git URL:
```bash
yunohost app install https://github.com/author/my_app_package.git
```

### Enhance package
You will find points to verify quality of your scripts:
* scripts should use `sudo cp -a ../sources/. $final_path` instead of `sudo cp -a ../sources/* $final_path`;
* install script must contain support in case of script errors to delete residuals files thanks to `set -e` and `trap`;
* install script should use command line method instead of curl call through web install form;
* install script should save install answers;
* application sources should be checked with a control sum (sha256, sha1 or md5) or a PGP signature;
* scripts had been tested on Debian Wheezy and Jessie as well as 32 bits, 64 bits and ARM architectures;
* backup and restore scripts are present and functional.

### Publish and ask for testing your application
* Publishing a [post on the Forum](https://forum.yunohost.org/) with the [`App integration` category](https://forum.yunohost.org/c/app-integration), to ask tests and returns on your application.

* Ask to add your application in the [app repository](https://github.com/YunoHost/apps) to be displayed in the [non-official apps list](https://yunohost.org/#/apps_in_progress_en). Precise his progress state: `notworking`, `inprogress`, or `working`.

### Officalization of an application
To become an official application, it must be enough tested, stable and should works on 64 bits, 32 bits et ARM processors architectures and on Debian Wheezy and Jessie. If you think thoses conditions are gather, ask for [official integration](https://github.com/YunoHost/apps) of your application.