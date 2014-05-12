---
layout: post
title: "Simple Guide to Install StatsD and Graphite"
date: 2014-05-12 20:46
comments: true
---
I've been playing around with [StatD](https://github.com/etsy/statsd) and [Graphite]() lately. It took me quite a while to set this stack up. So I think it worthwhile to write a post to walk through the whole installing and configuring process for future reference.

Here is a checklist of all the softwares I'm gonna use:

- StatD
- Graphite
- [Grafana](http://grafana.org/)
- [Gunicorn](http://gunicorn.org/)
- Nginx
- [Supervisor](http://supervisord.org/)

I'm setting up all these stuff inside a Ubuntu precise32 [Vagrant box](http://files.vagrantup.com/precise32.box). If you are also using a Vagrant box, add these settings in your Vagrant file.
``` ruby
 config.vm.network :forwarded_port, guest: 80, host: 8080
 config.vm.network :forwarded_port, guest: 8125, host: 8125, protocol: 'udp'
```
Let's ssh into vagrant and change to root first
``` bash
 vagrant ssh
 sudo su -
```

### Install Graphite

``` bash
 apt-get install git python-virtualenv python-dev
 virtualenv /opt/graphite
 source /opt/graphite/bin/activate
 pip install https://github.com/graphite-project/ceres/tarball/master
 pip install whisper
 pip install carbon
```

### Install Graphite Web
``` bash
 apt-get install libcairo2-dev
 cd /opt/graphite
 git clone https://github.com/graphite-project/graphite-web.git
 cd graphite-web
 git checkout 0.9.12
 python setup.py install
 pip install -r requirements.txt
 django-admin.py syncdb --settings=graphite.settings --pythonpath=/opt/graphite/webapp
```
Graphite includes a wsgi file in its installation. Just copy it for later deployment 
```
 cp /opt/graphite/conf/graphite.wsgi.example /opt/graphite/webapp/wsgi.py
```

### Configure Carbon
``` bash
 cd /opt/graphite/conf/
 cp carbon.conf.example carbon.conf
 cat > storage-schemas.conf << EOF
[stats]
pattern = ^stats.*
retentions = 10s:6h,1min:6d,10min:1800d
EOF
```
The storage schema is copied from [StatsD](https://github.com/etsy/statsd/blob/master/docs/graphite.md#storage-schemas). Tweak it to meet your needs.

### Install statsd

``` bash
 apt-get install nodejs
 cd /opt/
 git clone https://github.com/etsy/statsd.git
 cat > statsd/config.js << EOF
{
  graphitePort: 2003
, graphiteHost: "127.0.0.1"
, port: 8125
, backends: [ "./backends/graphite" ]
, legacyNamespace: false
}
EOF
```

### Install Grafana
``` bash
 cd /opt/
 git clone https://github.com/grafana/grafana.git
 cp grafana/src/config.sample.js grafana/src/config.js
```

### Change Permissions
``` bash
 adduser tom
 chown tom:tom -R /opt/graphite /opt/statsd /opt/grafana/
```

### Manage process with Supervisord
``` bash
 apt-get install supervisor
 cat > /etc/supervisor/conf.d/gunicorn.conf << EOF
[program:gunicorn]
command = /opt/graphite/bin/gunicorn -b 127.0.0.1:8080 -w 2 --pythonpath /opt/graphite/webapp/ wsgi:application
directory = /opt/graphite/webapp/
user = tom
autostart = true
autorestart = true
redirect_stderr = true
EOF
 cat > /etc/supervisor/conf.d/statsd.conf << EOF
[program:statsd]
command = /usr/bin/node stats.js config.js
directory = /opt/statsd/
user = tom
autostart = true
autorestart = true
redirect_stderr = true
EOF
 cat > /etc/supervisor/conf.d/carbon.conf << EOF
[program:carbon]
command = /opt/graphite/bin/carbon-cache.py start --debug
user = tom
autostart = true
autorestart = true
redirect_stderr = true
EOF
 supervisorctl reload
```

### Set up Nginx
```
server {
        listen   80;

        location / {
                add_header Access-Control-Allow-Origin "*";
                proxy_pass http://127.0.0.1:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }

        location /grafana/ {
                alias /opt/grafana/src/;
                index index.html;
        }
}
```
``` bash
 apt-get install nginx
 ln -s /etc/nginx/sites-available/graphite /etc/nginx/sites-enabled/
 rm /etc/nginx/sites-enabled/default
 /etc/init.d/nginx restart
```
Now go to http://127.0.0.1:8080/ for Graphite and http://127.0.0.1:8080/grafana/ for Grafana.
