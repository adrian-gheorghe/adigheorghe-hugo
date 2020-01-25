---
title: Using Docker Secrets in your Environment Variables
date: 2018-09-13
hero: "/img/nature-forest-trees-fog.jpeg"
excerpt: If you want to use Docker containers in production, chances are you’ll want to store your credentials in a secure way. A way to do that for Docker Swarm is to use Docker secrets.
timeToRead: 3
authors:
  - Adrian Gheorghe
---

If you want to use Docker containers in production, chances are you’ll want to store your credentials in a secure way. A way to do that for Docker Swarm is to use Docker secrets.

A secret can be defined easily enough on your swarm manager using the following:

```bash
echo "mysupersecurepassword" | docker secret create my_password_secret -
```

Now, you will probably want to reference secrets from your environment variables, but that is unfortunately not supported yet. In order to do just that, there is a workaround implemented in the official docker Mysql and WordPress containers.

Secrets are accessible from the containers that have access to them by using the file path /run/secrets/my_password_secret, so what you can do, is add another environment variable to your docker-compose, having a custom name (appending _FILE for example)

```yaml
version: '3.3'
secrets:
  my_password_secret:
    external: true
services:
  db:
    image: mysql:5.7
    environment:
      MYSQL_PASSWORD_FILE: /run/secrets/my_password_secret
```

And in your container entrypoint, call the following function for each environment variable you have set up.
```yaml
file_env() { local var="$1" local fileVar="${var}_FILE" local def="${2:-}" if [ "${!var:-}" ] && [ "${!fileVar:-}" ]; then
      echo >&2 "error: both $var and $fileVar are set (but are exclusive)" exit 1 fi
   local val="$def" if [ "${!var:-}" ]; then
      val="${!var}" elif [ "${!fileVar:-}" ]; then
      val="$(< "${!fileVar}")" fi
   export "$var"="$val" unset "$fileVar" }
```

This will export the value stored in the secret, to the correct environment variable (MYSQL_PASSWORD in this case)