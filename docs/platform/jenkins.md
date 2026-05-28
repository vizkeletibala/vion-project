# Jenkins

Jenkins is the CI/CD orchestrator in this stack.

## Current Service Definition

- image: `registry.vion.test:80/vion/jenkins-docker:lts-jdk17`
- build context: `platform-stack/jenkins`
- route: `jenkins.vion.test`
- direct host port: `8081`
- persistent volume: `jenkins-home`
- runs as `root`
- mounts `/var/run/docker.sock`

The Docker socket mount lets Jenkins build images and run Docker commands on the host daemon.

The image itself is built from `jenkins/jenkins:lts-jdk17` and preinstalls:

- Docker CLI
- Docker Compose v2 plugin

The custom image is built locally by Compose during platform startup, then can be pushed into the local registry through the LAN registry endpoint `registry.vion.test:80` after the registry is running. This avoids a bootstrap dependency on the platform registry before the platform exists.

Because Jenkins talks to the host Docker daemon through `/var/run/docker.sock`, helper containers launched from Jenkins jobs should not assume that container-internal paths can be safely bind-mounted with `-v "$PWD":...`. If a pipeline needs to share the Jenkins workspace with a helper container, a shared-volume pattern such as `--volumes-from "$HOSTNAME"` is more reliable.

## Why It Exists In This Stack

Jenkins is positioned to:

- build application images
- push images to the local registry
- trigger deployment workflows
- host project-specific automation for teams that want a visible CI server

## Bootstrap Notes

The setup wizard is enabled through:

```text
JAVA_OPTS=-Djenkins.install.runSetupWizard=true
```

A new environment should expect first-run manual setup unless that is later automated.

## Typical Workflow

For a game project, Jenkins can:

1. clone the repo
2. build backend or tools images
3. run tests
4. push tagged images to the registry
5. deploy updated services with Docker Compose or another target system

## Example Responsibilities In A Game Repo

- build the game backend API container
- build admin or launcher services
- package asset-processing workers
- publish server tools to the local registry
- run smoke tests against staging services

## Example Docker-Centric Pipeline Shape

```groovy
pipeline {
  agent any

  stages {
    stage('Build') {
      steps {
        sh 'docker build -t registry.vion.test:80/game-api:${BUILD_NUMBER} .'
      }
    }

    stage('Push') {
      steps {
        sh 'docker push registry.vion.test:80/game-api:${BUILD_NUMBER}'
      }
    }
  }
}
```

## Operational Notes

- Jenkins state is stored in the `jenkins-home` volume.
- Because it has Docker socket access, Jenkins is effectively privileged on the host.
- There is no reverse-proxy auth layer configured here; access control must be handled by Jenkins itself or by tightening Traefik exposure.

## If Reused In Another Repo

Keep Jenkins if:

- the team already knows it
- you want UI-based jobs and pipelines
- you need flexible Docker-heavy automation on a local server

Consider replacing it if:

- the new repo already uses GitHub Actions, GitLab CI, or another hosted system
- you want less maintenance
- you want stronger default security boundaries
