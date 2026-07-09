---
title: "Taskfile inheritance: includes, excludes, and overriding tasks"
date: 2026-07-09
description: "How to share a common Taskfile across projects, selectively exclude tasks, and override them with project-specific behaviour."
image: "/blogs/20260709_taskfile-include-exclude/images/taskfile_inherit.png"
tags:
  - taskfile
  - docker-compose
  - docker
---
In a [previous post](/blogs/20250613_dockercompose/) I showed some Taskfile magic for working with docker compose. That post covered the `DOCKERCOMPOSE` variable trick and a set of reusable tasks for managing containers. This post picks up where that left off: what do you do when you have multiple projects that all need those same tasks, but one of them needs to behave slightly differently?

## The shared root Taskfile

My `dockers` repository has a root `Taskfile.yml` that defines common docker compose tasks. Every sub-project (monitoring, paperless, pihole, …) can reuse them. The relevant part looks like this:

```yaml
vars:
  DOCKERCOMPOSE: "docker compose"
  DOCKER_COMPOSE_FILE: "compose.yml"

tasks:
  up:
    desc: "Start the docker-compose environment"
    dir: "{{.USER_WORKING_DIR}}"
    preconditions:
      - test -f {{.DOCKER_COMPOSE_FILE}}
    cmds:
      - "{{.SUDO}} {{.DOCKERCOMPOSE}} up -d --remove-orphans"

  down:
    desc: "Stop the docker-compose environment"
    # ...

  logs:
  ps:
  update:
  restart:
  # ... and more
```

Having this in one place means every sub-project gets `up`, `down`, `logs`, `ps`, `update`, and `restart` for free — without copying anything.

## Including it in a sub-project

Taskfile has an `includes` directive that lets you pull another Taskfile into yours. Combined with `flatten: true`, all tasks from the included file land directly in the current namespace — no prefix needed.

```yaml
includes:
  root:
    taskfile: ../Taskfile.yml
    flatten: true
```

With this, running `task up` inside `monitoring/` works exactly as if the root tasks were defined locally. All the shared tasks just work.

## The problem: name collision

The monitoring stack has a requirement the other projects don't: before starting the stack, it needs to render a config file from a template, substituting SNMP credentials from a `.env` file. The natural place to hook this in is the `up` task.

But if both the root Taskfile and the monitoring Taskfile define `up`, Taskfile will complain about a duplicate task definition. A plain include doesn't let you replace tasks — it only adds them.

## The `excludes` option

This is exactly what `excludes` is for. Adding it to the includes block tells Taskfile: "import everything from this file, *except* these tasks."

```yaml
includes:
  root:
    taskfile: ../Taskfile.yml
    flatten: true
    excludes: [up]  # don't import this one, we define it ourselves
```

Now the root `up` is never imported, so the monitoring Taskfile can define its own without any conflict. All other tasks — `down`, `logs`, `ps`, `update`, `restart` — are still inherited as-is.

## The local `up` override

With the slot free, the monitoring Taskfile defines its own `up`:

```yaml
tasks:
  up:
    desc: Render config then start stack
    deps: [render-snmp-config]
    cmds:
      - "{{.SUDO}} {{.DOCKERCOMPOSE}} up -d --remove-orphans"
```

The `deps` line ensures `render-snmp-config` always runs before the stack starts. That task reads `.env` secrets and substitutes them into the SNMP exporter config template:

```yaml
  render-snmp-config:
    desc: Render snmp-exporter config from template using .env secrets
    cmds:
      - |
        set -a && source .env && set +a
        sed -e "s|\${SNMP_USERNAME}|$SNMP_USERNAME|g" \
            -e "s|\${SNMP_PASSWORD}|$SNMP_PASSWORD|g" \
            snmp-synology/snmp.yml.template > snmp-synology/snmp.yml
    sources:
      - snmp-synology/snmp.yml.template
      - .env
    generates:
      - snmp-synology/snmp.yml
```

## Bonus: idempotency with `sources` and `generates`

Notice the `sources` and `generates` fields on `render-snmp-config`. Taskfile uses these to track whether the task's inputs have changed since the last run. If neither the template nor the `.env` file has been touched, Taskfile skips the render entirely — a small but nice efficiency when you're doing `task up` repeatedly during development.

## The complete monitoring Taskfile

Here's the full `monitoring/Taskfile.yml` — clean and to the point:

```yaml
version: "3"

includes:
  root:
    taskfile: ../Taskfile.yml
    flatten: true
    excludes: [up] # don't import this one, we define it ourselves

tasks:
  render-snmp-config:
    desc: Render snmp-exporter config from template using .env secrets
    cmds:
      - |
        set -a && source .env && set +a
        sed -e "s|\${SNMP_USERNAME}|$SNMP_USERNAME|g" \
            -e "s|\${SNMP_PASSWORD}|$SNMP_PASSWORD|g" \
            snmp-synology/snmp.yml.template > snmp-synology/snmp.yml
    sources:
      - snmp-synology/snmp.yml.template
      - .env
    generates:
      - snmp-synology/snmp.yml

  clean:
    desc: Remove generated snmp-exporter config
    cmds:
      - rm -f snmp-synology/snmp.yml

  up:
    desc: Render config then start stack
    deps: [render-snmp-config]
    cmds:
      - "{{.SUDO}} {{.DOCKERCOMPOSE}} up -d --remove-orphans"

  restart:prom:
    desc: Restart prometheus container
    deps:
      - render-snmp-config
    cmds:
      - "{{.SUDO}} {{.DOCKERCOMPOSE}} restart prometheus"

  restart:snmp:
    desc: Restart snmp-exporter container
    deps:
      - render-snmp-config
    cmds:
      - "{{.SUDO}} {{.DOCKERCOMPOSE}} restart snmp-exporter"
```

## Conclusion

The `includes` + `flatten` + `excludes` combination gives you clean Taskfile inheritance: share a common set of tasks across all your projects, and selectively replace only the ones that need project-specific behaviour. No duplication, no namespace clutter, and the override is explicit and easy to spot.
