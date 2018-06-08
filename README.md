# Argo installation (locally)

## Prerequisites
[Minikube](https://kubernetes.io/docs/tasks/tools/install-minikube/) with VirtualBox

## Argo
[Getting started](https://github.com/argoproj/argo/blob/master/demo.md)

In brief:

### Download Argo

On Mac: `brew install argoproj/tap/argo`

On Linux: `curl -sSL -o /usr/local/bin/argo https://github.com/argoproj/argo/releases/download/v2.1.1/argo-linux-amd64 && chmod +x /usr/local/bin/argo`

### Install the Controller and UI

`argo install`

### Configure the service account to run workflows

`kubectl create rolebinding default-admin --clusterrole=admin --serviceaccount=default:default`

### Launch UI

`minikube service -n kube-system --url argo-ui`

# Basic worflow definition to build

the worflow definition `submit basic-gradle-build.yaml` gets 2 parameters as input:
* the repo URL (public)
* the refs to build

These parameters must have a default value

By default it checkout the `master` branch of `https://github.com/SonarSource/sonar-dummy-oss.git`

## Build master branch of sonar-dummy-oss
`argo submit basic-gradle-build.yaml`

## Build tag 2.5.0.985 of sonar-dummy-oss
`argo submit basic-gradle-build.yaml -p revision=refs/tags/2.5.0.985`

## Build sonar-go
`argo submit basic-gradle-build.yaml -p repo=https://github.com/SonarSource/sonar-go.git`

## Build sonarqube
`argo submit basic-gradle-build.yaml -p repo=https://github.com/SonarSource/sonarqube.git`

Getting killed!!! :
```
:server:sonar-web:yarn
yarn install v1.5.1
[1/5] Validating package.json...
[2/5] Resolving packages...
[3/5] Fetching packages...
info fsevents@1.1.2: The platform "linux" is incompatible with this module.
info "fsevents@1.1.2" is an optional dependency and failed compatibility check. Excluding it from installation.
[4/5] Linking dependencies...
[5/5] Building fresh packages...
Done in 75.04s.
:server:sonar-web:yarn_run
yarn run v1.5.1
$ node scripts/build.js
Creating optimized production build...
 Killed
 ```

# Basic worflow definition to build and conditionally (using a parameter) run QA

Run the same container definition (gradle) but with different commands `./gradlew build` for build and `./gradlew integration`

Artifacts are shared between containers (TODO).

By default it checkout the `master` branch of `https://github.com/SonarSource/sonar-go.git`

## Build master branch of sonar-go

`argo submit basic-gradle-build-contitional-qa.yaml`

![no QA](https://github.com/drautureau-sonarsource/argo-test/build-no-qa.png)

## Build and qualified master branch of sonar-go

`argo submit basic-gradle-build-contitional-qa.yaml -p qa=true`
