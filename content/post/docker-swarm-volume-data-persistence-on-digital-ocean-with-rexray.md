---
title: Docker Swarm volume data persistence on Digital Ocean with RexRay
date: 2019-03-09
hero: /img/gray-rocks-in-front-of-mountain-939719.jpg
excerpt: This article is PART 2 of a series on Dockerizing your own personal infrastructure (Docker Swarm, RexRay, Traefik, Let’s Encrypt…
timeToRead: 5
authors:
  - Adrian Gheorghe

---

[Published to Medium](https://medium.com/@adrian.gheorghe.dev/docker-swarm-volume-data-persistence-on-digital-ocean-with-rexray-cd418f718131)

This article is PART 2 of a series on **Dockerizing your own personal infrastructure (Docker Swarm, RexRay, Traefik, Let’s Encrypt, DigitalOcean, Jenkins, Ansible)**

[Part 1](/blog/devops/dockerizing-your-own-personal-infrastructure-docker-swarm-rexray-traefik-lets-encrypt/) - [Part 3](/blog/devops/docker-swarm-ci-deployment-using-ansible-and-jenkins/)

### RexRay

> _REX-Ray enables stateful applications, such as databases, to persist and maintain its data after the life cycle of the container has ended. Built-in high availability enables orchestrators such as Docker Swarm, Kubernetes, and Mesos Frameworks like Marathon to automatically orchestrate storage tasks between hosts in a cluster._

In order to set up volume persistence for your data, you will need to install a docker storage plugin. After some failed attempts I have settled on rex-ray as it integrates with all the big cloud providers. [Rexray](https://github.com/rexray/rexray) can run both as a standalone service and as a docker plugin, but it was easier for me to configure it as a docker plugin.

In order to install the docker plugin and configure it to use Digital Ocean block storage you will need a Digital Ocean access token.

The following command should be run on all nodes in the swarm ( both managers and workers )

```bash
docker plugin install rexray/dobs DOBS_TOKEN=aaabbbcccdddeeefff DOBS_REGION=ams3 LINUX_VOLUME_FILEMODE=0775
```

This command installs RexRay as a docker plugin with the Digital Ocean token provided, allowing it to connect to the Digital Ocean api and manage your block storage, and with the droplet region you are using. I am not sure why this is required, but i assume the Digital Ocean api requires the droplet region for some of the methods used.

The filemode parameter is useful depending on the permissions your data required.

The parameter LIBSTORAGE_VOLUME_OPERATIONS_MOUNT_ROOT_PATH can be used in order to mount the data inside a specified subdirectory of the volume.

Now we can create the volume on the manager. The size option tells rexray to create a digital ocean block storage of 1GB and attach it to your manager. Your swarm workers will automatically have access to it as well

```bash
docker volume create --name=mynewvolume --opt=size=1 --driver=rexray/dobs
```

Now in your docker-compose.yml file for your stack, you will need to add the external volume and reference it

```yaml
version: '3.3'

volumes:  
  mynewvolume:  
    external: true  
services:  
  db:  
    image: mysql:5.7  
    volumes:  
      - mynewvolume:/var/lib/mysql
```
One limitation i have found is that you can attach a maximum of 7 block storage volumes to a droplet.

For more info, here’s a link to the official documentation for RexRay [RexRay Docs](https://rexray.readthedocs.io/en/latest/) — [RexRay Digital Ocean Provider Docs](https://rexray.readthedocs.io/en/latest/user-guide/storage-providers/digitalocean/).

The following article in the series contains information on how to deploy container stacks to a docker swarm using Ansible and Jenkins