# Maintainers Guide

Maintainers guide for [dLab open data portal](https://github.com/dlab-tz/dlab_portal)


## Requirements

Installed version of CKAN either in [development](https://github.com/dlab-tz/dlab_portal/blob/master/DEVELOP.md) or [production](https://github.com/dlab-tz/dlab_portal/blob/master/PRODUCTION.md).


## Creating Administrator User

- Activate CKAN virtual environment and navigate to CKAN source
```sh
$ . /usr/lib/ckan/default/bin/activate
$ cd /usr/lib/ckan/default/src/ckan
```

- Create a new administrator
```sh
ckan -c /etc/ckan/default/ckan.ini sysadmin add ckan_default email=ckan_default@localhost name=ckan_default
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


## Configuration Changes

All options that can be set on admin page can be edited in CKAN configuration file.

- Edit confgiuration file
```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

- Then restart
```sh
$ sudo supervisorctl restart ckan-uwsgi:*
```

or

```sh
ckan -c /etc/ckan/default/ckan.ini run
```

## FileStore and File Uploads

- Create a directory where CKAN will store uploaded files
```sh
$ sudo mkdir -p /var/lib/ckan/default
```

- Edit CKAN config file to set `storage_path`
```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

```ini
ckan.storage_path = /var/lib/ckan/default
```

- Set permissions to allow Ngix's user to read, write and execute
```sh
$ sudo chown www-data /var/lib/ckan/default
$ sudo chmod u+rwx /var/lib/ckan/default
$ sudo chown -R `whoami` /var/lib/ckan/default
```

- Then restart
```sh
$ sudo supervisorctl restart ckan-uwsgi:*
```

or

```sh
ckan -c /etc/ckan/default/ckan.ini run
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
$ sudo supervisorctl restart ckan-uwsgi:*
```

or

```sh
ckan -c /etc/ckan/default/ckan.ini run
