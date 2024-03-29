# Developer Guide

Setup guide for [dLab open data portal](https://github.com/dlab-tz/dlab_portal) development

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


### Install CKAN

- Create required symlink and directories for your home workspace/working directories

```sh
$ mkdir -p ~/workspace/ckan/lib
$ sudo ln -s ~/workspace/ckan/lib /usr/lib/ckan
$ mkdir -p ~/workspace/ckan/etc
$ sudo ln -s ~/workspace/ckan/etc /etc/ckan
$ mkdir -p ~/workspace/ckan/var/lib
$ sudo ln -s ~/workspace/ckan/var/lib /var/lib/ckan
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
ckan.site_url = http://127.0.0.1:5000
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

- Now restart Tomcat & Solr and check Solr running `[http://localhost:8983/solr/](http://localhost:8983/solr/)`

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


### Run CKAN

To run CKAN in development and testing use:

```sh
$ . /usr/lib/ckan/default/bin/activate
$ cd /usr/lib/ckan/default/src/ckan
$ ckan -c /etc/ckan/default/ckan.ini run
```

or

```sh
ckan -c /etc/ckan/default/ckan.ini run --host 0.0.0.0 -p 5000
```

Open [http://127.0.0.1:5000/](http://127.0.0.1:5000/)


Check [maintainers guide](https://github.com/dlab-tz/dlab_portal/blob/master/MAINTAINERS.md) for further configurations and management tasks.
