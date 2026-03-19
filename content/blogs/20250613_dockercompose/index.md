---
title: "Taskfile magic with docker[-]compose"
date: 2025-06-13
description: "Taskfile magic with docker compose"
image: "/blogs/20250613_dockercompose/images/taskfile_magic_docker.png"
tags:
  - taskfile
  - docker-compose
  - docker
---
I have been using [Taskfile](https://taskfile.dev/) for a while now, and I really like it. It is a great alternative to Makefile, and it has some nice features that make it easier to use.

In this post I wanted to share some "magic" to be able to select the right docker compose application to use. Either the older `docker-compose` or the newer `docker compose` command, using the compose plugin.

<!--more-->

I have a `Taskfile.yml` that looks like this that allows me to update running dockers, by finding them using `docker compose ls` and with those entries follow the path and pull the images and restart the containers.

I want to be able to use the same tasks, even though I have to work with a system that still uses the older `docker-compose` command.

## How to deal with this?

In the `var` section of the `Taskfile.yml`, I define a variable `DOCKERCOMPOSE` that will be set to either `docker compose` or `docker-compose` depending on the availability of the command.

The command is: `command -v docker-compose &> /dev/null && echo "docker-compose" || echo "docker compose"`

The section would therefore result in:

```yaml
vars:
  DOCKERCOMPOSE:
    sh: command -v docker-compose &> /dev/null && echo "docker-compose" || echo "docker compose"
```

## Taskfile usage

This will allow you to use the `DOCKERCOMPOSE` variable in your tasks, like so:

```yaml
tasks:
  up:
    dir: "}"
    preconditions:
      - test -f docker-compose.yml
    cmds:
      - "} } up -d --remove-orphans"
```

I have the same kind of logic around `}`. If this environment variable is set, it will use `sudo` to run the command, otherwise it will just run the command without `sudo`. This therefore depends on the system I used it on.

Essentially, this allows me to use the same `Taskfile.yml` on different systems, without having to change the commands or the tasks.

## More tasks around docker compose

Below some more tasks that I use in my `Taskfile.yml`:

```yaml
version: "3"

# env SUDO=sudo may be set and will be used if so

vars:
  DOCKERCOMPOSE: "docker compose"

tasks:
  default:
    cmd: task -l --sort none
    silent: true

  ### Per project tasks
  up:
    desc: "Start the docker-compose environment"
    dir: "}"
    preconditions:
      - test -f docker-compose.yml
    cmds:
      - "} } up -d --remove-orphans"

  down:
    desc: "Stop the docker-compose environment"
    dir: "}"
    preconditions:
      - test -f }/docker-compose.yml
    cmds:
      - "} } down"

  ps:
    desc: "List the containers in the docker-compose environment"
    dir: "}"
    preconditions:
      - test -f }/docker-compose.yml
    cmds:
      - "} } ps"

  update:
    desc: "Update the docker-compose environment"
    dir: "}"
    preconditions:
      - test -f }/docker-compose.yml
    cmds:
      - "} } pull"

  restart:
    desc: "Restart the docker-compose environment"
    dir: "}"
    preconditions:
      - test -f }/docker-compose.yml
    cmds:
      - "} } restart"

  ### Global tasks
  update-all:
    desc: "Update all docker-compose environments"
    aliases:
      - update_all
    preconditions:
      - "[ `uname -n` = nas ]"
    cmd: |
      this_dir=$(pwd)
      echo "Get running } envs"
      running=$(} } ls | grep running | awk '')
      for x in $running; do
        cd $x
        pwd
        task update
        task up
        cd $
        echo ""
      done

  restart-all:
    desc: "Restart all docker-compose environments"
    aliases:
      - restart_all
    preconditions:
      - "[ `uname -n` = nas ]"
    cmd: |
      this_dir=$(pwd)
      echo "Get running } envs"
      running=$(} } ls | grep running | awk '')
      for x in $running; do
        cd $x
        pwd
        } } restart
        cd $
        echo ""
      done

  prune:images:
    desc: "Prune unused docker images"
    interactive: true
    cmds:
      - "} docker image prune"

```
