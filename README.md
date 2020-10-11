
# PC-Admins Synapse Setup Guide 2


This guide covers complete Synapse setup for Debian 10 with Postgresql. It includes the often missing sections on how to configure postgresql and coturn with Synapse.

You can use this guide to make your own encrypted chat server for your existing domain that's running on another server. This is useful if you want to host a Matrix instance for your company/organisation and you already have a web address.

You will need at least a 1GB VPS although I recommend 2GB for a small server. You will need a desired domain name. This guide will setup a Matrix service at ‘matrix.example.org’ with Riot-Web hosted through NGINX on the same server at ‘chat.example.org‘. While user IDs appear without the 'matrix.' like so: @user:example.org

For a guide on how to make just a Matrix/Riot/Coturn service on its own domain name please see: https://github.com/PC-Admin/PC-Admins-Synapse-Setup-Guide

Join the discussion at: #synapsesetupguide:matrix.org if you get stuck or have an edit in mind.
***
## Licensing

This work is licensed under Creative Commons Attribution Share Alike 4.0, for more information on this license see here: https://creativecommons.org/licenses/by-sa/4.0/
***
## Server Setup

Configure a Debian 10 server with auto-updates, security and SSH access. Ports 80/tcp, 443/tcp, 8448/tcp, 3478/udp, 5349/udp, 50001-50100/udp will need to be open for the web service, synapse federation and a coturn service.
***
## DNS Records

Set up 2 simple A records. With ‘matrix.example.org’ and ‘chat.example.org‘ pointed to your servers IP. 
Additionally you might setup a DNS SRV record, though it's only necessary, when you changed your federation port to listen on another port the the default port 8448.

Example DNS SRV record:
```
_matrix._tcp        3600 IN SRV     10 0 8448 example.org.
```

***
## Installing Synapse

