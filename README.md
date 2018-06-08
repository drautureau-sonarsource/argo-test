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

# Artifacts repository (Minio)

We need a repository for artifacts in order to donwload the sources and to share them between the different steps of a workflow.

## Prerequisites

[Helm](https://docs.helm.sh/using_helm/)

## install Minio

```
helm init
helm install stable/minio --name argo-artifacts --set service.type=LoadBalancer
```

## Reconfigure the workflow controller to use the Minio artifact repository

Edit the workflow-controller config map to reference the service name (argo-artifacts-minio) and secret (argo-artifacts-minio) created by the helm install:

`kubectl edit configmap workflow-controller-configmap -n kube-system`

```
...
    executorImage: argoproj/argoexec:v2.1.1
    artifactRepository:
      s3:
        bucket: my-bucket
        endpoint: argo-artifacts-minio.default:9000
        insecure: true
        # accessKeySecret and secretKeySecret are secret selectors.
        # It references the k8s secret named 'argo-artifacts-minio'
        # which was created during the minio helm install. The keys,
        # 'accesskey' and 'secretkey', inside that secret are where the
        # actual minio credentials are stored.
        accessKeySecret:
          name: argo-artifacts-minio
          key: accesskey
        secretKeySecret:
          name: argo-artifacts-minio
          key: secretkey
```

## Minio UI

`minikube service --url argo-artifacts-minio`

AccessKey: `AKIAIOSFODNN7EXAMPLE`

SecretKey: `wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY`

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

Tests reports are archived in a Minio bucket.

By default it checkout the `master` branch of `https://github.com/SonarSource/sonar-go.git`

## Limitations: sharing gradle project cache dir and sources between build and QA steps with a volume (Gradle specific)

[K8s created volumes in container with root ownership and stric permissions](https://github.com/kubernetes/kubernetes/issues/2630)

SQ (specificaly ES) cannot run with root user. So we cannot run tests with the default user of openjdk images

Gradle run with user `gradle` and then cannot:
* initiate the project (`<project root dir>/.`). Workaound is to use the CLI option `--project-cache-dir=<other dir>`
* write in the volume

Then we have to send artifacts to Artifactory in the build step and retrieve them in QA step.

As an other consequence of the permission on volume permission (and then artifacts management by Argo) and that we cannot run QA with root user, we have to checkout the sources by our own in QA step.

## Using Secrets to push to Artifactory

Create a new secret named artifactory with url=http://localhost:8081, repo=libs-snapshot-local, user=admin, password=admin APIkey=key:

`kubectl create secret generic artifactory-push --from-literal=url=http://localhost:8081 --from-literal=repo=libs-snapshot-local --from-literal=user=admin --from-literal=password=admin --from-literal=APIkey=key`

## Build master branch of sonar-go

`argo submit basic-gradle-build-contitional-qa.yaml`

![no QA](https://github.com/drautureau-sonarsource/argo-test/build-no-qa.png)

## Build and qualified master branch of sonar-go

`argo submit basic-gradle-build-contitional-qa.yaml -p qa=true`
