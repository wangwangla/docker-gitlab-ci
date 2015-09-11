# sameersbn/gitlab-ci:7.14.3

- [Introduction](#introduction)
    - [Version](#version)
    - [Changelog](Changelog.md)
- [Contributing](#contributing)
- [Issues](#issues)
- [Announcements](https://github.com/sameersbn/docker-gitlab-ci/issues/1)
- [Installation](#installation)
- [Quick Start](#quick-start)
- [Configuration](#configuration)
    - [Persistence](#persistence)
    - [Database](#database)
        - [PostgreSQL (Recommended)](#postgresql)
            - [External PostgreSQL Server](#external-postgresql-server)
            - [Linking to PostgreSQL Container](#linking-to-postgresql-container)
        - [MySQL](#mysql)
            - [Internal MySQL Server](#internal-mysql-server)
            - [External MySQL Server](#external-mysql-server)
            - [Linking to MySQL Container](#linking-to-mysql-container)
    - [Redis](#redis)
        - [Internal Redis Server](#internal-redis-server)
        - [External Redis Server](#external-redis-server)
        - [Linking to Redis Container](#linking-to-redis-container)
    - [Mail](#mail)
    - [SSL](#ssl)
        - [Generation of Self Signed Certificates](#generation-of-self-signed-certificates)
        - [Strengthening the server security](#strengthening-the-server-security)
        - [Installation of the Certificates](#installation-of-the-certificates)
        - [Enabling HTTPS support](#enabling-https-support)
        - [Configuring HSTS](#configuring-hsts)
        - [Using HTTPS with a load balancer](#using-https-with-a-load-balancer)
        - [Establishing trust with your server](#establishing-trust-with-your-server)
        - [Installing Trusted SSL Server Certificates](#installing-trusted-ssl-server-certificates)
    - [Deploy to a subdirectory (relative url root)](#deploy-to-a-subdirectory-relative-url-root)
    - [Available Configuration Parameters](#available-configuration-parameters)
- [Maintenance](#maintenance)
    - [Creating Backups](#creating-backups)
    - [Restoring Backups](#restoring-backups)
    - [Automated Backups](#automated-backups)
        - [Remote Backups (AWS)](#remote-backups-aws)
    - [Upgrading](#upgrading)
    - [Shell Access](#shell-access)
- [References](#references)

# Introduction

Dockerfile to build a [GitLab CI](https://about.gitlab.com/gitlab-ci/) container image.

## Version

Current Version: `7.14.3`

*The gitlab-ci version should match the gitlab version to avoid any compatibility issues*

# Contributing

If you find this image useful here's how you can help:

- Send a Pull Request with your awesome new features and bug fixes
- Help new users with [Issues](https://github.com/sameersbn/docker-gitlab-ci/issues) they may encounter
- Support the development of this image with a [donation](http://www.damagehead.com/donate/)

# Issues

Docker is a relatively new project and is active being developed and tested by a thriving community of developers and testers and every release of docker features many enhancements and bugfixes.

Given the nature of the development and release cycle it is very important that you have the latest version of docker installed because any issue that you encounter might have already been fixed with a newer docker release.

For ubuntu users I suggest [installing docker](https://docs.docker.com/installation/ubuntulinux/) using docker's own package repository since the version of docker packaged in the ubuntu repositories are a little dated.

Here is the shortform of the installation of an updated version of docker on ubuntu.

```bash
sudo apt-get purge docker.io
curl -s https://get.docker.io/ubuntu/ | sudo sh
sudo apt-get update
sudo apt-get install lxc-docker
```

Fedora and RHEL/CentOS users should try disabling selinux with `setenforce 0` and check if resolves the issue. If it does than there is not much that I can help you with. You can either stick with selinux disabled (not recommended by redhat) or switch to using ubuntu.

If using the latest docker version and/or disabling selinux does not fix the issue then please file a issue request on the [issues](https://github.com/sameersbn/docker-gitlab-ci/issues) page.

In your issue report please make sure you provide the following information:

- The host ditribution and release version.
- Output of the `docker version` command
- Output of the `docker info` command
- The `docker run` command you used to run the image (mask out the sensitive bits).

# Installation

Pull the latest version of the image from the docker index. This is the recommended method of installation as it is easier to update image in the future. These builds are performed by the **Docker Trusted Build** service.

```bash
docker pull sameersbn/gitlab-ci:latest
```

Starting from GitLab CI version `5.1.0`, You can pull a particular version of GitLab CI by specifying the version number. For example,

```bash
docker pull sameersbn/gitlab-ci:7.14.3
```

Alternately you can build the image yourself.

```bash
git clone https://github.com/sameersbn/docker-gitlab-ci.git
cd docker-gitlab-ci
docker build --tag $USER/gitlab-ci .
```

# Quick Start

Before you can start the GitLab CI image you need to make sure you have a [GitLab](https://www.gitlab.com/) server running. Checkout the [docker-gitlab](https://github.com/sameersbn/docker-gitlab) project for getting a GitLab server up and running.

You need to provide the URL of the GitLab server while running GitLab CI using the `GITLAB_URL` environment configuration. Since version `5.4.0` you need to provide the `GITLAB_APP_ID` and `GITLAB_APP_SECRET`. Since `7.14.3` you also need to provide the `GITLAB_CI_SECRETS_SESSION_KEY_BASE` and `GITLAB_CI_SECRETS_DB_KEY_BASE` parameters.

The quickest way to get started is using [docker-compose](https://docs.docker.com/compose/).

```bash
wget https://raw.githubusercontent.com/sameersbn/docker-gitlab-ci/master/docker-compose.yml
docker-compose up
```

Alternately, you can manually launch the `gitlab-ci` container and the supporting `postgresql` and `redis` containers by following this three step guide.

Step 1. Launch a postgresql container

```bash
docker run --name gitlab-ci-postgresql -d \
    --env 'DB_NAME=gitlab_ci_production' \
    --env 'DB_USER=gitlab' --env 'DB_PASS=password' \
    --volume /srv/docker/gitlab-ci/postgresql:/var/lib/postgresql \
    sameersbn/postgresql:9.4-3
```

Step 2. Launch a redis container

```bash
docker run --name gitlab-ci-redis -d \
    --volume /srv/docker/gitlab-ci/redis:/var/lib/redis \
    sameersbn/redis:latest
```

Step 3. Launch the gitlab-ci container

```bash
docker run --name gitlab-ci -d \
    --link gitlab-ci-postgresql:postgresql --link gitlab-ci-redis:redisio \
    --publish 10081:80 \
    --env 'GITLAB_CI_PORT=10081' --env 'GITLAB_URL=http://localhost:10080' \
    --env 'GITLAB_APP_ID=xxx' --env 'GITLAB_APP_SECRET=yyy' \
    --volume /srv/docker/gitlab-ci/gitlab-ci:/home/gitlab_ci/data \
    sameersbn/gitlab-ci:7.14.3
```

Point your browser to `http://localhost:10081` and login using your GitLab credentials.

You should now have the GitLab CI ready for testing. If you want to use this image in production the please read on.

*The rest of the document will use the docker command line. You can quite simply adapt your configuration into a `docker-compose.yml` file if you wish to do so.*

**NOTE:** You need to install [GitLab CI Runner](https://gitlab.com/gitlab-org/gitlab-ci-runner/blob/master/README.md) if you want to do anything worth while with the GitLab CI server. Please take a look at [docker-gitlab-ci-runner](https://github.com/sameersbn/docker-gitlab-ci-runner) for a base runner image.

# Configuration

## Persistence

For storage of the application data, you should mount a volume at

* `/home/gitlab_ci/data`

SELinux users are also required to change the security context of the mount point so that it plays nicely with selinux.

```bash
mkdir -p /srv/docker/gitlab-ci/gitlab-ci
sudo chcon -Rt svirt_sandbox_file_t /srv/docker/gitlab-ci/gitlab-ci
```

Volumes can be mounted in docker by specifying the **'-v'** option in the docker run command.

```bash
docker run --name gitlab-ci -it --rm \
    --volume /srv/docker/gitlab-ci/gitlab-ci:/home/gitlab_ci/data \
    sameersbn/gitlab-ci:7.14.3
```

## Database

GitLab CI uses a database backend to store its data.

### PostgreSQL

#### External PostgreSQL Server

The image also supports using an external PostgreSQL Server. This is also controlled via environment variables.

```sql
CREATE ROLE gitlab_ci with LOGIN CREATEDB PASSWORD 'password';
CREATE DATABASE gitlab_ci_production;
GRANT ALL PRIVILEGES ON DATABASE gitlab_ci_production to gitlab_ci;
```

We are now ready to start the container.

*Assuming that the PostgreSQL server host is 192.168.1.100*

```bash
docker run --name gitlab-ci -it --rm \
    --env 'GITLAB_URL=http://172.17.0.2' \
    --env 'GITLAB_APP_ID=xxx' --env 'GITLAB_APP_SECRET=yyy' \
    --env 'DB_TYPE=postgres' --env 'DB_HOST=192.168.1.100' \
    --env 'DB_NAME=gitlab_ci_production' \
    --env 'DB_USER=gitlab_ci' --env 'DB_PASS=password' \
    sameersbn/gitlab-ci:7.14.3
```

#### Linking to PostgreSQL Container

You can link this image with a postgresql container for the database requirements. The alias of the postgresql server container should be set to **postgresql** while linking with the gitlab ci image.

If a postgresql container is linked, only the `DB_TYPE`, `DB_HOST` and `DB_PORT` settings are automatically retrieved using the linkage. You may still need to set other database connection parameters such as the `DB_NAME`, `DB_USER`, `DB_PASS` and so on.

To illustrate linking with a postgresql container, we will use the [sameersbn/postgresql](https://github.com/sameersbn/docker-postgresql) image. When using postgresql image in production you should mount a volume for the postgresql data store. Please refer the [README](https://github.com/sameersbn/docker-postgresql/blob/master/README.md) of docker-postgresql for details.

First, lets pull the postgresql image from the docker index.

```bash
docker pull sameersbn/postgresql:9.4-3
```

For data persistence lets create a store for the postgresql and start the container.

SELinux users are also required to change the security context of the mount point so that it plays nicely with selinux.

```bash
mkdir -p /srv/docker/gitlab-ci/postgresql
sudo chcon -Rt svirt_sandbox_file_t /srv/docker/gitlab-ci/postgresql
```

The run command looks like this.

```bash
docker run --name gitlab-ci-postgresql -d \
    --env 'DB_NAME=gitlab_ci_production' \
    --env 'DB_USER=gitlab_ci' --env 'DB_PASS=password' \
    --volume /srv/docker/gitlab-ci/postgresql:/var/lib/postgresql \
    sameersbn/postgresql:9.4-3
```

The above command will create a database named `gitlab_ci_production` and also create a user named `gitlab_ci` with the password `password` with access to the `gitlab_ci_production` database.

We are now ready to start the GitLab CI application.

```bash
docker run --name gitlab-ci -it --rm --link gitlab-ci-postgresql:postgresql \
    --env 'GITLAB_URL=http://172.17.0.2' \
    --env 'GITLAB_APP_ID=xxx' --env 'GITLAB_APP_SECRET=yyy' \
    sameersbn/gitlab-ci:7.14.3
```

Here the image will also automatically fetch the `DB_NAME`, `DB_USER` and `DB_PASS` variables from the postgresql container as they are specified in the `docker run` command for the postgresql container. This is made possible using the magic of docker links and works with the following images:

 - [postgres](https://hub.docker.com/_/postgres/)
 - [sameersbn/postgresql](https://hub.docker.com/r/sameersbn/postgresql/)
 - [orchardup/postgresql](https://hub.docker.com/r/orchardup/postgresql/)
 - [paintedfox/postgresql](https://hub.docker.com/r/paintedfox/postgresql/)

### MySQL

#### Internal MySQL Server

The internal mysql server has been removed from the image. Please use a [linked mysql](#linking-to-mysql-container) container or specify a connection to a [external mysql](#external-mysql-server) server.

If you have been using the internal mysql server follow these instructions to migrate to a linked mysql container:

Assuming that your mysql data is available at `/srv/docker/gitlab-ci/mysql`

```bash
docker run --name gitlab-ci-mysql -d \
    --volume /srv/docker/gitlab-ci/mysql:/var/lib/mysql \
    sameersbn/mysql:latest
```

This will start a mysql container with your existing mysql data. Now login to the mysql container and create a user for the existing `gitlab_ci_production` database.

All you need to do now is link this mysql container to the gitlab ci container using the `--link gitlab-ci-mysql:mysql` option and provide the `DB_NAME`, `DB_USER` and `DB_PASS` parameters.

Refer to [Linking to MySQL Container](#linking-to-mysql-container) for more information.

#### External MySQL Server

The image can be configured to use an external MySQL database instead of starting a MySQL server internally. The database configuration should be specified using environment variables while starting the GitLab CI image.

Before you start the GitLab CI image create user and database for GitLab CI.

```sql
CREATE USER 'gitlab_ci'@'%.%.%.%' IDENTIFIED BY 'password';
CREATE DATABASE IF NOT EXISTS `gitlab_ci_production` DEFAULT CHARACTER SET `utf8` COLLATE `utf8_unicode_ci`;
GRANT ALL PRIVILEGES ON `gitlab_ci_production`.* TO 'gitlab_ci'@'%.%.%.%';
```

We are now ready to start the container.

*Assuming that the mysql server host is 192.168.1.100*

```bash
docker run --name gitlab-ci -it --rm \
    --env 'GITLAB_URL=http://172.17.0.2' \
    --env 'GITLAB_APP_ID=xxx' --env 'GITLAB_APP_SECRET=yyy' \
    --env 'DB_HOST=192.168.1.100' --env 'DB_NAME=gitlab_ci_production' \
    --env 'DB_USER=gitlab_ci' --env 'DB_PASS=password' \
    sameersbn/gitlab-ci:7.14.3
```

#### Linking to MySQL Container

You can link this image with a mysql container for the database requirements. The alias of the mysql server container should be set to **mysql** while linking with the gitlab ci image.

If a mysql container is linked, only the `DB_TYPE`, `DB_HOST` and `DB_PORT` settings are automatically retrieved using the linkage. You may still need to set other database connection parameters such as the `DB_NAME`, `DB_USER`, `DB_PASS` and so on.

To illustrate linking with a mysql container, we will use the [sameersbn/mysql](https://github.com/sameersbn/docker-mysql) image. When using docker-mysql in production you should mount a volume for the mysql data store. Please refer the [README](https://github.com/sameersbn/docker-mysql/blob/master/README.md) of docker-mysql for details.

First, lets pull the mysql image from the docker index.

```bash
docker pull sameersbn/mysql:latest
```

For data persistence lets create a store for the mysql and start the container.

SELinux users are also required to change the security context of the mount point so that it plays nicely with selinux.

```bash
mkdir -p /srv/docker/gitlab-ci/mysql
sudo chcon -Rt svirt_sandbox_file_t /srv/docker/gitlab-ci/mysql
```

The run command looks like this.

```bash
docker run --name gitlab-ci-mysql -d \
    --env 'DB_NAME=gitlab_ci_production' \
    --env 'DB_USER=gitlab_ci' --env 'DB_PASS=password' \
    --volume /srv/docker/gitlab-ci/mysql:/var/lib/mysql \
    sameersbn/mysql:latest
```

The above command will create a database named `gitlab_ci_production` and also create a user named `gitlab_ci` with the password `password` with full/remote access to the `gitlab_ci_production` database.

We are now ready to start the GitLab CI application.

```bash
docker run --name gitlab-ci -it --rm --link gitlab-ci-mysql:mysql \
    --env 'GITLAB_URL=http://172.17.0.2' \
    --env 'GITLAB_APP_ID=xxx' --env 'GITLAB_APP_SECRET=yyy' \
    sameersbn/gitlab-ci:7.14.3
```

Here the image will also automatically fetch the `DB_NAME`, `DB_USER` and `DB_PASS` variables from the mysql container as they are specified in the `docker run` command for the mysql container. This is made possible using the magic of docker links and works with the following images:

 - [mysql](https://hub.docker.com/_/mysql/)
 - [sameersbn/mysql](https://hub.docker.com/r/sameersbn/mysql/)
 - [centurylink/mysql](https://hub.docker.com/r/centurylink/mysql/)
 - [orchardup/mysql](https://hub.docker.com/r/orchardup/mysql/)

## Redis

GitLab CI uses the redis server for its key-value data store. The redis server connection details can be specified using environment variables.

### Internal Redis Server

The internal redis server has been removed from the image. Please use a [linked redis](#linking-to-redis-container) container or specify a [external redis](#external-redis-server) connection.

### External Redis Server

The image can be configured to use an external redis server instead of starting a redis server internally. The configuration should be specified using environment variables while starting the GitLab CI image.

*Assuming that the redis server host is 192.168.1.100*

```bash
docker run --name gitlab-ci -it --rm \
    --env 'REDIS_HOST=192.168.1.100' --env 'REDIS_PORT=6379' \
    sameersbn/gitlab-ci:7.14.3
```

### Linking to Redis Container

You can link this image with a redis container to satisfy GitLab CI's redis requirement. The alias of the redis server container should be set to **redisio** while linking with the GitLab image.

To illustrate linking with a redis container, we will use the [sameersbn/redis](https://github.com/sameersbn/docker-redis) image. Please refer the [README](https://github.com/sameersbn/docker-redis/blob/master/README.md) of docker-redis for details.

First, lets pull the redis image from the docker index.

```bash
docker pull sameersbn/redis:latest
```

Lets start the redis container

```bash
docker run --name gitlab-ci-redis -d sameersbn/redis:latest
```

We are now ready to start the GitLab CI application.

```bash
docker run --name gitlab-ci -it --rm --link gitlab-ci-redis:redisio \
    sameersbn/gitlab-ci:latest
```

### Mail

The mail configuration should be specified using environment variables while starting the GitLab CI image. The configuration defaults to using gmail to send emails and requires the specification of a valid username and password to login to the gmail servers.

Please refer the [Available Configuration Parameters](#available-configuration-parameters) section for the list of SMTP parameters that can be specified.

```bash
docker run --name gitlab-ci -it --rm \
    --env 'SMTP_USER=USER@gmail.com' --env 'SMTP_PASS=PASSWORD' \
    sameersbn/gitlab-ci:7.14.3
```

Please look up the [Available Configuration Parameters](#available-configuration-parameters) section for all available SMTP configuration options.

### SSL

Access to the gitlab ci application can be secured using SSL so as to prevent unauthorized access to the data in your repositories. While a CA certified SSL certificate allows for verification of trust via the CA, a self signed certificates can also provide an equal level of trust verification as long as each client takes some additional steps to verify the identity of your website. I will provide instructions on achieving this towards the end of this section.

To secure your application via SSL you basically need two things:
- **Private key (.key)**
- **SSL certificate (.crt)**

When using CA certified certificates, these files are provided to you by the CA. When using self-signed certificates you need to generate these files yourself. Skip the following section if you are armed with CA certified SSL certificates.

Jump to the [Using HTTPS with a load balancer](#using-https-with-a-load-balancer) section if you are using a load balancer such as hipache, haproxy or nginx.

#### Generation of Self Signed Certificates

Generation of self-signed SSL certificates involves a simple 3 step procedure.

**STEP 1**: Create the server private key

```bash
openssl genrsa -out gitlab_ci.key 2048
```

**STEP 2**: Create the certificate signing request (CSR)

```bash
openssl req -new -key gitlab_ci.key -out gitlab_ci.csr
```

**STEP 3**: Sign the certificate using the private key and CSR

```bash
openssl x509 -req -days 3650 \
    -in gitlab_ci.csr -signkey gitlab_ci.key \
    -out gitlab_ci.crt
```

Congratulations! you have now generated an SSL certificate thats valid for 365 days.

#### Strengthening the server security

This section provides you with instructions to [strengthen your server security](https://raymii.org/s/tutorials/Strong_SSL_Security_On_nginx.html). To achieve this we need to generate stronger DHE parameters.

```bash
openssl dhparam -out dhparam.pem 2048
```

#### Installation of the SSL Certificates

Out of the four files generated above, we need to install the `gitlab_ci.key`, `gitlab_ci.crt` and `dhparam.pem` files at the gitlab ci server. The CSR file is not needed, but do make sure you safely backup the file (in case you ever need it again).

The default path that the gitlab ci application is configured to look for the SSL certificates is at `/home/gitlab_ci/data/certs`, this can however be changed using the `SSL_KEY_PATH`, `SSL_CERTIFICATE_PATH` and `SSL_DHPARAM_PATH` configuration options.

If you remember from above, the `/home/gitlab_ci/data` path is the path of the persistent volume, which means that we have to create a folder named `certs` inside `/srv/docker/gitlab-ci/gitlab-ci` and copy the files into it and as a measure of security we will update the permission on the `gitlab_ci.key` file to only be readable by the owner.

```bash
mkdir -p /srv/docker/gitlab-ci/gitlab-ci/certs
cp gitlab_ci.key /srv/docker/gitlab-ci/gitlab-ci/certs/
cp gitlab_ci.crt /srv/docker/gitlab-ci/gitlab-ci/certs/
cp dhparam.pem /srv/docker/gitlab-ci/gitlab-ci/certs/
chmod 400 /srv/docker/gitlab-ci/gitlab-ci/certs/gitlab_ci.key
```

Great! we are now just one step away from having our application secured.

#### Enabling HTTPS support

HTTPS support can be enabled by setting the `GITLAB_CI_HTTPS` option to `true`.

```bash
docker run --name gitlab-ci -it --rm \
    --publish 10081:80 --publish 10444:443 \
    --env 'GITLAB_CI_PORT=10444' -env 'GITLAB_CI_HTTPS=true' \
    --volume /srv/docker/gitlab-ci/gitlab-ci:/home/gitlab_ci/data \
    sameersbn/gitlab-ci:7.14.3
```

In this configuration, any requests made over the plain http protocol will automatically be redirected to use the https protocol. However, this is not optimal when using a load balancer.

#### Configuring HSTS

HSTS if supported by the browsers makes sure that your users will only reach your sever via HTTPS. When the user comes for the first time it sees a header from the server which states for how long from now this site should only be reachable via HTTPS - that's the HSTS max-age value.

With `GITLAB_CI_HTTPS_HSTS_MAXAGE` you can configure that value. The default value is `31536000` seconds. If you want to disable a already sent HSTS MAXAGE value, set it to `0`.

```bash
docker run --name gitlab-ci -it --rm \
    --env 'GITLAB_CI_HTTPS=true' \
    --env 'GITLAB_CI_HTTPS_HSTS_MAXAGE=2592000' \
    --volume /srv/docker/gitlab-ci/gitlab-ci:/home/gitlab_ci/data \
    sameersbn/gitlab-ci:7.14.3
```

If you want to completely disable HSTS set `GITLAB_CI_HTTPS_HSTS_ENABLED` to `false`.

#### Using HTTPS with a load balancer

Load balancers like nginx/haproxy/hipache talk to backend applications over plain http and as such the installation of ssl keys and certificates are not required and should **NOT** be installed in the container. The SSL configuration has to instead be done at the load balancer. Hoewever, when using a load balancer you **MUST** set `GITLAB_CI_HTTPS` to `true`.

With this in place, you should configure the load balancer to support handling of https requests. But that is out of the scope of this document. Please refer to [Using SSL/HTTPS with HAProxy](http://seanmcgary.com/posts/using-sslhttps-with-haproxy) for information on the subject.

When using a load balancer, you probably want to make sure the load balancer performs the automatic http to https redirection. Information on this can also be found in the link above.

In summation, when using a load balancer, the docker command would look for the most part something like this:

```bash
docker run --name gitlab-ci -it --rm \
    --env 'GITLAB_CI_HTTPS=true' \
    --volume /srv/docker/gitlab-ci/gitlab-ci:/home/gitlab_ci/data \
    sameersbn/gitlab-ci:7.14.3
```

#### Establishing trust with your server

This section deals will self-signed ssl certificates. If you are using CA certified certificates, your done.

This section is more of a client side configuration so as to add a level of confidence at the client to be 100 percent sure they are communicating with whom they think they.

This is simply done by adding the servers certificate into their list of trusted ceritficates. On ubuntu, this is done by copying the `gitlab_ci.crt` file to `/usr/local/share/ca-certificates/` and executing `update-ca-certificates`.

Again, this is a client side configuration which means that everyone who is going to communicate with the server should perform this configuration on their machine. In short, distribute the `gitlab_ci.crt` file among your developers and ask them to add it to their list of trusted ssl certificates.

You can do the same at the web browser. Instructions for installing the root certificate for firefox can be found [here](http://portal.threatpulse.com/docs/sol/Content/03Solutions/ManagePolicy/SSL/ssl_firefox_cert_ta.htm). You will find similar options chrome, just make sure you install the certificate under the authorities tab of the certificate manager dialog.

There you have it, thats all there is to it.

#### Installing Trusted SSL Server Certificates

If your GitLab server is using self-signed SSL certificates then you should make sure the GitLab server certificate is trusted on the GitLab CI server for them to be able to talk to each other.

The default path image is configured to look for the trusted SSL certificates is at `/home/gitlab_ci/data/certs/ca.crt`, this can however be changed using the `CA_CERTIFICATES_PATH` configuration option.

Copy the `ca.crt` file into a `certs` directory in the [persistent](#persistence). The `ca.crt` file should contain the root certificates of all the servers you want to trust. With respect to GitLab, this will be the contents of the `gitlab_ci.crt` file as described in the [README](https://github.com/sameersbn/docker-gitlab/blob/master/README.md#ssl) of the [docker-gitlab](https://github.com/sameersbn/docker-gitlab) container.

By default, our own server certificate [gitlab_ci.crt](#generation-of-self-signed-certificates) is added to the trusted certificates list.

### Deploy to a subdirectory (relative url root)

By default GitLab CI expects that your application is running at the root (eg. /). This section explains how to run your application inside a directory.

Let's assume we want to deploy our application to '/ci'. GitLab CI needs to know this directory to generate the appropriate routes. This can be specified using the `GITLAB_CI_RELATIVE_URL_ROOT` configuration option like so:

```bash
docker run --name gitlab-ci -it --rm \
    --env 'GITLAB_CI_RELATIVE_URL_ROOT=/ci' \
    --volume /srv/docker/gitlab-ci/gitlab-ci:/home/gitlab_ci/data \
    sameersbn/gitlab-ci:7.14.3
```

GitLab CI will now be accessible at the `/ci` path, e.g. `http://www.example.com/ci`.

**Note**: *The `GITLAB_CI_RELATIVE_URL_ROOT` parameter should always begin with a slash and* **SHOULD NOT** *have any trailing slashes.*

### Available Configuration Parameters

*Please refer the docker run command options for the `--env-file` flag where you can specify all required environment variables in a single file. This will save you from writing a potentially long docker run command. Alternately you can use docker-compose.*

Below is the complete list of available options that can be used to customize your GitLab CI installation.

- **GITLAB_URL**: Url of the GitLab server to allow connections from. No defaults. Automatically configured when a GitLab server is linked using docker links feature.
- **GITLAB_CI_SECRETS_SESSION_KEY_BASE**: Used to verify the integrity of signed cookies. You can generated one using `pwgen -Bsv1 64`.
- **GITLAB_CI_SECRETS_DB_KEY_BASE**: Used to encrypt variables. Ensure that you don't lose it. You can generate one using `pwgen -Bsv1 64`.
- **GITLAB_APP_ID**: Application ID used to connect to the gitlab server.
- **GITLAB_APP_SECRET**: Secret key used to connect to the gitlab server.
- **GITLAB_CI_HOST**: The hostname of the GitLab CI server. Defaults to `localhost`.
- **GITLAB_CI_PORT**: The port number of the GitLab CI server. Defaults to `80` for plain http and `443` when https is enabled.
- **GITLAB_CI_TIMEZONE**: Configure the timezone for the GitLab CI application. Defaults to `UTC`.
- **GITLAB_CI_EMAIL**: The email address for the GitLab CI server. Defaults to `gitlab@localhost`.
- **GITLAB_CI_SUPPORT**: The support email address for the GitLab CI server. Defaults to `support@localhost`.
- **GITLAB_CI_NOTIFY_ON_BROKEN_BUILDS**: Enable or disable broken build notification emails. Defaults to `true`
- **GITLAB_CI_NOTIFY_PUSHER**: Add pusher to recipients list of broken build notification emails. Defaults to `false`
- **GITLAB_CI_HTTPS**: Set to `true` to enable https support, disabled by default.
- **GITLAB_CI_HTTPS_HSTS_ENABLED**: Advanced configuration option for turning off the HSTS configuration. Applicable only when SSL is in use. Defaults to `true`. See [#138](https://github.com/sameersbn/docker-gitlab/issues/138) for use case scenario.
- **GITLAB_CI_HTTPS_HSTS_MAXAGE**: Advanced configuration option for setting the HSTS max-age in the gitlab-ci nginx vHost configuration. Applicable only when SSL is in use. Defaults to `31536000`.
- **GITLAB_CI_RELATIVE_URL_ROOT**: The sub URI of the GitLab CI server, e.g. `/ci`. No default.
- **GITLAB_CI_BACKUP_DIR**: The backup folder in the container. Defaults to `/home/gitlab_ci/data/backups`
- **GITLAB_CI_BACKUPS**: Setup cron job to automatic backups. Possible values `disable`, `daily`, `weekly` or `monthly`. Disabled by default
- **GITLAB_CI_BACKUP_EXPIRY**: Configure how long (in seconds) to keep backups before they are deleted. By default when automated backups are disabled backups are kept forever (0 seconds), else the backups expire in 7 days (604800 seconds).
- **GITLAB_CI_BACKUP_TIME**: Set a time for the automatic backups in `HH:MM` format. Defaults to `04:00`.
- **SSL_CERTIFICATE_PATH**: Location of the ssl certificate. Defaults to `/home/gitlab_ci/data/certs/gitlab_ci.crt`
- **SSL_KEY_PATH**: Location of the ssl private key. Defaults to `/home/gitlab_ci/data/certs/gitlab_ci.key`
- **SSL_DHPARAM_PATH**: Location of the dhparam file. Defaults to `/home/gitlab_ci/data/certs/dhparam.pem`
- **SSL_VERIFY_CLIENT**: Enable verification of client certificates using the `CA_CERTIFICATES_PATH` file. Defaults to `false`
- **NGINX_WORKERS**: The number of nginx workers to start. Defaults to `1`.
- **NGINX_MAX_UPLOAD_SIZE**: Maximum acceptable upload size. Defaults to `20m`.
- **NGINX_X_FORWARDED_PROTO**: Advanced configuration option for the `proxy_set_header X-Forwarded-Proto` setting in the gitlab-ci nginx vHost configuration. Defaults to `https` when `GITLAB_CI_HTTPS` is `true`, else defaults to `$scheme`.
- **REDIS_HOST**: The hostname of the redis server. Defaults to `localhost`
- **REDIS_PORT**: The connection port of the redis server. Defaults to `6379`.
- **UNICORN_WORKERS**: The number of unicorn workers to start. Defaults to `2`.
- **UNICORN_TIMEOUT**: Sets the timeout of unicorn worker processes. Defaults to `60` seconds.
- **SIDEKIQ_CONCURRENCY**: The number of concurrent sidekiq jobs to run. Defaults to `25`
- **DB_TYPE**: The database type. Possible values: `mysql`, `postgres`. Defaults to `mysql`.
- **DB_HOST**: The database server hostname. Defaults to `localhost`.
- **DB_PORT**: The database server port. Defaults to `3306` for mysql and `5432` for postgresql.
- **DB_NAME**: The database database name. Defaults to `gitlab_ci_production`
- **DB_USER**: The database database user. Defaults to `root`
- **DB_PASS**: The database database password. Defaults to no password
- **DB_POOL**: The database database connection pool count. Defaults to `10`.
- **SMTP_ENABLED**: Enable mail delivery via SMTP. Defaults to `true` if `SMTP_USER` is defined, else defaults to `false`.
- **SMTP_DOMAIN**: SMTP domain. Defaults to `www.gmail.com`
- **SMTP_HOST**: SMTP server host. Defaults to `smtp.gmail.com`
- **SMTP_PORT**: SMTP server port. Defaults to `587`.
- **SMTP_USER**: SMTP username.
- **SMTP_PASS**: SMTP password.
- **SMTP_OPENSSL_VERIFY_MODE**: SMTP openssl verification mode. Accepted values are `none`, `peer`, `client_once` and `fail_if_no_peer_cert`. SSL certificate verification is performed by default.
- **SMTP_STARTTLS**: Enable STARTTLS. Defaults to `true`.
- **SMTP_TLS**: Enable SSL/TLS. Defaults to `false`.
- **SMTP_AUTHENTICATION**: Specify the SMTP authentication method. Defaults to `login` if `SMTP_USER` is set.
- **SMTP_CA_ENABLED**: Enable custom CA certificates for SMTP email configuration. Defaults to `false`.
- **SMTP_CA_PATH**: Specify the `ca_path` parameter for SMTP email configuration. Defaults to `/home/gitlab_ci/data/certs`.
- **SMTP_CA_FILE**: Specify the `ca_file` parameter for SMTP email configuration. Defaults to `/home/gitlab_ci/data/certs/ca.crt`.
- **AWS_BACKUPS**: Enables automatic uploads to an Amazon S3 instance. Defaults to `false`.
- **AWS_BACKUP_REGION**: AWS region. No defaults.
- **AWS_BACKUP_ACCESS_KEY_ID**: AWS access key id. No defaults.
- **AWS_BACKUP_SECRET_ACCESS_KEY**: AWS secret access key. No defaults.
- **AWS_BACKUP_BUCKET**: AWS bucket for backup uploads. No defaults.
- **GITLAB_CI_ROBOTS_OVERRIDE**: Override `robots.txt`. Defaults to `false`.
- **GITLAB_CI_ROBOTS_PATH**: Location of `robots.txt`. See [www.robotstxt.org](http://www.robotstxt.org) for examples. Defaults to `robots.txt` which [prevents robots scanning gitlab](http://www.robotstxt.org/faq/prevent.html).

# Maintenance

## Creating Backups

Since release `7.11.0`, Gitlab CI defines a rake task to take a backup of your gitlab ci installation. The resulting backup primarily consists of the sql database.

Before taking a backup make sure the container is stopped and removed to avoid container name conflicts.

```bash
docker stop gitlab-ci && docker rm gitlab-ci
```

Execute the rake task to create a backup.

```bash
docker run --name gitlab-ci -it --rm [OPTIONS] \
    sameersbn/gitlab-ci:7.14.3 app:rake backup:create
```

A backup will be created in the backups folder of the [Persistence](#persistence). You can change the location of the backups using the `GITLAB_CI_BACKUP_DIR` configuration parameter.

*P.S. Backups can also be generated on a running instance using `docker exec` as described in the [Rake Tasks](#rake-tasks) section. However, to avoid undesired side-effects, I advice against running backup and restore operations on a running instance.*

## Restoring Backups

Since release `7.11.0`, Gitlab CI also defines a rake task to restore a backup.

Before performing a restore make sure the container is stopped and removed to avoid container name conflicts.

```bash
docker stop gitlab-ci && docker rm gitlab-ci
```

Execute the rake task to restore a backup. Make sure you run the container in interactive mode `-it`.

```bash
docker run --name gitlab-ci -it --rm [OPTIONS] \
    sameersbn/gitlab-ci:7.14.3 app:rake backup:restore
```

The list of all available backups will be displayed in reverse chronological order. Select the backup you want to restore and continue.

To avoid user interaction in the restore operation, specify the timestamp of the backup using the `BACKUP` argument to the rake task.

```bash
docker run --name gitlab-ci -it --rm [OPTIONS] \
    sameersbn/gitlab-ci:7.14.3 app:rake backup:restore BACKUP=1417624827
```

## Automated Backups

The image can be configured to automatically take backups `daily`, `weekly` or `monthly` using the `GITLAB_CI_BACKUPS` configuration option.

Daily backups are created at `GITLAB_CI_BACKUP_TIME` which defaults to `04:00` everyday. Weekly backups are created every Sunday at the same time as the daily backups. Monthly backups are created on the 1st of every month at the same time as the daily backups.

By default, when automated backups are enabled, backups are held for a period of 7 days. While when automated backups are disabled, the backups are held for an infinite period of time. This can behavior can be configured via the `GITLAB_CI_BACKUP_EXPIRY` option.

### Remote Backups (AWS)

The image can be configured to automatically upload the backups to an AWS S3 bucket. To enable automatic AWS backups first add `--env 'AWS_BACKUPS=true'` to the docker run command. In addition `AWS_BACKUP_REGION` and `AWS_BACKUP_BUCKET` must be properly configured to point to the desired AWS location. Finally an IAM user must be configured with appropriate access permission and their AWS keys exposed through `AWS_BACKUP_ACCESS_KEY_ID` and `AWS_BACKUP_SECRET_ACCESS_KEY`.

More details about the appropriate IAM user properties can found on [gitlab.com](https://gitlab.com/gitlab-org/gitlab-ci/blob/master/doc/raketasks/backup_restore.md#upload-backups-to-remote-cloud-storage)

AWS uploads are performed alongside normal backups, both through the appropriate `app:rake` command and when an automatic backup is performed.

## Upgrading

To upgrade to newer GitLab CI releases, simply follow this 3 step upgrade procedure.

- **Step 1**: Update the docker image.

```bash
docker pull sameersbn/gitlab-ci:7.14.3
```

- **Step 2**: Stop and remove the currently running image

```bash
docker stop gitlab-ci
docker rm gitlab-ci
```

- **Step 3**: Start the image

```bash
docker run --name gitlab-ci -d [OPTIONS] sameersbn/gitlab-ci:7.14.3
```

## Shell Access

For debugging and maintenance purposes you may want access the containers shell. If you are using docker version `1.3.0` or higher you can access a running containers shell using `docker exec` command.

```bash
docker exec -it gitlab-ci bash
```

# References
    * https://www.gitlab.com/gitlab-ci/
    * https://gitlab.com/gitlab-org/gitlab-ci/blob/master/README.md
