![todo](images/todo.png)
* Basic and rapid introduction
- Coding the CI / CD infrastructure
* Coding the build - part 1 (Jenkins)
* Coding the build - part 2 (build in containers)
* Coding the build - part 3 (docker in docker)

!SUB
## Docker
Docker containers wrap up a piece of software in a complete filesystem that contains everything it needs to run: code, runtime, system tools, system libraries - anything you can install on a server. This guarantees that it will always run the same, regardless of the environment it is running in.
![docker](images/docker-logo.png)


!SUB
## Docker architecture
![architecture](images/architecture.jpg)


!SUB
## Docker Compose

Compose is a tool for defining and running complex applications with Docker. With Compose, you define a multi-container application in a single file, then spin your application up in a single command which does everything that needs to be done to get it running.

![docker-compose](images/docker-compose.png)


!SUB
### Docker compose by example


```
slides:
  image: nginx

  environment:
    VIRTUAL_HOST: 'nextbuild.docker'
    VIRTUAL_PORT: 80
  volumes:
    - .:/usr/share/nginx/html
  ports:
    - "8889:80"
```

!SLIDE
## Build environment

![build-env-jenkins](images/build-env-jenkins.png)

!SUB
## Build environment

![build-env-jenkins-containers](images/build-env-jenkins-containers.png)


!SUB
## Code the infrastructure

`docker-compose.yml`
<pre class="fragment"><code>
gitlab:
  image: 'gitlab/gitlab-ce:latest'
  hostname: 'gitlab.docker'
  ports:
    - '81:8080'
  volumes:
    - '/opt/gitlab/config:/etc/gitlab'
    - '/opt/gitlab/logs:/var/log/gitlab'
    - '/opt/gitlab/data:/var/opt/gitlab'
</code></pre>

<pre class="fragment"><code>
jenkins:
  image: 'jenkins:1.642.3'
  links:
    - gitlab:gitlab.docker
  ports:
    - '82:8080'
  volumes:
    - '/opt/jenkins/jenkins_home:/var/jenkins_home'
</code></pre>

!SUB
## Code the infrastructure
OKAY now we have GIT and Jenkins running. But we still need to install some magick: the jenkins plugins.

![jenkins-oops](images/jenkins-oops.png)


!SUB
## Code the infrastructure
- Create a jenkins docker image on top of the jenkins image.
- Jenkins base image provides a mechanism to specify the plugins.

![jenkins-build](images/jenkins-build.png)


!SUB
## Code the infrastructure

<div class="fragment">
<code>Dockerfile</code>
<pre><code>
MAINTAINER Niek Palm <dev.npalm@gmail.com>
FROM jenkins:1.642.3

COPY plugins.txt /usr/share/jenkins/plugins.txt
RUN /usr/local/bin/plugins.sh /usr/share/jenkins/plugins.txt
</code></pre></div>

<div class="fragment">
<code>Plugins.txt snippet</code>
<pre><code>
workflow-support:1.15
workflow-multibranch:1.15
workflow-step-api:1.15
git:2.4.4
workflow-basic-steps:1.15
workflow-api:1.15
workflow-scm-step:1.15
pipeline-rest-api:1.0
pipeline-stage-view:1.0
</code></pre></div>

!SUB
## Code the infrastructure
Update `docker-compose.yml`

```
gitlab:
  image: 'gitlab/gitlab-ce:latest'

  ...

jenkins:
  build: 'jenkins'
  links:
    - gitlab:gitlab.docker
  ports:
    - '82:8080'
  volumes:
    - '/opt/jenkins/jenkins_home:/var/jenkins_home'
```


!SLIDE
## Code the build
![jenkins](images/jenkins.png)


!SUB
## Code the build - Sample app
![service-overview](images/service.png)

- The sample application is a Java based Spring Boot service witch uses a mongo db (data store).

- The build is scripted in gradle. We can build by running:
 ```
 gradlew <tasks>
 ```
- The application is distributed as a docker container.


!SUB
## Code the build - Jenkins Pipeline
- Jenkins pipeline plugin provides a way to code the build via a script file.
- Jenkins pipeline will be part of the new coming Jenkins 2.0
- Steps
  - Create a `Jenkinsfile` to code the build.
  - Create a pipeline job which uses SCM (GIT) to get the `Jenkinsfile`.





!SUB
## Code the build - Jenkins

`Jenkinsfile`
```
node {
  git url: 'http://gitlab.docker/nextbuild/service.git'

  stage 'Clean'
  sh './gradlew clean'

  stage 'Build'
  sh './gradlew assemble'

  stage 'Test'
  sh './gradlew check'

}
```

!SUB
## Jenkins build
![jenkins-overview](images/jenkins-overview.png)

!SUB
## Artifactory and Sonar


!SUB
## Managing artifacts - Artifactory
- Next we add Artifactory OSS as repository to maintain artifacts.
  - Add Artifactory OSS CI ecosystem and update docker-compose.
  - Gradle build is already aware of the repository.



