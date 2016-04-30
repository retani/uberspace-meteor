UBERSPACE HOSTING:

```
UBERSPACE_SERVER_DIR = the directory on uberspace, in which the meteor app should run. Example: "meteor-server/myapp"
UBERSPACE_SERVICENAME = the name of the service/deamon that runs this app. Example: "meteor-server-myapp"
```

### 0. Open a port

This will enable you to use websockets (unless you want to use HTTPS)
```
$ uberspace-add-port -p tcp --firewall
All good! Opened port 64991, tcp protocol(s).
```
Remember the port number 64991

### 1. Activate services

```
$ test -d ~/service || uberspace-setup-svscan 
```

### 2. set up folder structure

```
$ mkdir ~/UBERSPACE_SERVER_DIR
```

assuming: meteor bundle in ~/UBERSPACE_SERVER_DIR/bundle

### 3. bridge to http server

Add to ~/html/.htaccess :

```
RewriteEngine On
RewriteRule ^(.*)$ http://localhost:64991/$1 [P]
```

### 4. Setup npm

```
$ cat > ~/.npmrc <<__EOF__
prefix = $HOME
umask = 077
__EOF__
```

### 5. Setup meteor service

```
$ uberspace-setup-service UBERSPACE_SERVICENAME node ~/UBERSPACE_SERVER_DIR/bundle/main.js 
```

### 6. Setup mongo

```
$ uberspace-setup-mongodb 

Hostname: localhost
Portnum#: 21040
Username: username_mongoadmin
Password: XXXXXXXXXX
```

To connect on the shell, you can use:

```
$ mongo admin --port 21040 -u username_mongoadmin -p
```

### 7. Add mongo user by eecuting this code in mongo admin shell (above command to open it). The "pwd" is up to your choice (create one with 'openssl rand -base64 16' for example).

```
use myapp_prod
db.createUser(
    {
      user: "myapp_prod",
      pwd: "7hfkBYNPWPMOIDm4",
      roles: [
         { role: "readWrite", db: "myapp_prod" }
      ]
    }
)
```

### 8. Setup meteor service variables in ~/service/myapp-meteor/run using above ports and mongo credentials

```
export DISABLE_WEBSOCKETS=1
#OR
export DDP_DEFAULT_CONNECTION_URL=http://username.subdomain.uberspace.de:64991/

export ROOT_URL='http://username.subdomain.uberspace.de/'
export PORT=64991
export MONGO_URL='mongodb://myapp_prod:7hfkBYNPWPMOIDm4@localhost:21040/myapp_prod'
```

### 9. populate database from existing dump

```
$ mongorestore --port 21040 -u myapp_prod -p -d myapp_prod  ~/dumps/2016-02-01-19-42/myapp_meteor_com/
```

### 10. (optional) setup mongodb backup service

a) create backup script, for example in ~/bin/myapp-dump-mongodb. Note: This script does not delete old backups and will fill up your drive over time.

```
#!/bin/bash
mongodump -u myapp_prod -h localhost --port 21040 -d myapp_prod -p "7hfkBYNPWPMOIDm4" -o ~/mongodb-backups/`date +"%Y-%m-%d-%H-%M"`
```

b) make executable

```
$ chmod a+x ~/bin/myapp-dump-mongodb
```

c) setup runwhen

```
$ mkdir ~/mongodb-backups
$ runwhen-conf ~/etc/mongodb-backup ~/bin/myapp-dump-mongodb
```

d) configure runswhen script. open ~/etc/mongodb-backup/run in editor and change RUNWHEN variable

```
# every day at 4am
RUNWHEN=",H=4"
```

e) activate runwhen

```
$ ln -s ~/etc/mongodb-backup ~/service
```

f) test it

```
$ svc -a ~/service/mongodb-backup
```

A dump should appear in ~/mongodb-backups

### 11. (optional) add custom domain

```
$  uberspace-add-domain -d mydomain.com -w -m
The webserver's configuration is adapted; it will get active within at most 5 minutes.
Now you can use the following records for your dns:
  A -> 185.26.156.43
  AAAA -> 2a00:d0c0:200:0:b9:1a:9c2b:229
The mailserver's configuration is now adapted; it is now active.
Now you can use the following record for your dns:
  MX -> gienah.uberspace.de

If you want to use our automx service, you'll also need:
  A autoconfig.mydomain.com -> 185.26.156.43
  AAAA autoconfig.mydomain.com -> 2a00:d0c0:200:0:b9:1a:9c2b:229
  A autodiscover.mydomain.com -> 185.26.156.43
  AAAA autodiscover.mydomain.com -> 2a00:d0c0:200:0:b9:1a:9c2b:229
```
