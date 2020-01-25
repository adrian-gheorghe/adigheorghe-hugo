---
title: Using Jenkins Pipelines in your CI/CD flow
date: 2018-09-13
hero: "/img/nature-forest-trees-fog.jpeg"
excerpt: If you want to use Docker containers in production, chances are you’ll want to store your credentials in a secure way. A way to do that for Docker Swarm is to use Docker secrets.
timeToRead: 8
authors:
  - Adrian Gheorghe
---

Jenkins Pipelines are a very powerful tool that you can use to deploy your web applications. Pipelines allow you to leverage the full power of **Jenkins** but is simple enough to grasp quickly.

All you need is a Jenkinsfile file in your project repository and then add a **Pipeline** or **Multibranch Pipeline** job to your Jenkins server and you’re good to go.

![alt image](/img/1__bYUMVsPZM8QAxVdqoyAgGQ.png "Image")

What a pipeline allows you to do:

- Keep your pipeline/build and deployment process in code
- Parameterize your builds
- Deploy using stages
- Require input or confirmation before proceeding to following stages
- The main syntax is Groovy, but you can include bash and other scripts as well
- After separating your build process in stages you can easily see where bottlenecks appear and optimize that specific area
- Deploy to multiple environments successively and successfully.

A multibranch pipeline is basically a pipeline on steroids. You can filter the vcs branches which you can build from and change the build process accordingly for each branch. For example, for develop branches you could deploy to dev and qa and set only release branches to use the “Production Stage” of the pipeline.

![alt image](/img/1__GtgYp8pQdN91H6UdWcHLfw.png "Image")

Pipeline Example

The following pipeline will build a docker image with the tag value provided as a parameter, login to your docker registry (will need to have credentials stored in Jenkins) and push the image. The temporary image is then removed

```groovy
pipeline {
  agent any
  environment {
    CONTAINER='adighe/test'
  }

parameters {
    string(
        name: 'TAG',
        description: 'Docker Image Tag'
    )
  }
  stages {
    stage('Build') {
      steps {
         withCredentials([
   
     usernamePassword(
         credentialsId: 'docker-registry',
             usernameVariable: 'DOCKER_USER',
             passwordVariable: 'DOCKER_PASSWORD'
         )
     ]) {
     script {
         docker.withRegistry("${DOCKER_REGISTRY}", 'docker-registry') {
               def img = docker.build("${CONTAINER}:${env.TAG}")
               img.push()
               sh "docker rmi ${img.id}"
            }
         }
       } 
         
      }
    }
  }
}
```

Pretty straightforward. Give it a go and see how Jenkins Pipelines can help you out in you CI/CD process