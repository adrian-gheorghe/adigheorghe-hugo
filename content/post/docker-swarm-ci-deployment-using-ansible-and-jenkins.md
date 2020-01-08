---
title: Docker Swarm CI deployment using Ansible and Jenkins
date: 2019-03-05
hero: "/images/hero-3.jpg"
excerpt: This article is PART 3 of a series on Dockerizing your own personal infrastructure (Docker Swarm, RexRay, Traefik, Let’s Encrypt…
timeToRead: 8
authors:
  - Adrian Gheorghe

---

# **Docker Swarm CI deployment using Ansible and Jenkins**
[Published on Medium](https://medium.com/@adrian.gheorghe.dev/docker-swarm-ci-deployment-using-ansible-and-jenkins-ddfc99296db8)

This article is PART 3 of a series on **Dockerizing your own personal infrastructure (Docker Swarm, RexRay, Traefik, Let’s Encrypt, DigitalOcean, Jenkins, Ansible)**

[PART 1](https://medium.com/@adrian.gheorghe.dev/dockerizing-your-own-personal-infrastructure-docker-swarm-rexray-traefik-lets-encrypt-7b3b29b12ad0) and [PART 2](https://medium.com/@adrian.gheorghe.dev/docker-swarm-volume-data-persistence-on-digital-ocean-with-rexray-cd418f718131)

In order to set up a continuous integration workflow for Docker Swarm mode you can use an array of tools. For the purpose of this article and for my personal stack i have used Jenkins and Ansible. They are both open source tools with very large communities.

Let’s assume you want to deploy a single page application developed using React to your Docker Swarm using Jenkins and Ansible. You will need to set up a git repository that contains your application code. Some additional files will need to be added to the repo for this set up, but more on each below.

Jenkinsfile / Dockerfile / docker-compose.yml / devops directory

### Jenkins

> _Jenkins_ is an open source automation server written in Java which allows you to automate your build process.

I have written about Jenkins before and how you can implement [Jenkins pipelines to set up your continuous integration workflow](https://medium.com/@adrian.gheorghe.dev/using-jenkins-pipelines-b2cf684d42fb). Assuming you already have a Jenkins server set up, what you need to do first of all is add a Jenkinsfile to your application repository. Afterwards, you will need to set up a new Jenkins pipeline / multibranch pipeline project on your Jenkins install having the application repository as a scm source.

This will allow Jenkins to do multiple things. First off it will use your repository code as a build starting point. Checking out on the latest commit of your specified branch. Secondly, it will parse your **Jenkinsfile** and will execute all the steps provided by it. What the **Jenkinsfile** contains is up to you. Using Groovy syntax you can add stages and steps to your deployment process.

```groovy
pipeline {  
  agent any  
  
  
  environment {  
    PROJECT_NAME='myproject'  
    DOMAIN=’mydomain.com'  
    STACK=’mystack'  
    DOCKER_REGISTRY=’https://my.docker.registry'  
    CONTAINER=’vendor/app'  
    VERSION="1.${BUILD_NUMBER}"  
  }  
  stages {  
    stage(’Tag Git Commit’) {  
      steps {  
        sshagent ([’jenkins’\]) {  
            script {  
                sh '’'  
                    git tag -a "v${VERSION}" -m "Jenkins"  
                    git push origin "v${VERSION}" -vvv  
                '’'  
            }  
        }  
      }  
    }  
    stage(’Build Image’) {  
      steps {  
        script {  
            docker.withRegistry("${DOCKER_REGISTRY}", 'docker-registry-credentials’) {  
                def img = docker.build("${CONTAINER}:${VERSION}")  
                img.push()  
                sh "docker rmi ${img.id}"  
            }  
        }  
      }  
    }  
    stage(’Deploy Stack’) {  
      steps {  
          withCredentials([  
              usernamePassword(  
                  credentialsId: 'docker-registry-credentials’,  
                  usernameVariable: 'DOCKER_USER’,  
                  passwordVariable: 'DOCKER_PASSWORD'  
              )  
          \])  
          {  
            script {  
                echo "Deploying Container Stack to Docker Cluster"  
                sh "ansible-playbook -i devops/inventories/manager1/hosts devops/manager1.yml --extra-vars=\\"{’WORKSPACE’: '${env.WORKSPACE}’, 'DOMAIN’: '${env.DOMAIN}’, 'PROJECT’: '${env.PROJECT}’, 'STACK’: '${env.STACK}’, 'VERSION’: '${env.VERSION}’, 'DOCKER_REGISTRY’: '${env.DOCKER_REGISTRY}’, 'DOCKER_USER’: '${env.DOCKER_USER}’, 'DOCKER_PASSWORD’: '${env.DOCKER_PASSWORD}’}\\" -vvv"  
            }  
          }  
      }  
    }  
  }  
}
```

Reading through the **Jenkinsfile** pipeline, we see that the first stage adds a git tag with the Build Number to the repository. This step is not necessary, but i find it better to have a build reference stored as a tag in the git repo as well. This makes for an easier understanding of each release.

The following stage will build and push the Docker image for our react app.

> Store your docker registry credentials in Jenkins. They will need to be referenced when pushing / pulling your container images.

Our repository should contain the **Dockerfile** for the application Docker image we will be building.

```dockerfile
# base image  
FROM node as builder  
  
# set working directory  
RUN mkdir /usr/src/app  
WORKDIR /usr/src/app  
  
# install and cache app dependencies  
COPY package.json /usr/src/app/package.json  
RUN npm install --silent  
RUN npm install react-scripts@1.1.1 -g --silent  
  
COPY . /usr/src/app  
  
RUN npm run build  
  
## production environment

FROM nginx  
COPY --from=builder /**usr**/**src**/**app**/**build **/**usr**/**share**/**nginx**/**html  
EXPOSE 80  
CMD ["nginx", "-g", "daemon off;"\]
```

This **Dockerfile** uses a **Node** base image in order to be able to run **npm run build** and then switches to an **Nginx** image having the application code directly embedded in the image. This is of course an example, this should contain whatever suits your needs for your container.

The next step in the Jenkinsfile builds the new version of the image and pushes it to whatever docker registry you are using, passing in the registry credentials stored in Jenkins. Afterwards the local image is removed to save storage space.

Alongside your Dockerfile, you will need to set up the container stack using a **docker-compose.yml** file. In our case the stack will only contain one service. The nginx web server we have just build.

```yaml
version: '3.3'  
  
networks:  
  traefik-net:  
    external: true  
services:  
  web:  
    image: my.docker.registry/vendor/app:IMAGEVERSION  
    deploy:  
      placement:  
        constraints: 
          - node.role == worker  
      restart_policy:  
        condition: on-failure  
      labels: 
        - "traefik.enable=true"  
        - "traefik.basic.port=80"  
        - "traefik.basic.protocol=http"  
        - "traefik.backend=mydomain"  
        - "traefik.frontend.rule=Host:mydomain.com"  
        - "traefik.docker.network=traefik-net"  
        - "traefik.backend.loadbalancer.swarm=true"  
        - "traefik.frontend.headers.SSLRedirect=true"  
        - "traefik.frontend.headers.SSLHost=mydomain.com"  
        - "traefik.frontend.headers.SSLForceHost=true"  
    networks: 
- traefik-net
```
The service is set up to only reside on a worker node and not on a manager, will restart on failure and has the labels required by Traefik to correctly redirect requests to mydomain.com to your container on the correct port. You will of course need to point mydomain.com to your Docker Swarm manager.

Moving on in the Jenkinsfile to the next stage of the pipeline, where the stack is actually deployed to your docker swarm manager. We then run the ansible playbook using the parameters we need as extra arguments.

The Ansible playbook resides in the devops directory, but it can be placed anywhere you require.

```yaml
---  
- hosts: manager1  
  become: true  
  any_errors_fatal: true  
  
  tasks: 
    - name: Create project directory  
      file: path=/opt/{{ PROJECT }}/{{ DOMAIN }} state=directory  
      tags: docker-swarm-deploy  
  
    - name: Copy Docker Stack File  
      copy:  
        src: ../docker-compose.yml  
        dest: /opt/{{ PROJECT }}/{{ DOMAIN }}/docker-compose.yml  
        owner: "root"  
        group: "root"  
        mode: u=rw,g=r,o=r  
      tags: docker-swarm-deploy  
  
    - name: Replace Placeholders Version  
      shell: >  
        sed -i -e 's/IMAGEVERSION/{{ VERSION }}/g' docker-swarm.yml  
      args:  
        chdir: "/opt/{{ PROJECT }}/{{ DOMAIN }}/"  
      become: true  
      become_user: root  
      tags: docker-swarm-deploy  
  
    - name: Login to registry  
      shell: >  
        docker login -u {{ DOCKER_USER }} -p {{ DOCKER_PASSWORD }} {{ DOCKER_REGISTRY }}  
      become: true  
      become_user: root  
      tags: docker-swarm-deploy  
  
    - name: Deploy Docker Stack  
      shell: >  
        docker stack deploy -c /opt/{{ PROJECT }}/{{ DOMAIN }}/docker-compose.yml {{ STACK }} --with-registry-auth  
      become: true  
      become_user: root  
      tags: docker-swarm-deploy
```

Alongside the playbook we have a hosts file as well. The hosts file contains the docker swarm manager host or ip that ansible should connect to.

```yaml
[manager1] 
11.11.11.11  
  
[manager1:vars] 
ansible_ssh_user=root  
ansible_ssh_private_key_file=/jenkins/path/to/private/key
```

The playbook contains tasks that Ansible needs to run on the Docker manager in order to deploy your stack to the swarm. That means copying the docker-compose file to a specific location, replacing the image tag placeholder with the new version so we get new containers with the new code.

The next task is making sure we are logged into the docker registry we are using to store the image so we don’t get an unauthorized error.

Finally the last task deploys the stack in the docker-compose.yml to your docker swarm manager. The manager handles the rest and creates new services for **mydomain.com.** Traefik listens in on the services created and will now redirect all requests to mydomain.com to your stack, also generating a Let’s Encrypt certificate in the process.

So after adding all this to your application repository, and the task in Jenkins, you will be able to run a build that deploys your containerized application to your Docker Swarm.