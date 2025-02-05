# MJML docker microservice / server

Standalone mjml server, listening on port 80/tcp.

Due to various challenges this image sports the following features:

- Clean and fast shutdowns on docker.
- Simple CORS capabilities.
- Small footprint (at least in a npm way).
- Supports healthchecks.

# Table of contents
  - [Overview](#overview)
  - [Defaults](#defaults)
  - [Development](#development)
  - [Troubleshooting](#troubleshooting)
    - [Kubernetes](#kubernetes)

## Overview

This image spools up a simple mjml server instance, listening to port 80/tcp per default.

Due to GDPR / DSGVO reasons I required the mjml instance to be under my own control, as the processing personal information is processed in mail content generation.

Starting the image is as easy as running a test instance through docker

```sh
docker run -it --rm -p 8080:80 adrianrudnik/mjml-server
```

or `docker-compose` with the following example:

```yml
version: '3.7'

services:
  mjml:
    image: adrianrudnik/mjml-server
    ports:
      - 8080:80
    # environment:
    # to change the port:
    #   - PORT=8080
    # for development, uncomment the following lines:
    #   - CORS=*
    #   - MJML_KEEP_COMMENTS=true
    #   - MJML_VALIDATION_LEVEL=strict
    #   - MJML_MINIFY=false
    #   - MJML_BEAUTIFY=true
```

## Defaults

The production defaults, without any override, default to:

```sh
CORS ""
MJML_KEEP_COMMENTS "false"
MJML_VALIDATION_LEVEL "soft"
MJML_MINIFY "true"
MJML_BEAUTIFY "false"
HEALTHCHECK "true"
CHARSET="utf8"
DEFAULT_RESPONSE_CONTENT_TYPE="text/html; charset=utf-8"
```

## Development

For development environments I would suggest to switch it to

```sh
CORS "*"
MJML_KEEP_COMMENTS "true"
MJML_VALIDATION_LEVEL "strict"
MJML_MINIFY "false"
MJML_BEAUTIFY "true"
HEALTHCHECK "false"
```

This will escalate any issues you have with invalid mjml code to the docker log (`stdout` or `docker-compose logs`).

## Troubleshooting

Make sure you pass along a plain Content-Type header and pass the mjml as raw body.

Catch errors by looking at the HTTP response code.

### Kubernetes

As the default Dockerfile specific `HEALTHCHECK` directive is not supported by kubernetes, you might need to specify your own probes:

```
spec:
  containers:
  - name: ...
    livenessProbe:
      exec:
        command:
        - curl - -X POST - 'http://localhost:80/'
      initialDelaySeconds: 30
      periodSeconds: 30
    readinessProbe:
      exec:
        command:
        - curl - -X POST - 'http://localhost:80/'
      initialDelaySeconds: 25
```

Be aware that this does only check the connectivity and that the port might vary. If you want a functional check as well, you could shift to an approach like the ones used for docker with the result of the [healthcheck.sh](healthcheck.sh). But I'm not a kubernetes user, so feel free to do a pull request if you have a slim approach.

### Docker

If you want to rely on the Makefile or build for multiple architectures, ensure you have the experimental features activated for Docker and you can use [docker buildx](https://docs.docker.com/buildx/working-with-buildx/).

Setup on my Ubuntu 20.04 workstation was as follows, based on the docs mentioned above:

```bash
# Install additional platforms for the default node on the current host linux os
docker run --privileged --rm tonistiigi/binfmt --install all

# create a separate endpoint that uses the default node
docker buildx create --use mjml-server default

# After that a local build should be possible with something like
docker buildx build -f Dockerfile --platform linux/amd64,linux/arm64 -t [registry-and-tag] --push .

# ... or if you want to use it locally with the mjml-server tag
docker buildx build -f Dockerfile --platform linux/amd64,linux/arm64 -t mjml-server --load .
```