!SUB
### Artifactory - Update Compose
```
artifactory:
  image: 'jfrog-docker-reg2.bintray.io/jfrog/artifactory-oss'
  ports:
    - '87:8081'
  volumes:
    - '/opt/artifactory/data:/var/opt/jfrog/artifactory/data'
    - '/opt/artifactory/logs:/var/opt/jfrog/artifactory/logs'
    - '/opt/artifactory/backup:/var/opt/jfrog/artifactory/backup'

```

!SUB
## Measuring Code Quality - Sonar
- Next we add SonarQube to measure the code qaulity.
  - Add SonarQube to our CI ecosystem and update docker-compose.
  - Add Sonar to the build and update Jenkinsfile.

!SUB
### Sonar - Update Compose
```

sonardb:
  image: postgres:9
  environment:
    POSTGRES_USER: 'sonar'
    POSTGRES_PASSWORD: 'secret'

sonar:
  image: sonarqube:5.4
  environment:
    SONARQUBE_JDBC_USERNAME: 'sonar'
    SONARQUBE_JDBC_PASSWORD: 'secret'
    SONARQUBE_JDBC_URL: 'jdbc:postgresql://sonardb.docker/sonar'
  links:
    - sonardb:sonardb.docker
    - gitlab:gitlab.docker
  ports:
    - "9000:9000"

jenkins:
  ...
  links:
    ...
    - sonardb:sonardb.docker
    ...

```

!SUB
### Sonar - Update Jenkinsfile

```
node {
  git url: 'http://gitlab.docker/nextbuild/service.git'

  ...

  stage 'QA'
  sh './gradlew sonarRunner'

}
```


!SLIDE
COOL now we can code the build as part of our sources. But what about the infrastructure we are using for the build? Can we create a stable and reproducable infrastructure for the build as well?

!SLIDE
## Code the infrastructure part 2
- Running a build in a container provides a reproducable and consistent environment.
- GitLab CI fits better because of the model of runners that supports docker.
![docker-gitlab](images/docker-gitlab.png)

!SUB
## GitLab CI - Runners
![gitlab-ci-overview](images/gitlab-ci-overview-small.png)


!SUB
## Code the build - GitLab CI
- GitLab CI is fully integrated in GitLab.
- GitLab CI builds automatically once a `.gitlab-ci.yml` file is in the root of the repo.

`.gitlab-ci.yml` snippet
```
stages:
  - build

assemble:
  stage: build
  script:
    - ./gradlew clean assemble
```

!SUB
## Code the build in GitLab CI
We have GitLab CI already running since we use GitLab as our GIT server. So, we only need to create a runner.

<div class="fragment">
Create the runner `docker.compose.yml`
<pre><code>
gitlabrunner:
  image: 'gitlab/gitlab-runner:latest'
  environment:
    REGISTRATION_TOKEN: 'JCqWxz4LcNmth4F4PdTo'
    CI_SERVER_URL: 'http://gitlab.docker/ci'
  ...
</code></pre></div>
<div class="fragment">
Register the runner
<pre><code>
\> docker exec -i -t gitlab-runner-dind1 gitlab-runner register -n \
   --docker-links 'gitlab_gitlab_1:gitlab.docker'
</code></pre></div>

!SUB
## Code the build in GitLab CI
`.gitlab-ci.yml` for our service

```
stages:
  - build
  - test
  - qa

assemble:
  stage: build

  image: npalm/java:oracle-java8

  script:
    - ./gradlew clean distTar

test:
  stage: test

  image: npalm/java:oracle-java8

  script:
    - ./gradlew check

```

!SUB
## Code the build in GitLab CI
`.gitlab-ci.yml` for our service
```
stages:
  - build
  - test
  - qa

assemble:
  ...

test:
  ...

sonar:
  stage: qa

  image: npalm/java:oracle-java8

  script:
    - ./gradlew sonarRunner

```

!SLIDE
## Code the build part 3
- How can we run end-to-end test for our service?
![service-overview](images/service.png)


!SUB
## Test our app using docker
- Define a docker-compose file to start our service and dependencies.
  - The service has a `Dockerfile` which defines the service container.
  - For mongo we use the official docker image.

<div class="fragment">
`docker-compose.yml`
<pre><code>
mongo:
  image: mongo:3.2
service:
  build: ./build/docker
  ports:
    - "8888:8080"
  links:
    - mongo:mongodb
</pre></code></div>


!SUB
## Docker in Docker
![dind-build](images/dind-build.png)


!SUB
## Code the build part 3
`.gitlab-ci.yml`
```
stages:
  - build
  - test
  - qa

...

performance_test:
  stage: test

  image: npalm/dind-java:latest

  script:
    - ./gradlew assemble copyDockerfiles
    - docker-compose build
    - docker-compose up -d
    - ./wait.sh service_service_1 service 8888
    - ./gradlew jmeterRun

...

```

!SLIDE
# Thanks

All sources are available on GitHub
- Slides<br>
  - `open http://codethebuild.github.io/slides/`
  - `git clone https://github.com/codethebuild/slides.git`
- CI / CD enviroment
  - `git clone https://github.com/codethebuild/cicd.git`
- Sample sources Java Spring Boot service
  - `git clone https://github.com/codethebuild/service.git`
