# Production Guide

Setup guide for [dLab open data portal](https://github.com/dlab-tz/dlab_portal) production deployment


## Requirements


### Software Requirements

| Software      | Version         |
|:--------------|:----------------|
| Ubuntu Server | v18.04+, 64-bit |
| Nginx         | v1.14+          |
| PostgreSQL    | v9.5+           |
| Redis         | v6+             |
| Python        | v3.8+           |
| Java OpenJDK  | v11+            |
| Tomcat        | v9+             |
| Apache Solr   | v3.6.2+         |


### Ports Requirements

| Service     | Port  | Used for   |
| :---------- | :---: | :--------- |
| NGINX       | 80    | Proxy      |
| uWSGI       | 5000  | Web Server |
| uWSGI       | 8800  | DataPusher |
| Solr/Tomcat | 8983  | Search     |
| PostgreSQL  | 5432  | Database   |
| Redis       | 6379  | Search     |


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
sudo apt-get install -y libpq5 libpq-dev libsqlite3-dev libbz2-dev libreadline-dev
sudo apt-get install -y libxslt1-dev libxml2-dev zlib1g-dev libffi-dev
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

- Configure `.profile` for pyenv
```sh
sudo vi ~/.profile
```

Then, add below snippets after initial comment lines
```sh
# Setup Python Version Manager (pyenv)
export PYENV_ROOT="$HOME/.pyenv"
export PATH="$PYENV_ROOT/bin:$PATH"
eval "$(pyenv init --path)"
```

- Configure `.bashrc` for pyenv
```sh
$ sudo vi ~/.bashrc
```

Then, add below snippets at the end of the `.bashrc` file.
```bash
# Setup Python Version Manager (pyenv)
eval "$(pyenv init -)"
eval "$(pyenv virtualenv-init -)"
```

- Restart your shell so the path changes take effect
```sh
exec $SHELL
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

- Install latest postgresql
```sh
sudo su <<EOF
sudo apt-get update -y
sudo apt-get install -y postgresql postgresql-contrib
sudo systemctl start postgresql
EOF
```

- Enable administrative login by UNIX domain socket

```sh
$ sudo vi
```

Change,
```ini
# Database administrative login by Unix domain socket
local   all             postgres                                peer
```

to

```ini
# Database administrative login by Unix domain socket
local   all             postgres                                trust
```

- Restart postgresql
```sh
$ sudo systemctl restart postgresql
```


### Install OpenJDK (Java 11)

- Install default JDK and JRE

```sh
sudo su <<EOF
sudo apt-get update -y
sudo apt-get install -y default-jdk
sudo apt-get install -y default-jre
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

```ini
JAVA_HOME="/usr/lib/jvm/java-11-openjdk-amd64"
```

- Apply changes

```sh
$ source /etc/environment
```

- Restart your shell so the path changes take effect
```sh
exec $SHELL
```

- Verify `JAVA_HOME`
```sh
$ echo $JAVA_HOME
```


### Install Apache Solr and Tomcat(Jetty)

```sh
sudo su <<EOF
sudo apt-get update -y
sudo apt-get install -y solr-tomcat
sudo mv /etc/systemd/system/tomcat9.d /etc/systemd/system/tomcat9.service.d
sudo systemctl daemon-reload
sudo systemctl start tomcat9
EOF
```


### Install Supervisor

- Remove old system installed supervisor
```sh
sudo su <<EOF
sudo apt-get remove -y supervisor
sudo apt-get purge -y supervisor
sudo apt-get autoclean -y
sudo apt-get autoremove -y
sudo apt-get update -y
EOF
```

- Ensure common directories for supervisor
```sh
$ sudo mkdir -p /var/log/supervisor
$ sudo mkdir -p /etc/supervisor
$ sudo mkdir -p /etc/supervisor/conf.d
```

- Create supervisor config
```sh
$ sudo vi /etc/supervisor/supervisor.conf
```

```ini
; supervisor config file

[unix_http_server]
file=/var/run/supervisor.sock   ; (the path to the socket file)
chmod=0700                       ; sockef file mode (default 0700)

[supervisord]
logfile=/var/log/supervisor/supervisord.log ; (main log file;default $CWD/supervisord.log)
pidfile=/var/run/supervisord.pid ; (supervisord pidfile;default supervisord.pid)
childlogdir=/var/log/supervisor            ; ('AUTO' child log dir, default $TEMP)

; the below section must remain in the config file for RPC
; (supervisorctl/web interface) to work, additional interfaces may be
; added by defining them in separate rpcinterface: sections
[rpcinterface:supervisor]
supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

[supervisorctl]
serverurl=unix:///var/run/supervisor.sock ; use a unix:// URL  for a unix socket

; The [include] section can just contain the "files" setting.  This
; setting can list multiple files (separated by whitespace or
; newlines).  It can also contain wildcards.  The filenames are
; interpreted as relative to this file.  Included files *cannot*
; include files themselves.

[include]
files = /etc/supervisor/conf.d/*.conf
```

- Activate python 3.8.0 in the shell
```sh
$ pyenv shell 3.8.0
```

- Install supervisor using `pip`
```sh
$ pip install supervisor
```

- Create supervisor service
```sh
$ sudo vi /etc/systemd/system/supervisor.service
```

```ini
[Unit]
Description=Supervisor process control system for UNIX
Documentation=http://supervisord.org
After=network.target

[Service]
ExecStart=/home/<user>/.pyenv/versions/3.8.0/bin/supervisord -n -c /etc/supervisor/supervisord.conf
ExecStop=/home/<user>/.pyenv/versions/3.8.0/bin/supervisorctl $OPTIONS shutdown
ExecReload=/home/<user>/.pyenv/versions/3.8.0/bin/supervisorctl -c /etc/supervisor/supervisord.conf $OPTIONS reload
KillMode=process
Restart=on-failure
RestartSec=50s

[Install]
WantedBy=multi-user.target
Alias=supervisord.service
```

**Note:**
Replace user with actual $USER

- Enable supervisor service
```sh
sudo su <<EOF
sudo systemctl daemon-reload
sudo systemctl enable supervisor
sudo systemctl start supervisor
EOF
```


### Install CKAN

- Create required symlink and directories for your server home directories

```sh
$ mkdir -p ~/ckan/lib
$ sudo ln -fs ~/ckan/lib /usr/lib/ckan
$ mkdir -p ~/ckan/etc
$ sudo ln -fs ~/ckan/etc /etc/ckan
$ mkdir -p ~/ckan/var/lib
$ sudo ln -fs ~/ckan/var/lib /var/lib/ckan
```

- Create a Python(`python3`) virtual environment (virtualenv) to install CKAN into, and activate it.

```sh
$ sudo mkdir -p /usr/lib/ckan/default
$ sudo chown `whoami` /usr/lib/ckan/default
$ pyenv shell 3.8.0
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

- Create a new PostgreSQL user called `dlab_ckan_default`

```sh
$ sudo -u postgres createuser -S -D -R -P dlab_ckan_default
```

- Create a new PostgreSQL database, called dlab_ckan_default

```sh
$ sudo -u postgres createdb -O dlab_ckan_default dlab_ckan_default -E utf-8
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

```sh
sudo vi /etc/ckan/default/ckan.ini
```

```ini
sqlalchemy.url = postgresql://dlab_ckan_default:<dlab_ckan_default_pass>@localhost/dlab_ckan_default
ckan.site_id = default
ckan.site_url = http://<public_ip>:5000
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

- Create user and database

```sh
$ sudo -u postgres createuser -S -D -R -P -l dlab_datastore_default
$ sudo -u postgres createdb -O dlab_ckan_default dlab_datastore_default -E utf-8
```

- Set datastore URLs
```sh
$ sudo vi /etc/ckan/default/ckan.ini
```

```ini
ckan.datastore.write_url = postgresql://dlab_ckan_default:<dlab_ckan_default_pass>@localhost/dlab_datastore_default
ckan.datastore.read_url = postgresql://dlab_datastore_default:<dlab_datastore_default_pass>@localhost/dlab_datastore_default
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


### Setup DataPusher

> TODO


### Production Deployment

- Deactivate and re-activate CKAN Python virtual environment

```sh
$ pyenv shell 3.8.0
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

- Change `ckan-uwsgi` port to `5000`
```sh
$ sudo vi /etc/ckan/default/ckan-uwsgi.ini
```

```ini
http            =  0.0.0.0:5000
```

- Create uwsgi supervisor configuration file
```sh
$ sudo vi /etc/supervisor/conf.d/ckan-uwsgi.conf
```

```ini
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

- Allow `5000` from firewall
```sh
$ sudo ufw allow 5000/tcp
```

- Restart Supervisor

```sh
$ sudo systemctl restart supervisor
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
        proxy_pass http://127.0.0.1:5000/;
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

- Enable CKAN sites and restart

```sh
$ sudo ln -s /etc/nginx/sites-available/ckan /etc/nginx/sites-enabled/ckan
$ sudo nginx -t
$ sudo systemctl restart nginx
```

Check [maintainers guide](https://github.com/dlab-tz/dlab_portal/blob/master/MAINTAINERS.md) for further configurations and management tasks.
