# shadowban.dev

***Quickstart:***
*[Install mkcert](https://github.com/FiloSottile/mkcert#installation), [node.js](https://nodejs.org/en/download/) and run `./manage init`*

```bash
$ mkdir shadowban-eu; cd $_
$ git clone https://github.com/shadowban-eu/docker ./docker; cd $_
$ ./manage init
```

Windows users, please install a bash shell! The emulation of [git for windows](https://gitforwindows.org/) works fine.

---
The compose file `./docker-compose.yml` creates a
network (`shadowban-dev_common`) with mongo (`db`),
nginx (`www`), frontend (`pwa`) and `testing` containers.

| host: shadowban-db | |
|-:|-|
| image | mongo |
| ports | 127.0.0.1:27017:27017 |
| config | ./env/mongo.env |
| **shadowban-www** |  |
| image | www/. (nginx:alpine) |
| ports | 127.0.0.1:80:80 (http)<br>127.0.0.1:443:443 (https) |
| config | www/nginx.conf<br>www/sites-/\*<br>www/ssl/\*\*/(key\|cert).pem |
| logs | ./logs/nginx/ |
| **shadowban-[testing](https://github.com/shadowban-eu/testing)** |  |
| image | ../testing/. (python:3.7-slim-buster) |
| config | ./env/testing.env |
| logs | ./logs/testing/ |
| **shadowban-[pwa](https://github.com/shadowban-eu/pwa)** |  |
| image | ../pwa/. (node:slim) |
| config | ./env/pwa.env |
| **shadowban-[timeline-termination](https://github.com/shadowban-eu/timeline-termination)**| |
| image | ../timeline-termination/. (node:slime) |
| config | ./env/timeline-termination.env |
| logs | ./logs/timeline-termination/ |

## manage
```
Usage: ./manage [init|mkcert <domain>|*] [-b <branch>] [-d <uri>] [-h]
Setup and manage shadowban.eu development docker containers

Depends:
  docker >=18.0.9, docker-compose >=1.25, [mkcert]

Managing commands:
  init              Clones all services, builds images,
                    sets up SSL, etc.
  dev <service>     Run <service> in foreground and all other services detached.
  mkcert <domain>   Creates a new SSL cert/key pair for <domain>

Managing options:
  -g <uri>          Base URI for repositories to clone from
                    e.g. https://github.com/shadowban-eu/
  -b <branch>       Branch name to check out; default: master
  -h                You're looking at it.

Other:
  *                 All other arguments are passed to docker-compose
                    e.g. './manage help' to see the docker-compose help
```

```bash
# Bring up containers
# [-d] detach from your terminal process
./manage up [-d] (db|pwa|testing|timeline-termination|www)

# Stop a service
./manage stop (db|pwa|testing|timeline-termination|www)

# Bring all services down (stop and destroy)
# This will wipe your DB!
./manage down

# Execute command inside of running container
./manage exec (db|pwa|testing|timeline-termination|www) <CMD>
# e.g. reload nginx configs
./manage exec www nginx -s reload
```

## SSL certificate

### mkcert
[Install mkcert](https://github.com/FiloSottile/mkcert#installation) and generate/install the CA certificate.
You probably want to restrict the automatic installation to browsers, setting `$TRUST_STORES` to `nss`.

```bash
TRUST_STORES=nss mkcert -install
```

Finish the SSL setup by creating a certificate for `shadowban.dev`

```bash
mkcert \
  -key-file ./www/ssl/shadowban.dev/key.pem \
  -cert-file ./www/ssl/shadowban.dev/cert.pem \
  shadowban.dev *.shadowban.dev

# or use ./manage
./manage mkcert shadowban.dev
```

You can use any other way to generate your cert/key pair. Just copy both .pem
files somewhere into `./www/ssl/` and reference them in your host config.

The server is already set up for `shadowban.dev`. The next section describes how
to add new domains.

### Server

*Suppose you want the PWA to respond on `shadow-ban.dev`.*

Start by generating the SSL certificate and key

```bash
./manage mkcert shadow-ban.dev
```

The path `./www/ssl` is mounted to `/etc/nginx/ssl` and
can be referenced in configs relative to `/etc/nginx`.

For our example:

```bash
# The certificate you just created...
<project_root>/docker/www/ssl/shadow-ban.dev/cert.pem

# is mounted inside the container...
/etc/nginx/ssl/shadow-ban.dev/cert.pem

# and can be used in host configs with a relative path
ssl_certificate ssl/shadow-ban.dev/cert.pem;
```

Now copy `./www/sites-enabled/shadowban.dev` to `./www/sites-enabled/shadow-ban.dev`
and change the paths to `ssl_certificate` and `_key`, as well as `$DOMAIN` and `$server_name`.

```nginx
  server {
    set $DOMAIN 'shadow-ban.dev';
    # name and port of the frontend container
    set $PWA_ADDRESS 'shadowban-pwa';
    set $PWA_PORT '3000';

    server_name shadow-ban.dev;
    listen 80;
    listen 443 ssl; # remove the default_server, here
    ssl_certificate ssl/shadow-ban.dev/cert.pem;
    ssl_certificate_key ssl/shadow-ban.dev/key.pem;

    ...
  }
```

All that's left is to reload the nginx configs

```bash
./manage exec www nginx -s reload
```

### Client

***No need to do any of this, if you are using [mkcert](#mkcert) and the installation didn't fail.***

Go to your browser settings (e.g. brave://settings/certificates) and add the mkcert `rootCA.pem`
to your "Authorities". You can get its location with

```bash
mkcert -CAROOT
```

When asked, select the "identifying websites" option. The mkcert root certificate - which you
use to sign the certificates for domains - is now in the list, named "org-mkcert development CA".
From now on, all certificates generated with mkcert are accepted by your browser.

Also have a look at the [advanced topics](https://github.com/FiloSottile/mkcert#advanced-topics) in the mkcert README!

### Known Issues

#### net::ERR_BLOCKED_BY_CLIENT
If, despite the browser seeing a valid certificate, the page is blank and all other requests are blocked with a `net::ERR_BLOCKED_BY_CLIENT` error, try disabling all ad blocking extensions! Brave's built-in "Shields" for example blocks everything.
