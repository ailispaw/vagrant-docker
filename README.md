![docker](https://dab1nmslvvntp.cloudfront.net/wp-content/uploads/2015/04/1429543497dockerimg.png)

# A lightweight Docker VM

## Getting started

### Requirements

* Brew
* Brew Cask
* Virtualbox
* Vagrant
* Docker
* Docker Compose

### Install Requirements

#### Vagrant and Virtualbox

```
brew cask install vagrant
brew cask install virtualbox
```

#### Docker

```
brew install docker 
brew install docker-compose --ignore-dependencies 
```

### Vagrant up

```
git clone -b dev https://github.com/ailispaw/vagrant-docker.git
cd vagrant-docker
vagrant up
```

An issue with port-forwarding, reload to fix this.

```
vagrant reload
```

### Flush DNS cache

```
sudo dscacheutil -flushcache;sudo killall -HUP mDNSResponder
```

### Reboot

You may need to reboot your device to ensure `/etc/resolver/docker` is being used.

### Add the `DOCKER_HOST` env var

```
export DOCKER_HOST=tcp://localhost:2375
```

You can add this in `.bashrc` or any file that runs every interactive shell launch.


## Test Docker

```
docker ps
```

When the following line appears:

```
CONTAINER ID        IMAGE                 COMMAND             CREATED             STATUS              PORTS                    NAMES
153375aa33fe        portainer/portainer   "/portainer"        11 seconds ago      Up 11 seconds       0.0.0.0:9000->9000/tcp   portainer-portainer
f612ef7a8721        ailispaw/dnsdock      "dnsdock"           30 seconds ago      Up 29 seconds       0.0.0.0:53->53/udp       dnsdock
```

You can now visit `http://dnsdock.docker/services`.

## Install Docker Compose

```
brew install docker-compose
```


## Example docker-compose.yml for Symfony development in PHP 7.1.

```yml
version: '3'

services:
    nginx:
        image: yappabe/nginx:1.9
        volumes:
            - ./docker/shared/:/shared
            - ./:/var/www/app
        depends_on:
            - php
        env_file: 'app/config/.env'
        environment:
            DNSDOCK_ALIAS: project.docker, admin.project.docker

    mysql:
        image: mariadb:10
        env_file: 'app/config/.env'
        environment:
            DNSDOCK_ALIAS: mysql.project.docker

    php:
        image: yappabe/php:7.1
        volumes:
            - ./docker/shared/:/shared
            - ./:/var/www/app
            - vendor:/vendor
        links:
            - mysql
        working_dir: /var/www/app
        env_file: 'app/config/.env'
        environment:
            - HISTFILE=/shared/.bash_history
        depends_on:
            - mysql
            - mailcatcher

    mailcatcher:
        image: yappabe/mailcatcher
        environment:
            DNSDOCK_ALIAS: mailcatcher.project.docker

volumes:
  mysql-data:
  vendor:
```

And `app/config/.env`

```
APP_ENV=dev
APP_DEBUG=1
DOCUMENT_ROOT=/var/www/app/web
INDEX_FILE=app_dev.php
PHP_FPM_SOCKET=php:9000
MYSQL_ROOT_PASSWORD=dev
MYSQL_DATABASE=project
MYSQL_USER=root
MYSQL_HOST=mysql
MYSQL_PORT=3306
PHP_FPM_USER=root
PHP_ERROR_REPORTING=E_ALL
HISTFILE=/shared/.bash_history
MAILER_TRANSPORT=smtp
MAILER_HOST=mailcatcher
MAILER_USER=null
MAILER_PASSWORD=null
MAILER_PORT=1025
MAILER_SECURITY=null
```

```
docker-compose up -d
```

## Troubleshoot

[Troubleshoot issues](/docs/troubleshoot.md)
