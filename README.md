# Deploy Keycloak to Dokku

This repository deploys the [Keycloak](https://www.keycloak.org) Identity and Access Manangement Solution 
to Dokku.  It is based on Keycloak's official docker image with some slight modifications to use the
Heroku variable for `PORT` and `DATABASE_URL` properly.

## Prerequisites

The instructions and examples below are targeted at an audience looking to use Keycloak on a production(ish)
installation of Dokku requiring HTTPS with a properly signed certificate from [Let's Encrypt](https://letsencrypt.org/).

**Dokku Installation Prerequisites**

- A host with at least 2GB of RAM. (Keycloak is a Java beast!)
- [Dokku 0.24.7](https://dokku.com/docs~v0.24.7/getting-started/installation/) or greater installed.
- Dokku hostname configured to match the associated domain name.
    - The fake "example.com" will be used in examples below.
- Dokku configured to utilize virtual hosts (vhosts) for apps.
- A sense of joy!

## Install Dokku Plugins

All of the commands in this section need to be **executed on the Dokku host machine**!

**PostgreSQL**

```shell
dokku plugin:install https://github.com/dokku/dokku-postgres.git
```

**Let's Encrypt**

```shell
dokku plugin:install https://github.com/dokku/dokku-letsencrypt.git
```

## Create and Configure Keycloak App

You can name the Keycloak app and database anything you like, but for this example we'll
stick with "keycloak" to keep things simple.

All of the commands in this section need to be **executed on the Dokku host machine**!

**Create App**

```shell
dokku apps:create keycloak
```

**Create PostgreSQL Database**

```shell
dokku postgres:create keycloakdb
```

**Link Database to App**

```
dokku postgres:link keycloakdb keycloak
```

**Set Keycloak Admin Credentials**

Provide a strong password for the Keycloak admin account.

```shell
dokku config:set --no-restart keycloak KEYCLOAK_USER=admin KEYCLOAK_PASSWORD=strongpassword
```

**Enable Keycloak Proxy Forwarding**

Enable [proxy forwarding in order for Keycloak](https://stackoverflow.com/questions/44624844/configure-reverse-proxy-for-keycloak-docker-with-custom-base-url#44627360) to work correctly behind the [Nginx reverse proxy](https://dokku.com/docs~v0.24.7/configuration/nginx/).

```shell
dokku config:set --no-restart keycloak PROXY_ADDRESS_FORWARDING=true
```

**Configure Keycloak Hostname**

The hostname must match the name of the Dokku vhost app and domain name!

```shell
dokku config:set --no-restart keycloak KEYCLOAK_HOSTNAME=keycloak.example.com
dokku domains:add keycloak keycloak.example.com
```
> Note: you may want to remove the default dokku vhost for this app - run `dokku domains:report keycloak` to see all registered and `dokku domains:remove keycloak vhost.example.com` to remove it.

**Configure Keycloak Port and Proxy Map**

```shell
dokku config:set --no-restart keycloak KEYCLOAK_HTTP_PORT=80
dokku proxy:ports-add keycloak http:80:80
```

## Deploy Keycloak and Verify

Before fully enabling HTTPS with a signed certificate from Let's Encrypt, the Dokku app must be
deployed and tested.

All of the commands in this section need to be **executed on your local machine**!

**Clone "keycloak-dokku" Repo**

```shell
git clone https://github.com/cowofevil/keycloak-dokku.git
cd keycloak-dokku
git checkout -b refactor origin/refactor
```

**Add Git Remote to Dokku App**

You can name the remote anything like, but we'll be using "dokku" to keep things simple.

```shell
git remote add dokku dokku@example.com:keycloak
```

**Deploy "keycloak-dokku"**

Deployment can take a while the first time.

```shell
git push dokku refactor:master
```

**Verify Deployment**

Keycloak is a JBoss Java app which means it has a long startup time. You may need to wait as long
as 5 minutes for the service to be ready. Execute the following command on the **on the Dokku host machine**
to check progress.

```shell
dokku logs -t keycloak
```

Once the Keycloak deployment has completed, verify that the service is accessible by navigating to
`http://keycloak.example.com/auth/admin` in your browser. **DO NOT LOGIN! THIS IS AN UNSAFE HTTP
CONNECTION!** Just verify that the login screen is accessible.

## Create SSL Certificate and Enable HTTPS

All of the commands in this section need to be **executed on the Dokku host machine**!

```shell
dokku config:set --no-restart keycloak DOKKU_LETSENCRYPT_EMAIL=user@example.com
dokku letsencrypt:enable keycloak
```

## Login to Keycloak!

Navigate to `https://keycloak.example.com/auth/admin` and login with your admin credentials!
