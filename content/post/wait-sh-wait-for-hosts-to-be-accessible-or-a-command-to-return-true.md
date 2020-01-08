---
title: Wait.sh — wait for hosts to be accessible or a command to return true
excerpt: wait.sh is a bash script I wrote to help me set up my docker powered projects and allow me to have a functioning build process for database…
date: 2018-09-24
hero: /img/railroad-tracks-in-city-against-sky-258458.jpg
---

[Published to Medium](https://medium.com/@adrian.gheorghe.dev/wait-sh-wait-for-hosts-to-be-accessible-or-a-command-to-return-true-7f1d64e7fae4)

[wait.sh](https://github.com/adrian-gheorghe/wait) is a bash script I wrote to help me set up my docker powered projects and allow me to have a functioning build process for database migrations.

The code is deeply inspired by [https://github.com/vishnubob/wait-for-it](https://github.com/vishnubob/wait-for-it) and [https://github.com/eficode/wait-for](https://github.com/eficode/wait-for).

The script is able to wait for a host or multiple hosts to respond on a TCP port and can also wait for a command to output a value. For example you can wait for a file to exist or contain something.

This is mainly useful to link containers that depend on one another to start. For example you can have a container that runs install scripts that will have to wait for the database to be accessible. Or you can have a process that runs a command when the files are in place.

Multiple commands can be added as parameters and will be run sequentially.

### Usage example from bash

```bash
$ ./wait.sh --wait "database_host:3306" --wait "ls -al /var/www/html | grep docker-compose.yml" --command "Database is up and files exist"
```

An Docker image containing this script can be accessed on DockerHub. and can be used either directly in your docker-compose file or as a base image in your Dockerfile

```yaml
version: '3.3'
services:  
  db:    
    image: mysql:5.7
    environment:
        MYSQL_ROOT_PASSWORD: root
        MYSQL_DATABASE: database
  wait:
    image: adighe/wait
    command: "./wait.sh --wait \"db:3306\" --command \"ls -al\""
```

The adighe/wait docker image can be used as a base image to add the wait.sh file to your own custom Docker image

### Dockerfile

```Dockerfile
FROM adighe/wait as waitFROM php:7.1.3-fpm
# Install dependencies
RUN apt-get update \  && apt-get install -y \    netcat
RUN curl -sS https://getcomposer.org/installer | php && \
    mv composer.phar /usr/local/bin/composer
COPY --from=wait /app/wait.sh /app/wait.sh
ENTRYPOINT ["docker-php-entrypoint"]CMD ["php-fpm"]
```

### docker-compose.yml

```yaml
version: '3.3'
services:
  db:
    image: mysql:5.7
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: database
  install:
    build:
      context: .
      command: "/bin/bash -c \"/app/wait.sh --wait 'db:3306' --wait 'ls -al /var/www/html/ | grep composer.json' --command 'cd /var/www/html' --command 'ls -al' --command 'composer install' --command 'php /var/www/html/bin/console doctrine:migrations:migrate -n -vvv'\""
```