Follow [the official Debian install instructions](https://github.com/matrix-org/synapse/blob/master/INSTALL.md#debianubuntu) using the matrix.org packages.

You will be asked to set the name of your server after `apt install`, enter your desired domain name here. (eg: example.org)

Finally check that the synapse server is shutdown

`$ sudo systemctl stop matrix-synapse`

***
## Installing Postgresql
The default synapse install generates a config that uses sqlite. It has the advantage of being easy to setup as there's no db server setup to take care about. But from my experience the performance penalty is quite big and if you want to do something more then testing or running a small non federated server, switching to postgres should be a mandatory step.

So let's install postgresql:

`$ sudo apt install postgresql postgresql-client`

Create Role and Database

`$ sudo -i -u postgres`

`$ createuser synapse -P --interactive`
```
postgres@VM:~$ createuser synapse -P --interactive
Enter password for new role: 
Enter it again: 
Shall the new role be a superuser? (y/n) n
Shall the new role be allowed to create databases? (y/n) n
Shall the new role be allowed to create more new roles? (y/n) n
```
Remember the db user password, you'll need it later

Now we're back at $postgres. Let's create a database for Synapse with correct settings and set the owner to be the user we just created:

Type: `psql`

..And create the database as follows:

`postgres=# CREATE DATABASE synapse WITH ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0 OWNER synapse;`

Exit from psql by typing `'\q'` 

All done. Let's exit from postgres account by typing 'exit' so land back at our own user.

***
## Adapt Synapse config to use Postgresql
Now as we have created the db and a user to be able to connect, we need to change the synapse config to use it:

`$ sudo nano /etc/matrix-synapse/homeserver.yaml`

Before the change it should look like this:
```
# For more information on using Synapse with Postgres, see `docs/postgres.md`.
#
database:
  name: sqlite3
  args:
    database: /var/lib/matrix-synapse/homeserver.db
```

Modify it to look like this:
```
database:
    name: psycopg2
    args:
        user: synapse
        password: "your-db-user-password"
        database: synapse
        host: localhost
        cp_min: 5
        cp_max: 10
```

***
## Certbot Setup

`$ sudo apt install certbot`

Test if server IP can be pinged first, if it can then run:

`$ sudo certbot certonly --rsa-key-size 4096 -d matrix.example.org -d chat.example.org -d turn.example.org`

Choose ‘spin up a temporary webserver’

enter a recovery email

enter ‘matrix.example.org’ as the domain

```
Performing the following challenges:
http-01 challenge for matrix.example.org
http-01 challenge for chat.example.org
http-01 challenge for turn.example.org
Waiting for verification...
Cleaning up challenges

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/matrix.example.org/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/matrix.example.org/privkey.pem
```

***
## Setup SSL Auto-renewal

for monthly renewal, set a crontab:

`$ sudo crontab -e`

Insert Line:

`@monthly certbot renew --rsa-key-size 4096 --quiet --post-hook "systemctl reload nginx"`

***
## Set public_baseurl and web_client_location:

Change these values, these are the locations of the Riot client and the Matrix service.
```
$ sudo nano /etc/matrix-synapse/homeserver.yaml
```
Edit so:
```
web_client_location: https://chat.example.org/

public_baseurl: https://matrix.example.org/
```

***
## Configure NGINX with A+ SSL

Generate dhparam key and move it to your letsencrypt folder:
```
$ openssl dhparam -out dhparam4096.pem 4096
$ sudo mv ./dhparam4096.pem /etc/letsencrypt/live/matrix.example.org/dhparam4096.pem
$ sudo chown root:root /etc/letsencrypt/live/matrix.example.org/dhparam4096.pem
```
Install NGINX and configure:
```
$ sudo apt install nginx -y
```
Remove default NGINX configuration:
```
$ sudo rm /etc/nginx/sites-available/default
$ sudo rm /etc/nginx/sites-enabled/default
```
Edit a new NGINX configuration for matrix.example.org:
```
$ sudo nano /etc/nginx/sites-available/matrix.example.org.conf
```
Add:
```
server {
    listen         80;
    server_name    matrix.example.org;
    return         301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen 8448 ssl http2;  # for federation (skip if pointing SRV or .well-known to port 443)
    gzip off;
    server_name matrix.example.org;

    ssl_certificate     /etc/letsencrypt/live/matrix.example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.org/privkey.pem;
    ssl_session_cache   shared:NGX_SSL_CACHE:10m;
    ssl_session_timeout 12h;
    ssl_protocols       TLSv1.3 TLSv1.2;
    ssl_ciphers		"TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256";
    ssl_dhparam         /etc/letsencrypt/live/matrix.example.org/dhparam4096.pem;
    ssl_ecdh_curve      X25519:secp521r1:secp384r1:prime256v1;

    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains" always;
    add_header X-Content-Type-Options "nosniff" always;

    location / {  
        return 301 https://chat.example.org;
    }

    location /_matrix {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }

}
```

^ Make sure to replace the server name here with yours!

Create a symbolic link for it like so:

`$ sudo ln -s /etc/nginx/sites-available/matrix.example.org.conf /etc/nginx/sites-enabled/matrix.example.org.conf`

Restart service and renew SSL:

`$ sudo certbot renew --rsa-key-size 4096 --quiet --post-hook "systemctl reload nginx"`

If you get a 'Cert not yet due for renewal' error wait a few hours and try again.


Now synapse should be ready and we can see if it starts without errors:

`$ sudo systemctl start matrix-synapse`

`$ sudo systemctl status matrix-synapse`

***
## Fine Tune Synapse
There're two files that manage the behaviour of synapse:
 
- Server config file in /etc/matrix-synapse/homeserver.yaml
- Env file in /etc/default/matrix-synapse

The first is used to do the configuration of synapse, the second is used to setup the environment synapse is running in.

### Registration and guest access

- Set minimum version for federation TLS:

    File: /etc/matrix-synapse/homeserver.yaml: `federation_client_minimum_tls_version: 1.2`

- Federation Blacklist (optional):

    File:  /etc/matrix-synapse/homeserver.yaml: 
```
    federation_ip_range_blacklist:
    #  - '127.0.0.0/8'
    #  - '10.0.0.0/8'
    #  - '172.16.0.0/12'
    #  - '192.168.0.0/16'
    #  - '100.64.0.0/10'
    #  - '169.254.0.0/16'
    #  - '::1/128'
    #  - 'fe80::/64'
    #  - 'fc00::/7'
```

- Allow public room list to federate (optional)

    File:  /etc/matrix-synapse/homeserver.yaml: `allow_public_rooms_over_federation: true`

- Registration (optional)

    File:  /etc/matrix-synapse/homeserver.yaml: `enable_registration: true`

- Admin email (optional)

    File: /etc/matrix-synapse/homeserver.yaml: `admin_contact: 'mailto:admin@example.org’`


**There are other settings here you may want to adjust. I would do so one at a time, testing each change as you go.**

### Cache factor

For a small server (<=2GB), an adoption of the cache factor might improve performance. Some time ago the advice was to lower the cache factor to use less RAM. Experience has shown that the effect is quite the opposite, see [Issue](https://github.com/matrix-org/synapse/pull/4276).

So the new advice is to **raise the cache factor to use less RAM**, with a value of 2 being a good starting point:

- Cache factor

    File: /etc/default/matrix-synapse: 
```
SYNAPSE_CACHE_FACTOR=2.0
```
 
***

<span style="color:red">**Don't forget to restart synapse and examine the RAM usage after each change:**</span>

`$ sudo systemctl restart matrix-synapse`

***
## Load Riot-Web client into NGINX

NGINX content location: /var/www/chat.example.org
```
$ sudo mkdir /var/www/chat.example.org
```

Edit NGINX configuration for chat.example.org:
```
$ sudo nano /etc/nginx/sites-available/chat.example.org.conf
```
Add:
```
server {
    listen         80;
    server_name    chat.example.org;
    return         301 https://chat.example.org$request_uri;
}

server {
    listen 443 ssl http2;
    gzip off;
    server_name chat.example.org;

    ssl_certificate     /etc/letsencrypt/live/matrix.example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.org/privkey.pem;
    ssl_session_cache   shared:NGX_SSL_CACHE:10m;
    ssl_session_timeout 12h;
    ssl_protocols       TLSv1.3 TLSv1.2;
    ssl_ciphers		"TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256";
    ssl_dhparam         /etc/letsencrypt/live/matrix.example.org/dhparam4096.pem;
    ssl_ecdh_curve      X25519:secp521r1:secp384r1:prime256v1;

    add_header Strict-Transport-Security "max-age=31536000; includeSubdomains" always;
    add_header X-Content-Type-Options "nosniff" always;

    root /var/www/chat.example.org;
    index index.html; 

    location / {
        try_files $uri $uri/ =404;
    }

}
```

Create a symbolic link for it like so:
```
$ sudo ln -s /etc/nginx/sites-available/chat.example.org.conf /etc/nginx/sites-enabled/chat.example.org.conf
```

https://github.com/vector-im/riot-web/releases/latest

Download latest riot-web and move contents into nginx folder:
```
$ wget https://packages.riot.im/riot-release-key.gpg
$ gpg --import riot-release-key.gpg
$ wget https://github.com/vector-im/riot-web/releases/download/v1.6.0/riot-v1.6.0.tar.gz
$ wget https://github.com/vector-im/riot-web/releases/download/v1.6.0/riot-v1.6.0.tar.gz.asc
$ gpg --verify riot-v1.6.0.tar.gz.asc riot-v1.6.0.tar.gz
gpg: Signature made Fri 01 May 2020 03:37:12 PM UTC
gpg:                using RSA key 5EA7E0F70461A3BCBEBE4D5EF6151806032026F9
gpg:                issuer "releases@riot.im"
gpg: Good signature from "Riot Releases <releases@riot.im>" [unknown]
$ tar -zxvf ./riot-v1.6.0.tar.gz
$ sudo mv ./riot-v1.6.0/* /var/www/chat.example.org/
$ rm -r ./riot-v*
```
Create and edit config.json in nginx directory:

Feel free to customize config.json to suit your needs. All of the lines in config.json are optional.
```
$ sudo cp /var/www/chat.example.org/config.sample.json /var/www/chat.example.org/config.json

$ sudo nano /var/www/chat.example.org/config.json
```
Add:
```
{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://matrix.example.org",
            "server_name": "example.org"
        },
        "m.identity_server": {
            "base_url": "https://vector.im"
        }
    },
    "disable_custom_urls": false,
    "disable_guests": false,
    "disable_login_language_selector": false,
    "disable_3pid_login": false,
    "brand": "Riot",
    "integrations_ui_url": "https://scalar.vector.im/",
    "integrations_rest_url": "https://scalar.vector.im/api",
    "integrations_widgets_urls": [
        "https://scalar.vector.im/_matrix/integrations/v1",
        "https://scalar.vector.im/api",
        "https://scalar-staging.vector.im/_matrix/integrations/v1",
        "https://scalar-staging.vector.im/api",
        "https://scalar-staging.riot.im/scalar/api"
    ],
    "bug_report_endpoint_url": "https://riot.im/bugreports/submit",
    "defaultCountryCode": "AU",
    "showLabsSettings": false,
    "features": {
        "feature_pinning": "labs",
        "feature_custom_status": "labs",
        "feature_custom_tags": "labs",
        "feature_state_counters": "labs"
    },
    "default_federate": true,
    "default_theme": "light",
    "roomDirectory": {
        "servers": [
            "matrix.org",
            "perthchat.org"
        ]
    },
    "welcomeUserId": "@riot-bot:matrix.org",
    "piwik": {
        "url": "https://piwik.riot.im/",
        "whitelistedHSUrls": ["https://matrix.org"],
        "whitelistedISUrls": ["https://vector.im", "https://matrix.org"],
        "siteId": 1
    },
    "enable_presence_by_hs_url": {
        "https://matrix.org": false,
        "https://matrix-client.matrix.org": false
    },
    "settingDefaults": {
        "breadcrumbs": true
    },
    "jitsi": {
        "preferredDomain": "jitsi.riot.im"
    }
}
```

Reset NGINX:

`$ sudo systemctl restart nginx`

You should be able to view and use Riot-Web through your URL now, test it out.

***
## Edit .well-known on the original website:

On the original website (not the server you just setup) you need to make a .well-known (https://github.com/matrix-org/synapse/blob/master/docs/delegate.md), here is an example on how to do it with NGINX:
```
server {
    listen         80;
    server_name    example.org www.example.org;
    return         301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;

    server_name www.example.org;
    ssl_certificate     /etc/letsencrypt/live/example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.org/privkey.pem;

    rewrite ^/(.*) https://example.org/$1 permanent;
}

server {
    listen 443 ssl;
    gzip off;
    server_name example.org;

    ssl_certificate     /etc/letsencrypt/live/example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.org/privkey.pem;
    ssl_session_cache   shared:NGX_SSL_CACHE:10m;
    ssl_session_timeout 12h;
    ssl_protocols       TLSv1.3 TLSv1.2;
    ssl_ciphers		"TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256";
    ssl_dhparam         /etc/letsencrypt/live/example.org/dhparam4096.pem;
    ssl_ecdh_curve      X25519:secp521r1:secp384r1:prime256v1;

    add_header Strict-Transport-Security "max-age=0; includeSubdomains" always;
    add_header X-Content-Type-Options "nosniff" always;

    root /var/www/example.org;
    index index.html; 

    location / {
        try_files $uri $uri/ =404;
    }

    location /.well-known/matrix/server {
        return 200 '{ "m.server": "matrix.example.org:8448" }';
        add_header content-type application/json;
    }

    location /.well-known/matrix/client {
        return 200 '{ "m.homeserver": { "base_url": "https://matrix.example.org" }, "im.vector.riot.jitsi": { "preferredDomain": "jitsi.riot.im" } }';
        add_header access-control-allow-origin *;
        add_header content-type application/json;
    }
```

Notice the 2 .well-known locations being served at the bottom.

***
## Configure TURN service:

Your matrix server still cannot make calls across NATs (different routers), for this we need to configure coturn.

Configure a simple A DNS record pointing turn.example.org to your servers IP.

Install coturn:

`$ sudo apt install coturn`

Edit coturn configs:
```
$ sudo nano /etc/default/coturn

#
# Uncomment it if you want to have the turnserver running as
# an automatic system service daemon
#
TURNSERVER_ENABLED=1
```

Create certs that turnserver user can access:
```
mkdir -p /etc/coturn/certs
chown -R turnserver:turnserver /etc/coturn/
chmod -R 700 /etc/coturn/
nano /etc/letsencrypt/renewal-hooks/deploy/coturn-certbot-deploy.sh
chmod 700 /etc/letsencrypt/renewal-hooks/deploy/coturn-certbot-deploy.sh
```

In the renewal hook insert:
```
#!/bin/sh

set -e

for domain in $RENEWED_DOMAINS; do
        case $domain in
        example.org)
                daemon_cert_root=/etc/coturn/certs

                # Make sure the certificate and private key files are
                # never world readable, even just for an instant while
                # we're copying them into daemon_cert_root.
                umask 077

                cp "$RENEWED_LINEAGE/fullchain.pem" "$daemon_cert_root/$domain.cert"
                cp "$RENEWED_LINEAGE/privkey.pem" "$daemon_cert_root/$domain.key"

                # Apply the proper file ownership and permissions for
                # the daemon to read its certificate and key.
                chown turnserver "$daemon_cert_root/$domain.cert" \
                        "$daemon_cert_root/$domain.key"
                chmod 400 "$daemon_cert_root/$domain.cert" \
                        "$daemon_cert_root/$domain.key"

                service coturn2 restart >/dev/null
                ;;
        esac
done
```

Renew certbot:

`$ sudo certbot renew --force-renewal`

Generate a ‘shared-secret-key’ and record it, this can be done like so:
```
$ < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-64};echo;
5PFhfL1Eoe8Wa6WUxpR4wcwKUqkcl3UUg9QeOmpfnGHpW2O9cOsZ5yIoCDgMMdVP
```

Generate a cli-password and record it, then generate a ’hash’ from it using turnadmin:
```
$ turnadmin -P -p "your-cli-password"
$5$6fc2691fa3d289f9$8a7079825a7d4bfce772ed278c4d1549936b96b27ab1b3014a090492437feb45
```

Edit turnserver config like so:
```
$ sudo nano /etc/turnserver.conf
```

Add:
```
# Append this to the bottom of the new turnserver config:
listening-port=3478
tls-listening-port=5349
lt-cred-mech
fingerprint
stale-nonce
use-auth-secret
static-auth-secret=shared-secret-key
cli-password=hash
server-name=turn.example.org
realm=turn.example.org
cert=/etc/coturn/certs/example.org.cert
pkey=/etc/coturn/certs/example.org.key
dh2066
userdb=/var/lib/turn/turndb
#no-stdout-log
log-file=/var/tmp/turn.log
#syslog
simple-log
mobility
no-tlsv1
no-tlsv1_1
no-tcp-relay
#allowed-peer-ip=10.0.0.1
user-quota=12
total-quota=1200
no-multicast-peers
no-tcp
min-port=55001
max-port=50100
```

Edit homeserver.yaml:
```
$ sudo nano /etc/matrix-synapse/homeserver.yaml

turn_uris: [ "turn:turn.example.org:3478?transport=udp", "turn:turn.example.org:5349?transport=udp" ]
turn_shared_secret: shared-secret-key
turn_user_lifetime: 1h
turn_allow_guests: true
```

Restart both the new coturn service and matrix-synapse, then test cross-NAT calling:
```
$ sudo systemctl start coturn
$ sudo systemctl restart matrix-synapse
```

If you already have jitsi installed, this process is much easier and slightly modified. Firstly, do not set up a new domain for turn, instead open up `/etc/turnserver.conf` to find out your secret key and tls port. Now edit your homeserver.yaml like so:

```
# nano /etc/matrix-synapse/homeserver.yaml

turn_uris: [ "turn:meet.example.org:4445?transport=udp", "turn:meet.example.org:4445?transport=tcp" ]
turn_shared_secret: "YOUR_SECRET_FROM_TURN_CONFIG"
turn_user_lifetime: 86400000
turn_allow_guests: true
```

Then restart only synapse (not coturn since we never edited the coturn config).
***
## Done!

Your Synapse is now up and running and your hosting the latest Riot through the Nginx web server. You're also running a jitsi-meet instance.

You should familiarise yourself with the Synapse wiki: https://github.com/matrix-org/synapse/wiki

Find extra steps to configure your Synapse server here: https://github.com/PC-Admin/PC-Admins-Synapse-Extras

