---
layout: classic-docs
title: "Enabling Docker Layer Caching"
short-title: "Enabling Docker Layer Caching"
description: "How to reuse unchanged cache layers in images you build to reduce overall run time"
categories: [optimization]
order: 70
contentTags: 
  platform:
  - Cloud
  - Server v4.x
  - Server v3.x
  - Server v2.x
---

Docker layer caching (DLC) can reduce Docker image build times on CircleCI. DLC is available on
the [Free and above](https://circleci.com/pricing/) usage plans (credits are charged per run job) and on installations of [CircleCI server](https://circleci.com/enterprise/).

## Overview
{: #overview }

Docker layer caching (DLC) is beneficial if building Docker images is a regular part of your CI/CD process. DLC will save image layers created within your jobs, rather than impact the actual container used to run your job.

DLC caches the individual layers of any Docker images built during your CircleCI jobs, and then reuses unchanged image layers on subsequent CircleCI runs, rather than rebuilding the entire image every time. In short, the less your Dockerfiles change from commit to commit, the faster your image-building steps will run.

Docker layer caching can be used with both the `machine` executor and in the [remote Docker environment](/docs/building-docker-images/) (`setup_remote_docker`).

The underlying implementation of DLC is in the process of being updated. **There is no action required from users.** All further content on this page refers to the implementation of DLC that is in the process of being phased out. Once all jobs have been migrated to the new implementation, the content currently on this page will become outdated and will be replaced with information based on the new architecture.
<br>
<br>
Visit the [Discuss post](https://discuss.circleci.com/t/fyi-small-dlc-update-no-action-required/44614) to learn more details regarding the new architecture and to follow updates regarding the rollout.
{: class="alert alert-info"}

### Limitations
{: #limitations }

Please note that high usage of [parallelism](/docs/configuration-reference/#parallelism) (that is, a parallelism of 30 or above) in your configuration may cause issues with DLC, notably pulling a stale cache or no cache.  For example:

- A single job with 30 parallelism will work if only a single workflow is running, however, having more than one workflow will result in cache misses.
- any job with `parallelism` beyond 30 will experience cache misses regardless of number of workflows running.

If you are experiencing issues with cache-misses or need high-parallelism, consider trying the experimental [docker-registry-image-cache](https://circleci.com/developer/orbs/orb/cci-x/docker-registry-image-cache) orb.  **This limitation does not apply to the new DLC implementation mentioned in the [announcement](#overview) above.**

DLC is only useful when creating your own Docker image with docker build, docker compose, or similar docker commands.It does not decrease the wall clock time that all builds take to spin up the initial environment.

{:.tab.switcher.Cloud}
```yaml
version: 2.1
orbs:
  browser-tools: circleci/browser-tools@1.2.3
jobs:
 build:
    docker:
      - image: cimg/node:17.2-browsers # DLC does nothing here, its caching depends on commonality of the image layers.
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true # DLC will explicitly cache layers here and try to avoid rebuilding.
      - run: docker build .
```

{:.tab.switcher.Server_3}
```yaml
version: 2.1
jobs:
 build:
    docker:
      - image: cimg/node:17.2-browsers # DLC does nothing here, its caching depends on commonality of the image layers.
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true # DLC will explicitly cache layers here and try to avoid rebuilding.
      - run: docker build .
```

{:.tab.switcher.Server_2}
```yaml
# Legacy convenience images (i.e. images in the `circleci/` Docker namespace)
# will be deprecated starting Dec. 31, 2021. Next-gen convenience images with 
# browser testing require the use of the CircleCI browser-tools orb, available 
# with config version 2.1.
version: 2
jobs:
 build:
    docker:
      - image: circleci/node:14.17.3-buster-browsers # DLC does nothing here, its caching depends on commonality of the image layers.
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference
    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true # DLC will explicitly cache layers here and try to avoid rebuilding.
      - run: docker build .
```

DLC has **no** effect on Docker images used as build containers. That is, containers that are used to _run_ your jobs are specified with the `image` key when using the [`docker` executor](/docs/using-docker/) and appear in the **Spin up Environment** step on your jobs pages.
{: class="alert alert-info"}

## How DLC works
{: #how-dlc-works }

DLC caches your Docker image layers by creating an external volume and attaching it to the instances that execute the `machine` and Remote Docker jobs. The volume is attached in a way that makes Docker save the image layers on the attached volume. When the job finishes, the volume is disconnected and re-used in a future job. The layers downloaded in a previous job with DLC will be available in the next job that uses the same DLC volume.

One DLC volume can only be attached to one `machine` or Remote Docker job at a time. If one DLC volume exists but two jobs that request DLC are launched, CircleCI will create a new DLC volume and attach it to the second job. From that point on, the project will have two DLC volumes associated with it. This applies to parallel jobs as well: if two `machine` jobs are run in parallel, they will get different DLC volumes.

Depending on which jobs the volumes are used in, they might end up with different layers saved on them. For instance, the volumes that are used less frequently might have older layers saved on them.

The DLC volumes are deleted after 3 days of not being used in a job.

CircleCI will create a maximum of 30 DLC volumes per project, so a maximum of 30 concurrent `machine` or Remote Docker jobs per project can have access to DLC. This takes into account the parallelism of the jobs, so a maximum of 1 job with 30x parallelism will have access to DLC per project, or 2 jobs with 15x parallelism, and so on.

![Docker Layer Caching](/docs/assets/img/docs/dlc_cloud.png)

### Scope of cache
{: #scope-of-cache }
With DLC enabled, the entirety of `/var/lib/docker` is cached to the remote volume, which also includes any custom networks created in previous jobs.

### Remote Docker environment
{: #remote-docker-environment }

To use DLC in the Remote Docker Environment, add `docker_layer_caching: true` under the `setup_remote_docker` key in your [`.circleci/config.yml`](/docs/configuration-reference/) file:

```yml
- setup_remote_docker:
    docker_layer_caching: true  # default - false
```

Every layer built in a previous job will be accessible in the Remote Docker Environment. However, in some cases your job may run in a clean environment, even if the configuration specifies `docker_layer_caching: true`.

If you run many concurrent jobs for the same project that depend on the same environment, all of them will be provided with a Remote Docker environment. Docker layer caching guarantees that jobs will have exclusive Remote Docker Environments that other jobs cannot access. However, some of the jobs may have cached layers, some may not have cached layers, and not all of the jobs will have identical caches.

DLC was previously enabled via the `reusable: true` key. The `reusable` key has been deprecated in favor of the `docker_layer_caching` key.
<br>
<br>
In addition, the `exclusive: true` option is deprecated and all Remote Docker VMs are now treated as exclusive. This means that when using DLC, jobs are guaranteed to have an exclusive Remote Docker environment that other jobs cannot access.
{: class="alert alert-info"}

### Machine executor
{: #machine-executor }

Docker layer caching can also reduce job runtimes when building Docker images using the [`machine` executor](/docs/configuration-reference/#machine). Use DLC with the `machine` executor by adding `docker_layer_caching: true` below your `machine` key (as seen above in our [example](#configyml)):

```yml
machine:
  image: ubuntu-2004:202104-01  # any available image
  docker_layer_caching: true    # default - false
```

## Examples
{: #examples }

We will use the following example Dockerfile to illustrate how Docker layer caching works. This eDockerfile is adapted from our [Elixir legacy convenience image](https://hub.docker.com/r/circleci/elixir/~/dockerfile):

### Dockerfile
{: #dockerfile }

```dockerfile
FROM elixir:1.11.4

# make Apt non-interactive
RUN echo 'APT::Get::Assume-Yes "true";' > /etc/apt/apt.conf.d/90circleci \
  && echo 'DPkg::Options "--force-confnew";' >> /etc/apt/apt.conf.d/90circleci

ENV DEBIAN_FRONTEND=noninteractive

# Debian Jessie is EOL'd and original repos do not work.
# Switch to the archive mirror until we can get people to
# switch to Stretch.
RUN if grep -q Debian /etc/os-release && grep -q jessie /etc/os-release; then \
	rm /etc/apt/sources.list \
    && echo "deb http://archive.debian.org/debian/ jessie main" >> /etc/apt/sources.list \
    && echo "deb http://security.debian.org/debian-security jessie/updates main" >> /etc/apt/sources.list \
	; fi

# Make sure PATH includes ~/.local/bin
# https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=839155
# This only works for root. The circleci user is done near the end of this Dockerfile
RUN echo 'PATH="$HOME/.local/bin:$PATH"' >> /etc/profile.d/user-local-path.sh

# man directory is missing in some base images
# https://bugs.debian.org/cgi-bin/bugreport.cgi?bug=863199
RUN apt-get update \
  && mkdir -p /usr/share/man/man1 \
  && apt-get install -y \
    git mercurial xvfb apt \
    locales sudo openssh-client ca-certificates tar gzip parallel \
    net-tools netcat unzip zip bzip2 gnupg curl wget make


# Set timezone to UTC by default
RUN ln -sf /usr/share/zoneinfo/Etc/UTC /etc/localtime

# Use unicode
RUN locale-gen C.UTF-8 || true
ENV LANG=C.UTF-8

# install jq
RUN JQ_URL="https://circle-downloads.s3.amazonaws.com/circleci-images/cache/linux-amd64/jq-latest" \
  && curl --silent --show-error --location --fail --retry 3 --output /usr/bin/jq $JQ_URL \
  && chmod +x /usr/bin/jq \
  && jq --version

# Install Docker

#>    # To install, run the following commands as root:
#>    curl -fsSLO https://download.docker.com/linux/static/stable/x86_64/docker-17.05.0-ce.tgz && tar --strip-components=1 -xvzf docker-17.05.0-ce.tgz -C /usr/local/bin
#>
#>    # Then start docker in daemon mode:
#>    /usr/local/bin/dockerd

RUN set -ex \
  && export DOCKER_VERSION=docker-19.03.12.tgz \
  && DOCKER_URL="https://download.docker.com/linux/static/stable/x86_64/${DOCKER_VERSION}" \
  && echo Docker URL: $DOCKER_URL \
  && curl --silent --show-error --location --fail --retry 3 --output /tmp/docker.tgz "${DOCKER_URL}" \
  && ls -lha /tmp/docker.tgz \
  && tar -xz -C /tmp -f /tmp/docker.tgz \
  && mv /tmp/docker/* /usr/bin \
  && rm -rf /tmp/docker /tmp/docker.tgz \
  && which docker \
  && (docker version || true)

# docker compose
RUN COMPOSE_URL="https://circle-downloads.s3.amazonaws.com/circleci-images/cache/linux-amd64/docker-compose-latest" \
  && curl --silent --show-error --location --fail --retry 3 --output /usr/bin/docker-compose $COMPOSE_URL \
  && chmod +x /usr/bin/docker-compose \
  && docker-compose version

# install dockerize
RUN DOCKERIZE_URL="https://circle-downloads.s3.amazonaws.com/circleci-images/cache/linux-amd64/dockerize-latest.tar.gz" \
  && curl --silent --show-error --location --fail --retry 3 --output /tmp/dockerize-linux-amd64.tar.gz $DOCKERIZE_URL \
  && tar -C /usr/local/bin -xzvf /tmp/dockerize-linux-amd64.tar.gz \
  && rm -rf /tmp/dockerize-linux-amd64.tar.gz \
  && dockerize --version

RUN groupadd --gid 3434 circleci \
  && useradd --uid 3434 --gid circleci --shell /bin/bash --create-home circleci \
  && echo 'circleci ALL=NOPASSWD: ALL' >> /etc/sudoers.d/50-circleci \
  && echo 'Defaults    env_keep += "DEBIAN_FRONTEND"' >> /etc/sudoers.d/env_keep

# BEGIN IMAGE CUSTOMIZATIONS

# END IMAGE CUSTOMIZATIONS

USER circleci
ENV PATH /home/circleci/.local/bin:/home/circleci/bin:${PATH}

CMD ["/bin/sh"]
```

### Config.yml
{: #configyml }

In the `.circleci/config.yml` snippet below, let's assume the `build_elixir` job is regularly building an image using the above Dockerfile.

By adding `docker_layer_caching: true` underneath our `machine` executor key, we ensure that CircleCI will save each Docker image layer as this Elixir image is built.

```yaml
version: 2
jobs:
  build_elixir:
    machine:
      image: ubuntu-2004:202104-01
      docker_layer_caching: true
    steps:
      - checkout
      - run:
          name: build Elixir image
          command: docker build -t circleci/elixir:example .
```

On subsequent commits, if our example Dockerfile has not changed, then DLC will pull each Docker image layer from cache during the `build Elixir image` step, and our image will theoretically build almost instantaneously.

Now, let's say we add the following step to our Dockerfile, in between the `# use unicode` and `# install docker` steps:

```dockerfile
# Install jq
RUN JQ_URL="https://circle-downloads.s3.amazonaws.com/circleci-images/cache/linux-amd64/jq-latest" \
  && curl --silent --show-error --location --fail --retry 3 --output /usr/bin/jq $JQ_URL \
  && chmod +x /usr/bin/jq \
  && jq --version
```

On the next commit, DLC will ensure that we still get cached image layers for the first few steps in our Dockerfile—pulling from `elixir:1.11.4` as our base image, the `# make apt non-interactive` step, the step starting with `RUN apt-get update`, the `# set timezone to UTC` step, and the `# use unicode` step.

However, because our `#install jq` step is new, it as well as subsequent steps need to be run from scratch, because the Dockerfile changes will invalidate the rest of the image layer cache. But overall, with DLC enabled our image will still build more quickly, due to the unchanged layers/steps towards the beginning of the Dockerfile.

If we were to change the first step in our example Dockerfile (for example, perhaps we want to pull from a different Elixir base image), then our entire cache for this image would be invalidated even if every other part of our Dockerfile stayed the same.

## Video: overview of Docker layer caching
{: #video-overview-of-docker-layer-caching }

In the video example, the job runs all of the steps in a Dockerfile with the `docker_layer_caching: true` for the `setup_remote_docker` step. On subsequent runs of that job, steps that have not changed in the Dockerfile will be reused.

The first run takes over two minutes to build the Docker image. If nothing changes in the Dockerfile before the second run, those steps happen instantly: in zero seconds.

```yaml
version: 2
jobs:
  build:
    docker:
      - image: cimg/node:14.17.3
        auth:
          username: mydockerhub-user
          password: $DOCKERHUB_PASSWORD  # context / project UI env-var reference

    steps:
      - checkout
      - setup_remote_docker:
          docker_layer_caching: true
      - run: docker build .
```

When none of the layers in the image change between job runs, DLC pulls the layers from cache from the image that was built previously and reuses those instead of rebuilding the entire image.

If part of the Dockerfile changes (which changes part of the image), a subsequent run of the exact same job with the modified Dockerfile may still finish faster than rebuilding the entire image. This is because the cache is used for the first few steps that did not change in the Dockerfile. The steps that follow the change must be rerun because the Dockerfile change invalidates the cache.

This means that if you change something in the Dockerfile, all of those later steps are invalidated and the layers have to be rebuilt. When some of the steps remain the same (the steps before the one you removed), those steps can be reused. So, it is still faster than rebuilding the entire image.

<div class="video-wrapper">
  <iframe width="560" height="315" src="https://www.youtube.com/embed/AL7aBN7Olng" frameborder="0" allow="autoplay; encrypted-media" allowfullscreen></iframe>
</div>

## Learn More
{: #learn-more }
Take the [DLC course](https://academy.circleci.com/docker-layer-caching?access_code=public-2021) with CircleCI Academy to learn more.
