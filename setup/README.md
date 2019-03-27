### Variables used in this manual

```
UBERSPACE_SERVER_DIR = the directory on uberspace, in which the meteor app should run. Example: "meteor-server/myapp"
UBERSPACE_SERVICENAME = the name of the service/deamon that runs this app. Example: "meteor-server-myapp"
```

### 0. Open a port

This will enable you to use websockets
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

### 3. bridge to https server

If you don't want to use HTTPS for whatever reason, just ignore the middle part and use only `RewriteEngine On` and `RewriteRule ^(.*)$ http://localhost:64991/$1 [P]`.

Add to ~/html/.htaccess :

```
RewriteEngine On

RewriteCond %{HTTPS} !=on
RewriteCond %{ENV:HTTPS} !=on
RewriteRule .* https://%{SERVER_NAME}%{REQUEST_URI} [R=301,L]

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

If you do not plan to intialize your database through an existing dump, you need to add a "dbAdmin" role for your user. This will enable your meteor app to create new collections in the empty dabatase. See https://docs.mongodb.com/manual/reference/built-in-roles/ for details on roles.
```
use myapp_prod
db.createUser(
    {
      user: "myapp_prod",
      pwd: "7hfkBYNPWPMOIDm4",
      roles: [
         { role: "readWrite", db: "myapp_prod" },
         { role: "dbAdmin", db: "myapp_prod" }
      ]
    }
)
```


### 8. Setup meteor service variables in ~/service/myapp-meteor/run using above ports and mongo credentials
If you use a custom domain (step 11) you can also use your own domain as `DDP_DEFAULT_CONNECTION_URL` and `ROOT_URL`. Make sure you have a valid HTTPS certificate then. 

It's explained how to get one here for your own domain: https://wiki.uberspace.de/webserver:https .

If you don't want to use HTTPS, just write `https://...`for both URLs. 

```
export DISABLE_WEBSOCKETS=1
#OR
export DDP_DEFAULT_CONNECTION_URL=https://username.subdomain.uberspace.de/

export ROOT_URL='https://username.subdomain.uberspace.de/'
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
