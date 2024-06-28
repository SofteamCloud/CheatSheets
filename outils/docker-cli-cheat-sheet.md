# Docker cli
## Inspect 
### Dive
A tool for exploring a docker image, layer contents, and discovering ways to shrink the size of your Docker/OCI image

```
dive imagenameOrId
```
### History 
show image history, list of ancestors
```
docker history image 
```

## Logs 
```
docker container logs --tail 100 containerName
```

## Clean 
### Images clean by name  
```sh
docker rmi $(docker images |grep 'imagename')
```

### Prune system
#### To clean or prune unused (dangling) images
```
docker image prune 
```
#### To remove all images which are not in use containers , add - a
```
docker image prune -a 
```
#### To Purne your entire system
```
docker system prune 
```

### Delete all running and stopped containers
docker container rm -f $(docker ps -aq)


## Manual transfert 
Export image as a tar file 
```sh
docker save repo/image:tag   

```
Load image from tar file 
```
docker load 
```

## Build docker image without docker 
*useful for ci cd* 
of course there is ugly and very often used solution called dind (Docker-in-Docker) but for security reasons you should not do this . 
## with kaniko 
kaniko is a tool to build container images from a Dockerfile, inside a container or a pod 

```yaml
build:
  stage: build
  image:
    name: gcr.io/kaniko-project/executor:debug
    # need to overwrite entrypoint 
    entrypoint: [""]
  script:
    - mkdir -p /kaniko/.docker
    - echo "{\"auths\":{\"https://index.docker.io/v1/\":{\"username\":\"$DOCKERHUB_USERNAME\",\"password\":\"$DOCKERHUB_PASSWORD\"}}}" > /kaniko/.docker/config.json
    - /kaniko/executor --context $CI_PROJECT_DIR --dockerfile $CI_PROJECT_DIR/Dockerfile --destination ${IMAGE_NAME}:${CI_COMMIT_SHA:0:8}
```

## with buildha
use [buildha](https://github.com/containers/buildah) intead 

This image contain buildah for building images and podman which is used for container registry login and running containers. 
Following Dockerfile will give you buildah image to create container 

```sh
docker build -t myimage:latest -<<EOF
FROM fedora
RUN dnf install -y \
  buildah \
  podman \
  make \
  && \
 dnf clean all
EOF
```

still need a mount path mount_path = "/var/lib/containers/" and privileged mode :(   
```yaml 
build:
  stage: build
  image: cyparisot/buildah:v1
  before_script:
    - podman version
    - buildah version
    - podman login --username "${DOCKERHUB_USERNAME}" --password "${DOCKERHUB_PASSWORD}" 
  script:
    - buildah bud -t ${IMAGE_NAME}:${CI_COMMIT_SHA} .
    - buildah push ${IMAGE_NAME}:${CI_COMMIT_SHA} docker://${IMAGE_NAME}:${CI_COMMIT_SHA}
  after_script:
    - podman logout "${REGISTRY_SERVER}"
```
