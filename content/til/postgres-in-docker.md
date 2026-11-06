---
title: "Running a PostgreSQL service inside a Docker container"
date: "2026-10-12"
description: "Running a throwaway Postgres in Docker for experiments: pull, run, connect with psql, stop and restart."
tags:
  - postgres
  - docker
---

I'm designing a job queue in Go, backed by Postgres ([pgqueue](https://github.com/frapaparatto/pgqueue)). To make the design decisions, I needed a Postgres service running in the background, to run experiments and understand how Postgres handles some details and their trade-offs. I didn't need a full setup, just a throwaway database to experiment with, without touching the project one. So I decided to run it inside a Docker container. I followed the [official image documentation](https://hub.docker.com/_/postgres).

An image is the template, a container is a running instance of that image. First you pull the image, then you run a container from it.

## Pull the image

```bash
docker pull postgres
```

`docker pull` downloads the `postgres` image from Docker Hub.

## Run the container

```bash
docker run --name postgres-experiment -e POSTGRES_PASSWORD=examplepassword -p 5431:5432 -d postgres
```

- `--name postgres-experiment`: the name of the container, used in the next commands.
- `-e POSTGRES_PASSWORD=examplepassword`: sets the password of the `postgres` superuser. The image requires it. For local experiments, a simple password is fine.
- `-p 5431:5432`: publishes the container port 5432, the default Postgres port, on port 5431 of my machine.
- `-d`: runs the container in the background.
- `postgres`: the image the container is created from.

## Connect with psql

```bash
docker exec -it postgres-experiment psql -U postgres
```

`docker exec` runs a command inside a running container.

- `-i`: keeps the input open.
- `-t`: allocates a terminal. Together with `-i`, it gives you an interactive shell.
- `postgres-experiment`: the container to run the command in.
- `psql`: the Postgres command line client.
- `-U postgres`: connects as the `postgres` user.

## Stop and restart

```bash
docker stop postgres-experiment
docker restart postgres-experiment
```

`docker restart` starts the same container again, with its data: a table I created before stopping it was still there. `docker run` would create a new one.

## Documentation

- [postgres image](https://hub.docker.com/_/postgres)
- [docker run](https://docs.docker.com/reference/cli/docker/container/run/)
- [docker exec](https://docs.docker.com/reference/cli/docker/container/exec/)
- [psql](https://www.postgresql.org/docs/current/app-psql.html)
- [docker restart](https://docs.docker.com/reference/cli/docker/container/restart/)
