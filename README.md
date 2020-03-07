# generix_prototype
Prototype for the GeneriX project.

## Installation on Debian 10

Each step in the installation is described below.  If you already
have a prerequisite step installed, you can skip it.

### Build environment

_install package managers pip3 and npm:_

apt-get install python3-pip

apt-get install npm nodejs

_upgrade pip:_

pip3 install --upgrade pip

### Other Python prerequisites

pip3 install pandas pyArango dumper xarray

### Jupyterhub installation

_(based on https://jupyterhub.readthedocs.io/en/stable/quickstart.html)_

_install jupyter:_

pip3 install jupyter

_install jupyterhub and dependencies:_

pip3 install jupyterhub

npm install -g configurable-http-proxy

pip3 install notebook

_set up /etc/jupyterhub and /srv/jupyterhub files as described in docs above, or copy from another installation._

_make these files owned by jupyterhub, delete old sqlite and jupyterhub_cookie_secret_

_ssl certs need to be readable by jupyterhub user_


### Sudo spawner for Jupyterhub

useradd jupyterhub

_remember to set shell to nologin, add to shadow group_

pip3 install sudospawner

apt-get install sudo

_note: was getting "OSError: [Errno 99] Cannot assign requested address" on spawning jupyterhub_

_this is because ip 127.0.0.1 needs to be explicitly specified_

_had to create file /usr/local/bin/sudospawner-singleuser:_

```
#!/bin/bash -l
exec "/usr/local/bin/jupyterhub-singleuser" --ip 127.0.0.1 $@
```

_In /etc/sudoers:_

```
Cmnd_Alias JUPYTER_CMD = /usr/local/bin/sudospawner
```

_actually give the Hub user permission to run the above command on behalf of the clearinghouse users without prompting for a password_

```
%jupyterhub ALL=(jupyterhub) /bin/sudo
jupyterhub ALL=(%clearinghouse) NOPASSWD:JUPYTER_CMD
```

_add users who use clearinghouse, and jupyterhub, to linux group clearinghouse_

### Directory setup

_set up new base directory for clearinghouse_

```
mkdir /home/clearinghouse/
cd /home/clearinghouse
chown jupyterhub .
chgrp clearinghouse .
chmod -R g+w .
chmod -R g+s .
setfacl -dm g:clearinghouse:rw .
```

### ArangoDB setup

_Follow directions from ArangoDB website; something like this:_

```
cd /tmp
curl -OL https://download.arangodb.com/arangodb36/DEBIAN/Release.key
sudo apt-key add - < Release.key
echo 'deb https://download.arangodb.com/arangodb36/DEBIAN/ /' | tee /etc/apt/sources.list.d/arangodb.list
apt-get install apt-transport-https
apt-get update
apt-get install arangodb3=3.6.1-1
```

### Make SystemD start up Jupyterhub automatically

_in /etc/systemd/system, make jupyterhub.service:_

```
[Unit]
Description=Jupyterhub
After=network-online.target

[Service]
User=jupyterhub
ExecStart=/usr/local/bin/jupyterhub --JupyterHub.spawner_class=sudospawner.SudoSpawner
WorkingDirectory=/etc/jupyterhub

[Install]
WantedBy=multi-user.target
```

_make it start automatically:_

```
systemctl enable jupyterhub
service jupyterhub start
```

_to test:_

service jupyterhub status

### Get Jupyterhub and ArangoDB working behind Apache

_enable all options needed by apache:_

a2enmod ssl rewrite proxy proxy_http proxy_wstunnel

_in apache conf (/etc/apache2/sites-enabled/000-default.conf):_

```
        SSLProxyEngine on
        SSLProxyVerify none
        SSLProxyCheckPeerCN off
        SSLProxyCheckPeerName off
        SSLProxyCheckPeerExpire off
        ProxyPreserveHost On
        ProxyRequests off

        TraceEnable Off

        <Location /jupyterhub>
            ProxyPass https://localhost:8000/jupyterhub
            ProxyPassReverse https://localhost:8000/jupyterhub
            ProxyPassReverseCookieDomain localhost brass.lbl.gov
        </Location>

        <LocationMatch "/jupyterhub/(user/[^/]*)/(api/kernels/[^/]+/channels|terminals/websocket)(.*)">
            ProxyPassMatch wss://localhost:8000/jupyterhub/$1/$2$3
            ProxyPassReverse wss://localhost:8000/jupyterhub/$1/$2$3
        </LocationMatch>

        <Location /arangodb/>
            ProxyPass http://localhost:8529/
            ProxyPassReverse http://localhost:8529/
            ProxyPreserveHost On
            AuthType Basic
            AuthName "Restricted Content"
            AuthUserFile /etc/apache2/htpasswd
            Require valid-user
        </Location>

        <Location /_db/>
            ProxyPass http://localhost:8529/_db/
            ProxyPassReverse http://localhost:8529/_db/
            ProxyPreserveHost On
        </Location>

        <Location /_api/>
            ProxyPass http://localhost:8529/_api/
            ProxyPassReverse http://localhost:8529/_api/
            ProxyPreserveHost On
        </Location>
```

_Set up Data Clearinghouse front end links_

_e.g., in /home/httpd/html/dc/index.html:_

```
<html>
<body>
Data Clearinghouse
  <ul>
    <li><a href="/generix-ui/">CORAL UI</a>
    <li><a href="/jupyterhub/">JupyterHub</a>
    <li><a href="/arangodb/">ArangoDB</a>
  </ul>
</body>
</html>
```

### ArangoDB Config

Now that you can get at ArangoDB through Apache, log in and
create databases.  You need at least a "production" database,
but you could have "test" or other versions for development.
They can be called whatever you want.

### Generix Web Services

_These run in a virtualenv, so install there_

```
pip3 install virtualenv 
python3 -m virtualenv /home/clearinghouse/env/
source /home/clearinghouse/env/bin/activate
pip3 install flask flask_cors pandas simplepam pyjwt pyArango dumper xarray openpyxl
```

_note:  Be careful to install pyjwt, NOT jwt!  Or login will fail!_

_setup /etc/systemd/system/generix-web-services.service:_

```
[Unit]
Description=Generix Web Services
After=network.target

[Service]
User=root
EnvironmentFile=/etc/sysconfig/generix-web-services
ExecStart=/home/clearinghouse/env/bin/python -m generix.web_services
WorkingDirectory=/home/clearinghouse/prod/modules/generix_prototype
Restart=always
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

### More setup of directory structure

Here I'm going to assume you have only a "prod" environment for
production.  But as mentioned above, you might want more than one,
for testing and development.

```
cd /home/clearinghouse
mkdir prod
cd prod
mkdir images
mkdir data_import
mkdir data_store
mkdir data_store/tmp
mkdir notebooks
mkdir modules
cd modules
git clone git@github.com:jmchandonia/generix_prototype.git
```

### Initial data load

_load in the data from a jupyter notebook:_

_rsync data_import, notebooks, images from server with data_

_run "reload_data" notebook to load and set up._

_this will set up tables required for web services to start.  It may need to be done semi simultaneously with the following step, though:_

### Start Web Services

_to start web services_

service generix-web-services start

_debug by looking in /var/log/daemon.log_

### Install UI

See https://github.com/jmchandonia/generix-ui
