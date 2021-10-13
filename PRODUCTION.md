# Production Guide

Setup guide for [dLab open data portal](https://github.com/dlab-tz/dlab_portal) production deployment

## Requirements

### Software Requirements

- Ubuntu Server 18.04 64-bit
- Python v3.6+
- Java OpenJDK v11+
- PostgreSQL v9.5+
- Git   v2.17+
- Apache Solr
- Jetty/Tomcat
- Redis
- Nginx

### Ports Requirements

| Service   | Port  | Used for  |
| :-------- | :---: | :-------- |
|NGINX      | 80    | Proxy     |
|uWSGI      | 8080  | Web Server|
|uWSGI      | 8800  | DataPusher|
|Solr/Jetty | 8983  | Search    |
|PostgreSQL | 5432  | Database  |
|Redis      | 6379  | Search    |


## Installation

### Update Ubuntu Server

- Ensure latest Ubuntu updates
```sh
sudo su <<EOF
sudo apt-get autoclean -y
sudo apt-get autoremove -y
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get dist-upgrade -y
EOF
```

- Install common required dependencies

```sh
sudo su <<EOF
sudo apt-get update -y
sudo apt-get install -y curl wget make g++ git
sudo apt-get install -y libkrb5-dev build-essential libgeos-dev proj-bin
sudo apt-get install -y libpq5 libpq-dev supervisor
sudo apt-get install -y python-dev libxslt1-dev libxml2-dev zlib1g-dev libffi-dev
EOF
```

### Install Python

- Install [pyenv](https://github.com/pyenv/pyenv) using [pyenv-installer](https://github.com/pyenv/pyenv-installer)

```sh
$ curl https://pyenv.run | bash
```

- Install latest Python
```sh
$ pyenv install -v 3.8.0
```

- Install required Python dependencies
```sh
sudo su <<EOF
sudo apt-get update -y
sudo apt-get install -y python3-dev python3-pip python3-venv
EOF
```

### Install Nginx Web Server
```sh
sudo su <<EOF
sudo apt-get update -y
sudo apt-get install nginx -y
sudo systemctl enable nginx
sudo systemctl start nginx
EOF
```

### Install Redis

```sh
sudo su <<EOF
sudo add-apt-repository ppa:chris-lea/redis-server
sudo apt-get update -y
sudo apt-get upgrade -y
sudo apt-get install -y redis-server
sudo systemctl start redis-server
EOF
```

### Install PostgreSQL

```sh
sudo su <<EOF
sudo apt update -y
sudo apt install -y postgresql postgresql-contrib
sudo systemctl start postgresql
EOF
```

### Install OpenJDK (Java 11)

- Install default JDK and JRE

```sh
sudo su <<EOF
sudo apt update -y
sudo apt install -y default-jdk
sudo apt install -y default-jre
EOF
```

- Set the default Java version to `java-11-openjdk`

```sh
$ sudo update-alternatives --config java
```

- Set the default Javac version to `java-11-openjdk`

```sh
$ sudo update-alternatives --config javac
```

- Set the `JAVA_HOME` environment variable to `java-11-openjdk`

```sh
$ sudo update-alternatives --config java
```

Copy printed installation paths and edit `/etc/environment` and set `JAVA_HOME`
but do not include the /bin portion of the path.

```sh
$ sudo vi /etc/environment
```

> JAVA_HOME="copied path without /bin"

Apply changes and verify `JAVA_HOME`

```sh
$ source /etc/environment
```

```sh
$ echo $JAVA_HOME
```

### Install Apache Solr and Tomcat(Jetty)

```sh
sudo su <<EOF
sudo apt update -y
sudo apt install -y solr-tomcat
EOF
```

### Install CKAN

- Create required symlink and directories for your server home directories

```sh
$ mkdir -p ~/ckan/lib
$ sudo ln -s ~/ckan/lib /usr/lib/ckan
$ mkdir -p ~/ckan/etc
$ sudo ln -s ~/ckan/etc /etc/ckan
```

- Create a Python(`python3`) virtual environment (virtualenv) to install CKAN into, and activate it.

```sh
$ sudo mkdir -p /usr/lib/ckan/default
$ sudo chown `whoami` /usr/lib/ckan/default
$ python3 -m venv /usr/lib/ckan/default
$ . /usr/lib/ckan/default/bin/activate
```

- Install setup tools and upgrade pip

```sh
$ pip install setuptools==44.1.0
$ pip install -U pip
```

- Install the CKAN source code into your virtualenv

```sh
$ pip install -e 'git+https://github.com/ckan/ckan.git@ckan-2.9.4#egg=ckan[requirements]'
```

- Deactivate and re-activate your virtual environment
```sh
deactivate
. /usr/lib/ckan/default/bin/activate
```


## Configuration

### Configure PostgreSQL

- Create a new PostgreSQL user called `ckan_default`

```sh
$ sudo -u postgres createuser -S -D -R -P ckan_default
```

- Create a new PostgreSQL database, called ckan_default

```sh
$ sudo -u postgres createdb -O ckan_default ckan_default -E utf-8
```

### Configure CKAN

- Create directory to contain CKAN site's configuration file

```sh
$ sudo mkdir -p /etc/ckan/default
$ sudo chown -R `whoami` /etc/ckan/
```

- Generate CKAN configuration file

```sh
$ ckan generate config /etc/ckan/default/ckan.ini
```

- Edit `ckan.ini` to include the following configurations

```ini
sqlalchemy.url = postgresql://ckan_default:<ckan_default_pass>@localhost/ckan_default
ckan.site_id = default
ckan.site_url = http://127.0.0.1:5000 (or http://<sub-domain>.<domain>.<com|org>)
```

### Configure Apache Solr

- Change the default port Tomcat runs on (8080) to the one expected by CKAN(8983)

```sh
$ sudo cp /etc/tomcat9/server.xml /etc/tomcat9/server.xml.bak
$ sudo vi /etc/tomcat9/server.xml
```

```xml
From:

<Connector port="8080" protocol="HTTP/1.1"


To:

<Connector port="8983" protocol="HTTP/1.1"
```

- Replace the default schema.xml file with a symlink to the CKAN schema file

```sh
$ sudo mv /etc/solr/conf/schema.xml /etc/solr/conf/schema.xml.bak
$ sudo ln -s /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml /etc/solr/conf/schema.xml
```

- Now restart Tomcat & Solr

```sh
$ sudo mv /etc/systemd/system/tomcat9.d /etc/systemd/system/tomcat9.service.d
$ sudo systemctl daemon-reload
$ sudo systemctl restart tomcat9
```

- Change the solr_url setting in your CKAN configuration file

```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

```ini
solr_url=http://127.0.0.1:8983/solr
```

### Configure who.ini

```sh
$ ln -s /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini
```

### Create CKAN database tables

```sh
$ cd /usr/lib/ckan/default/src/ckan
$ ckan -c /etc/ckan/default/ckan.ini db init
```

### Set up the DataStore

- Enable the plugin

```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

```ini
ckan.plugins = stats text_view image_view recline_view datastore
```

- Create user

```sh
$ sudo -u postgres createuser -S -D -R -P -l datastore_default
$ sudo -u postgres createdb -O ckan_default datastore_default -E utf-8
```

- Set datastore URLs
```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

```ini
ckan.datastore.write_url = postgresql://ckan_default:<ckan_default_pass>@localhost/datastore_default
ckan.datastore.read_url = postgresql://datastore_default:<datastore_default_pass>@localhost/datastore_default
```

- Set permissions

```sh
$ sudo -u postgres psql
$ ckan -c /etc/ckan/default/ckan.ini datastore set-permissions | sudo -u postgres psql --set ON_ERROR_STOP=1
```

If failed, copy output of

```sh
$ ckan -c /etc/ckan/default/ckan.ini datastore set-permissions
```

- Restart CKAN to Test and verify

```sh
$ cd /usr/lib/ckan/default/src/ckan
$ ckan -c /etc/ckan/default/ckan.ini run
$ curl -X GET "http://127.0.0.1:5000/api/3/action/datastore_search?resource_id=_table_metadata"
```

### DataPusher Configuration (TODO)

```ini
ckan.plugins = <other plugins> datapusher
ckan.datapusher.url = http://127.0.0.1:8800/
```

### Production Deployment

- Deactivate and re-activate CKAN Python virtual environment

```sh
$ deactivate
$ . /usr/lib/ckan/default/bin/activate
```

- Create the WSGI script file

```sh
sudo cp /usr/lib/ckan/default/src/ckan/wsgi.py /etc/ckan/default/
```

- Install and Create the WSGI Server

```sh
$ pip install -U uwsgi
```

```sh
$ sudo cp /usr/lib/ckan/default/src/ckan/ckan-uwsgi.ini /etc/ckan/default/
```

- Ensure Supervisor for the uwsgi

```sh
sudo su <<EOF
sudo apt-get install -y supervisor
sudo systemctl restart supervisor
EOF
```

- Create uwsgi supervisor configuration file
```sh
$ sudo vi /etc/supervisor/conf.d/ckan-uwsgi.conf
```

```conf
[program:ckan-uwsgi]

command=/usr/lib/ckan/default/bin/uwsgi -i /etc/ckan/default/ckan-uwsgi.ini

; Start just a single worker. Increase this number if you have many or
; particularly long running background jobs.
numprocs=1
process_name=%(program_name)s-%(process_num)02d

; Log files - change this to point to the existing CKAN log files
stdout_logfile=/etc/ckan/default/uwsgi.OUT
stderr_logfile=/etc/ckan/default/uwsgi.ERR

; Make sure that the worker is started on system start and automatically
; restarted if it crashes unexpectedly.
autostart=true
autorestart=true

; Number of seconds the process has to run before it is considered to have
; started successfully.
startsecs=10

; Need to wait for currently executing tasks to finish at shutdown.
; Increase this if you have very long running tasks.
stopwaitsecs = 600

; Required for uWSGI as it does not obey SIGTERM.
stopsignal=QUIT
```

- Create the NGINX config file

```sh
$ sudo vi /etc/nginx/sites-available/ckan
```

```nginx
proxy_cache_path /tmp/nginx_cache levels=1:2 keys_zone=cache:30m max_size=250m;
proxy_temp_path /tmp/nginx_proxy 1 2;

server {
    client_max_body_size 100M;
    location / {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
        proxy_cache cache;
        proxy_cache_bypass $cookie_auth_tkt;
        proxy_no_cache $cookie_auth_tkt;
        proxy_cache_valid 30m;
        proxy_cache_key $host$scheme$proxy_host$request_uri;
        # In emergency comment out line to force caching
        # proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
    }

}
```

- Disable your default nginx sites and restart

```sh
$ sudo rm -vi /etc/nginx/sites-enabled/default
$ sudo ln -s /etc/nginx/sites-available/ckan /etc/nginx/sites-enabled/ckan
$ sudo service restart nginx
```

### Setup a worker for background jobs

- Copy the configuration file template
```sh
$ sudo cp /usr/lib/ckan/default/src/ckan/ckan/config/supervisor-ckan-worker.conf /etc/supervisor/conf.d
```

- Restart Supervisor

```sh
$ sudo service restart supervisor
```

- Restart the CKAN worker via

```sh
$ sudo supervisorctl restart ckan-worker:*
```

- Test that background jobs are processed correctly

```sh
$ ckan -c |ckan.ini| jobs test
```
