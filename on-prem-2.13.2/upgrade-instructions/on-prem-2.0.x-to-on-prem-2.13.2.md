# Scope AR

Upgrade version `on-prem-2.0.x` to `on-prem-2.13.2`.

This update may require several hours of downtime. Please notify any affected parties and plan accordingly.

## Recommended Requirements

* Docker version >= `19.03`

## Pre-Upgrade

* This release only supports one active company.
* All companies except the first company created will be disabled.

## Database Backup

* If an external database is being used (such as AWS [RDS](https://aws.amazon.com/rds/)) then you can create a backup there and skip these steps.
* These steps assume on-prem-2.0.x is currently running.

1. Change directory to `on-prem-2.0.x` files.

    ```bash
    $ cd on-prem-2.0.x
    ```

1. Prevent access to the site.

    ```bash
    $ docker service rm web_nginx
    ```

1. Find the running CMS container ID.

    ```bash
    $ CONTAINER_ID=$(docker ps | grep ' web_cms\.' | head -n 1 | cut -d' ' -f1)
    ```

1. Backup the database to file.

    ```bash
    $ docker exec -ti "$CONTAINER_ID" /bin/bash -c '. bin/secrets_to_env; secret_file_to_env > /dev/null 2>&1; bin/mysql_scope mysqldump' > scope-cms-database-2.0.x.sql
    ```

1. Verify backup was created successfully.

    ```bash
    $ head -n 5 scope-cms-database-2.0.x.sql
    ```

    ```sql
    -- MySQL dump 10.16  Distrib 10.1.32-MariaDB, for Linux (x86_64)
    --
    -- Host: mariadb    Database: scope_cms
    -- ------------------------------------------------------
    -- Server version       10.2.18-MariaDB-1:10.2.18+maria~bionic
    ...
    ```

## Stop Existing Stack

1. Stop existing stack.

    ```bash
    $ docker stack rm web
    $ docker stack rm comms
    $ docker stack rm proxy
    ```

## Update Docker Configuration Files

1. Change directory to `on-prem-2.0.x` files.

    ```bash
    $ cd on-prem-2.0.x
    ```

### CMS Front End

* File: `web.yml`

Update docker images.

Service Name|Image
---|---
`cms`|scopear/scope-cms:on-prem-2.13.2
`cms_resque`|scopear/scope-cms:on-prem-2.13.2
`nginx`|scopear/docker-nginx:on-prem-2.13.2
`redis_cms`|scopear/docker-redis:on-prem-2.13.2

#### Muxer

Add a new service for the muxer in `web.yml`. Adjust the volume `/shared/cms_private` to be the same volume which the `cms` service uses for `cms_private`.

```
services:
  ...
  muxer:
    image: scopear/docker-muxer:on-prem-2.13.2
    networks:
    - backend
    deploy:
      mode: global
      resources:
        limits:
          cpus: '0.75'
      update_config:
        parallelism: 1
        delay: 0s
        failure_action: rollback
        order: start-first
      rollback_config:
        parallelism: 0
        delay: 0s
        failure_action: pause
        order: start-first
    logging:
      driver: json-file
      options:
        max-size: 100m
        max-file: '5'
    volumes:
    - "/shared/cms_private:/cms/private"
    env_file:
    - muxer.env
```

Create an environment file for the muxer service called `muxer.env`:

```
# Muxer
# -----------------------------------------------------------------------------
REDIS=redis://redis_cms:6379/6
QUEUE=assemble_recordings
```

### Comms

* File: `comms.yml`

Update docker images.

Service Name|Image
---|---
`zeus`|scopear/docker-zeus:on-prem-2.13.2
`redis_zeus`|scopear/docker-redis:on-prem-2.13.2

### Proxy

* File: `proxy.yml`

Update docker images.

Service Name|Image
---|---
`turn`|scopear/docker-turnserver:on-prem-2.13.2
`nginx`|scopear/docker-nginx:on-prem-2.13.2

## Load Docker Images

If the server does not have an internet connection please see the **Installation Without the Internet** section of `README.md` and use the following list of images:

- `scopear/scope-cms:on-prem-2.13.2`
- `scopear/docker-muxer:on-prem-2.13.2`
- `scopear/docker-nginx:on-prem-2.13.2`
- `scopear/docker-db:on-prem-2.13.2`
- `scopear/docker-zeus:on-prem-2.13.2`
- `scopear/docker-redis:on-prem-2.13.2`
- `scopear/docker-turnserver:on-prem-2.13.2`

## Deploy

1. Change directory to `on-prem-2.0.x` files.

    ```bash
    $ cd on-prem-2.0.x
    ```

1. Remove previous cache manifest files from host file system (this directory is listed in `web.yml` under the `cms` service for `cms_public` volume).

    ```bash
    $ rm /shared/cms_public/assets/.sprockets-manifest-*
    ```

1. Deploy stacks:

    ```bash
    $ docker stack deploy --compose-file web.yml --prune --with-registry-auth web
    $ docker stack deploy --compose-file comms.yml --prune --with-registry-auth comms
    $ docker stack deploy --compose-file proxy.yml --prune --with-registry-auth proxy
    ```

This process may take some time and all Scope AR products will be intermittently unavailable.

1. To monitor migration progress, the list of services can be viewed. Some services can take several minutes to start and will show as `0` until their health check has passed.

    ```bash
    $ docker service ls
    ```

## Migrate Existing User Data

1. View the logs for all `cms` services.

    ```
    $ docker service logs -f --since 15m web_cms
    ```

1. Wait until all instances show the following message:

    ```
    -----------------------------------------------------------------------------
    Initialization ended
    -----------------------------------------------------------------------------
    ```

1. Find the running CMS container ID.

    ```bash
    $ CONTAINER_ID=$(docker ps | grep ' web_cms\.' | head -n 1 | cut -d' ' -f1)
    ```

## Troubleshooting

1. Check all ports which are listed under **Firewall Ports** in the `README` are open.

1. Check that the shared directory is mounted.

1. Check status of all services. Service name can be found under the `NAME` column for subsequent commands.

    ```bash
    $ docker service ls
    ```

1. If a container cannot start, check any error messages for the service.

    ```bash
    $ docker service ps --no-trunc web_cms
    ```

1. View the logs for a service.

    ```
    $ docker service logs --since 1h -f web_cms
    ```

## Verify installation

After the update process is complete, please verify the update by following the "Verifying a Proper Setup" section of the `README`.
