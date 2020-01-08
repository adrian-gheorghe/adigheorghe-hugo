---
title: Dockerizing your personal infrastructure (Docker Swarm, RexRay, Traefik, Let’s Encrypt, DigitalOcean, Jenkins, Ansible)
excerpt: This article is PART 1 of a series on Dockerizing your own personal infrastructure
date: 2018-08-31
hero: "/img/gray-and-black-hive-printed-textile-691710.jpg"
timeToRead: 5
authors:
  - Adrian Gheorghe
---

[Published to Medium](https://medium.com/@adrian.gheorghe.dev/dockerizing-your-own-personal-infrastructure-docker-swarm-rexray-traefik-lets-encrypt-7b3b29b12ad0)

This article is PART 1 of a series on Dockerizing your own personal infrastructure


[PART 2](/posts/docker-swarm-volume-data-persistence-on-digital-ocean-with-rexray) and [PART 3](/posts/docker-swarm-volume-data-persistence-on-digital-ocean-with-rexray)

In January 2018, i published [this post](/posts/dockerizing-your-personal-infrastructure) on how to host your own personal websites or projects by your self, using Docker containers, an Nginx container as a reverse proxy and Let’s Encrypt for SSL certificates. I initially set this up somewhere around October 2017 and it has served me remarkably well even though it was a rudimentary set up.

That being said, there were a couple of downsides to the setup which I wanted to address and/or optimize:

*   The Nginx container I was using worked very well, but certificates were not being renewed when expired
*   Everything was running on a single vps with no way of scaling if needed
*   Deployment was rudimentary via ssh
*   Volumes were mounted directly from the VPS filesystem. If that machine had a hard drive fault, all data would be lost.

Therefore after a lot of research and several failed attempts, this is the setup i have worked out:

*   DigitalOcean hosted infrastructure. 3 VPS: 1 Jenkins server and 2 Docker Swarm nodes
*   Jenkins CI machine to streamline build pipelines as well as maintain history
*   Docker Swarm cluster: 2 VPS — 1 Manager, 1 Worker
*   Traefik container for load balancing and SSL generation
*   RexRay docker storage plugin, integrated with DigitalOcean block storage volumes
*   Ansible playbooks for deployment

### DigitalOcean

I’ll start off by giving a big thumbs up to the guys at Vultr. Their services were really very good and except for 2 outages which they sorted out quickly, everything worked smoothly for the whole period I rented my VPS from them. Also their API is very well documented and easy to use.

I actually did start setting everything up on their infrastructure, but the fact that I was not able to purchase block storage was a show stopper for me. So I decided to move over to Digtal Ocean or AWS (went with DO since the costs were a lot cheaper).

You will need a DO account and of course a credit card 🙂. You can easily create as many Droplets to suit your need. For my setup, I created a droplet for Jenkins, and 2 for Docker Swarm.

### Jenkins

_Jenkins_ is an open source automation server written in Java which allows you to automate your build process. It is very powerful and can be configured to do a lot more. I wrote a short post on Jenkins Pipelines [here](https://medium.com/@adrian.gheorghe.dev/using-jenkins-pipelines-b2cf684d42fb).

I set up a droplet that runs only Jenkins. This can be set up in a matter of minutes by running only a couple of shell commands.

[https://wiki.jenkins.io/display/JENKINS/Installing+Jenkins+on+Ubuntu](https://wiki.jenkins.io/display/JENKINS/Installing+Jenkins+on+Ubuntu)

### Docker Swarm

I created 2 droplets running Ubuntu 16.04 for Docker Swarm. One swarm manager (required) and one swarm worker (or as many as you like)

#### Install Docker on each of them
```bash
sudo apt-get remove docker docker-engine docker.io   
sudo apt-get update   
sudo apt-get install  apt-transport-https  ca-certificates  curl  software-properties-common   
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -  
sudo apt-key fingerprint 0EBFCD88   
sudo add-apt-repository  "deb [arch=amd64] https://download.docker.com/linux/ubuntu  $(lsbrelease -cs)  stable"  
sudo apt-get update  
sudo apt-get install docker-ce
```
#### Set up Swarm Mode

On the manager node run
```bash
docker swarm init
```

Run the output command returned by the swarm initialization on the worker nodes you want to connect to the swarm

```bash
docker swarm join  --token ZZZZZZZ XX.XX.XX.XX:YYYY
```

### Traefik

> A reverse proxy / load balancer that’s easy, dynamic, automatic, fast, full-featured, open source, production proven, provides metrics, and integrates with every major cluster technologies

Traefik works as a reverse proxy / load balancer container for your containerized infrastructure. Mainly, all requests to ports 80 and 443 should be routed to Traefik, which will proxy them to their specific containers. In addition to this, by having access to the docker socket, the Traefik container can detect when a new stack or container is being created.

With Docker Swarm this would work in the following way. Let’s assume you want to host multiple swarm stacks on your swarm. Let’s say their domains are corporateexample.com and blogexample.com.

Docker Traefik Setup repository:

> [https://github.com/adrian-gheorghe/docker-setup](https://github.com/adrian-gheorghe/docker-setup)

The first step would be to create an overlay network Traefix can use accross stacks

```bash
docker network create -d overlay traefik
```

Then you will need to deploy Traefik either in a stack / or a separate container to your Docker Swarm.
```yaml
version: '3'  
  
networks:  
    traefik:  
      external: true  
volumes:  
    data:  
  
services:  
  traefik:  
    image: traefik:1.7  
    command: 
      - "--docker"  
      - "--docker.swarmmode=true"  
      - "--docker.domain=docker.localhost"  
      - "--docker.watch=true"  
      - "--docker.exposedbydefault=true"  
      - "--docker.endpoint=unix:///var/run/docker.sock"  
    ports: 
      - "80:80" # The HTTP port
      - "8080:8080"  # The Web UI (enabled by --api)
      - "443:443"  
    volumes: 
      - /var/run/docker.sock:/var/run/docker.sock  # So that Traefik can listen to the Docker events
      - /opt/traefik/traefik.toml:/traefik.toml  
      - /opt/traefik/acme.json:/etc/traefik/acme/acme.json  
    networks: 
      - traefik  
    labels: 
      - "traefik.enable=false"  
    deploy:  
      placement:  
        constraints: [node.role == manager]  
      restart_policy:  
        condition: on-failure
```
In order to use SSL certificates provided by Let’s Encrypt you will need to add an empty acme.json file and a configuration toml file as volumes mounted to the traefik container.

traefik.

```toml
# defaultEntryPoints must be at the top  
# because it should not be in any table below  
  
defaultEntryPoints = ["http", "https"]  
  
[entryPoints]  
    [entryPoints.http]  
  address = ":80"  
  compress = true  
    [entryPoints.http.redirect]  
    entryPoint = "https"  
  [entryPoints.https]  
  address = ":443"  
    [entryPoints.https.tls]  
  
# Enable ACME (Let's Encrypt): automatic SSL  
[acme]  
email = "example@example.org"  
storage = "/etc/traefik/acme/acme.json"  
entryPoint = "https"  
onHostRule = true  
acmeLogging = true  
[acme.httpChallenge]  
entryPoint = "http"
```

The setup above routes all requests on ports 80 and 443 to the traefik container. Now, for both corpexample.com and blogexample.com we will need to deploy a stack that contains the labels Traefik is looking for.

```yaml
version: '3'  
  
networks:  
    traefik:  
      external: true  
  
services:  
  whoami:  
    image: emilevauge/whoami  
    deploy:  
      labels: 
        - "traefik.enable=true"  
        - "traefik.basic.port=80"  
        - "traefik.basic.protocol=http"  
        - "traefik.backend=e_whoami0"  
        - "traefik.frontend.rule=Host:corpexample.com"  
        - "traefik.docker.network=traefik-net"  
        - "traefik.backend.loadbalancer.swarm=true"  
    networks: 
      - traefik
```
Finally, route your domain corpexample.com to your Swarm Manager IP and voila.

[PART 2 contains a detailed tutorial on using RexRay for data persistence](/posts/docker-swarm-volume-data-persistence-on-digital-ocean-with-rexray).
