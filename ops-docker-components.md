---
layout: page
exclude_from_nav: true
title: Docker Components
---

This diagram shows the dependencies between the docker services (containers), images, and build processes.

These are defined in the ops-docker [docker-compose.yml](https://github.com/openphacts/ops-docker/blob/master/docker-compose.yml) file.

Each service is either loaded from a docker image file stored at http://data.openphacts.org/, or built according to the 'Dockerfile' in the listed build sub-directory of ops-docker.

![Diagram of docker components.](/images/ops-docker-deps.png)
