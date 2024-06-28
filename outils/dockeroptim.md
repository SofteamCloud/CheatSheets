# Docker advices
## Optimizing your application containerization
for optimizing your application containerization best practices is to follow Twelve-Factor App methodology

### Build the smallest image possible
You can use several strategies to achieve that: start with a minimal base image, leverage common layers between images and make use of Docker’s multi-stage build feature.

For example this is the size og python base images
python:3  is 933MB
python:3.7-slim  is 179MB
python:3.7-alpine is 95.8MB 

This decreases download times, cold start times, and disk usage.
and unnecessary binary

### Optimize for the Docker build cache
Take this Dockerfile as example:
```DockerFile
  FROM python:3
COPY my_code/ /src
RUN pip install my_requirements
```
You should swap the last two lines:
```DockerFile
FROM python:3
RUN pip install my_requirements
COPY my_code/ /src
```
In the new version, the result of the pip command will be cached and will not be rerun each time the source code changes.

### Remove unnecessary tools
Reducing the attack surface of your host system is always a good idea, and it’s much easier to do with containers than with traditional systems. Remove everything that the application doesn’t need from your container. Or better yet, include just your application minimum configuration to run 

Example : 
```DockerFile
RUN pip install --prefix=/install -Ur requirements.txt
# Install packages needed to run your application (not build deps):
#   mime-support -- for mime types when serving static files
#   postgresql-client -- for running database commands
# We need to recreate the /usr/share/man/man{1..8} directories first because
# they were clobbered by a parent image.
RUN set -ex \
    && RUN_DEPS=" \
    libpcre3 \
    mime-support \
    postgresql-client \
    " \
    && seq 1 8 | xargs -I{} mkdir -p /usr/share/man/man{} \
    && apt-get update && apt-get install -y --no-install-recommends $RUN_DEPS \
    && rm -rf /var/lib/apt/lists/*
RUN pip install --prefix=/install -Ur requirements.txt
```
## use multistage build
multi-stage build process is to copy only binary of you application. 
cf images  
this way you can get the miminum image size. 

```Dockerfile
FROM python:3.7-alpine AS builder

ENV PYTHONUNBUFFERED 1
WORKDIR /app

# Install System dependencies (add yours )
RUN apk update && \
    apk add --no-cache jpeg-dev zlib-dev \
        python-dev build-base

# Install Python dependencies
RUN pip install --prefix=/install -Ur requirements.txt

# Start a new image.
FROM python:3.7-alpine

ENV PYTHONUNBUFFERED 1
WORKDIR /app

# Copy the dependencies
COPY --from=builder /install /install

# Copy local application into image.
COPY ./app.py /app/
```

### Do Not Run Dockerized Applications as Root
Most containerized processes are application services and therefore don't require root access. While Docker requires root to run, containers themselves do not. Well written, secure and reusable Docker images should not expect to be run as root and should provide a predictable and easy method to limit access
```Dockerfile
# Create a group and user to run our app
ARG APP_USER=appuser
RUN groupadd -r ${APP_USER} && useradd --no-log-init -r -g ${APP_USER} ${APP_USER}

# # Change to a non-root user
USER ${APP_USER}:${APP_USER}
```

### Externalise your variable 
Config varies substantially across deploys, code does not.
These configurations are then injected into the container either in the form of environment variables or in the form of configuration files directly mounted in the containers.

```Dockerfile

ENV PYTHONUNBUFFERED 1 \
    APP_ENV='demo'
    DATABASE_URL=''\
    (every Config needed)

....
## Call collectstatic (customize the following line with the minimal environment variables needed for manage.py to run):
RUN DATABASE_URL='' python manage.py collectstatic --noinput

```

## Security tools  
### OWASP ZAP on docker   
https://www.grottedubarbu.fr/owasp-zap-docker/