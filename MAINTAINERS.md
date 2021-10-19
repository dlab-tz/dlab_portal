# Maintainers Guide

Maintainers guide for [dLab open data portal](https://github.com/dlab-tz/dlab_portal)


## Requirements

Installed version of CKAN either in [development](https://github.com/dlab-tz/dlab_portal/blob/master/DEVELOP.md) or [production](https://github.com/dlab-tz/dlab_portal/blob/master/PRODUCTION.md).


## Creating Administrator User

- Activate CKAN virtual environment and navigate to CKAN source
```sh
$ pyenv shell 3.8.0
$ . /usr/lib/ckan/default/bin/activate
$ cd /usr/lib/ckan/default/src/ckan
```

- Create a new administrator
```sh
ckan -c /etc/ckan/default/ckan.ini sysadmin add dlab_ckan_default email=info@dlab.or.tz name=dlab_ckan_default
```
Accept to create a new user by press `Y`, then press `Enter`, then enter a password on the prompt.


**Note:**
A template for creating administrator users is:
```sh
ckan -c /etc/ckan/default/ckan.ini sysadmin add <username> email=<email> name=<name>
```


## Creating Test Data

During `development` and `testing` phase, test data allow for quick checks if everything works.

- Loading test data into database

```sh
ckan -c /etc/ckan/default/ckan.ini seed basic
ckan -c /etc/ckan/default/ckan.ini seed family
ckan -c /etc/ckan/default/ckan.ini seed gov
ckan -c /etc/ckan/default/ckan.ini seed hierarchy
ckan -c /etc/ckan/default/ckan.ini seed search
ckan -c /etc/ckan/default/ckan.ini seed translations
ckan -c /etc/ckan/default/ckan.ini seed user
ckan -c /etc/ckan/default/ckan.ini seed vocabs
```

**Note:**
Once development is through, remember to clean the database and re-initialize it for production usage.


## Cache Store, FileStore and File Uploads

- Create a directory where CKAN will store uploaded files
```sh
$ sudo mkdir -p /var/lib/ckan/default
```

- Edit CKAN config file to set `cache_dir` and `storage_path`
```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

```ini
cache_dir = /var/lib/ckan/default/data/
ckan.storage_path = /var/lib/ckan/default
```

- Set permissions to allow Ngix's user to read, write and execute
```sh
$ sudo chown -R www-data /var/lib/ckan/default
$ sudo chmod -R u+rwx /var/lib/ckan/default
$ sudo chown -R `whoami` /var/lib/ckan/default
```

- Then restart
```sh
$ sudo systemctl restart supervisor
```

or

```sh
$ ckan -c /etc/ckan/default/ckan.ini run
```

## Page View Tracking

- Edit CKAN config file to set `tracking_enabled = true` in the `[app:main]`
```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

```ini
ckan.tracking_enabled = true
```

- Then restart
```sh
$ sudo systemctl restart supervisor
```

or

```sh
$ ckan -c /etc/ckan/default/ckan.ini run
```
