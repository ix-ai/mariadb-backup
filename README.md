# mariadb-backup

## Deprecation Notice

**This project is deprecated and has been archived**. Please switch to [gitlab.com/egos-tech/mariadb-backup](https://gitlab.com/egos-tech/mariadb-backup).

Replace your docker image with `registry.gitlab.com/egos-tech/mariadb-backup:latest`.

Please note, a new versioning format is established, starting with `1.0.0` - this version is one-to-one compatible with the latest version in this repository:

```yml
image: registry.gitlab.com/egos-tech/mariadb-backup:1.0.0
```

All future updates will only be done to that project.

## Description

[![Pipeline Status](https://gitlab.com/ix.ai/mariadb-backup/badges/master/pipeline.svg)](https://gitlab.com/ix.ai/mariadb-backup/)
[![Gitlab Project](https://img.shields.io/badge/GitLab-Project-554488.svg)](https://gitlab.com/ix.ai/mariadb-backup/)

The mariadb-backup Docker image will provide you a container to backup and restore a [MySQL](https://hub.docker.com/_/mysql/) or [MariaDB](https://hub.docker.com/_/mariadb/) database container.

The backup is made with [mydumper](http://centminmod.com/mydumper.html), a fast MySQL backup utility.

## Usage

To backup a [MySQL](https://hub.docker.com/_/mysql/) or [MariaDB](https://hub.docker.com/_/mariadb/) database, you simply specify the credentials and the host. You can optionally specify the database as well.

## Environment variables

| **Variable**  | **FILE Support** | **Default** | **Mandatory** | **Description**                                      |
|:--------------|:----------------:|:-----------:|:-------------:|:-----------------------------------------------------|
| `DB_HOST`     | `DB_HOST__FILE`  | -           | *yes*         | The host to connect to                               |
| `DB_PASS`     | `DB_PASS__FILE`  | -           | *yes*         | The password for the SQL server                      |
| `DB_NAME`     | `DB_NAME__FILE`  | -           | *no*          | If specified, only this database will be backed up   |
| `DB_PORT`     | N/A              | `3306`      | *no*          | The port of the SQL server                           |
| `DB_USER`     | `DB_USER__FILE`  | `root`      | *no*          | The user to connect to the SQL server                |
| `MODE`        | N/A              | `BACKUP`    | *no*          | One of `BACKUP` or `RESTORE`                         |
| `BASE_DIR`    | N/A              | `/backup`   | *no*          | Path of the base directory (aka working directory)   |
| `RESTORE_DIR` | N/A              | -           | *no*          | Name of a backup directory to restore                |
| `BACKUP_UID`  | N/A              | `666`       | *no*          | UID of the backup                                    |
| `BACKUP_GID`  | N/A              | `666`       | *no*          | GID of the backup                                    |
| `UMASK`       | N/A              | `0022`      | *no*          | Umask which should be used to write the backup files |
| `OPTIONS`     | N/A              | `-c` / `-o` | *no*          | Options passed to `mydumper` / `myloader`            |

The following environment variables support setting via `*__FILE`, where

Please note the backup will be written to `/backup` by default, so you might want to mount that directory from your host.

## Example Docker CLI client

To **create a backup** from a MySQL container via `docker` CLI client:

```bash
docker run --name my-backup -e DB_HOST=mariadb -e DB_PASS=amazing_pass -v /var/mysql_backups:/backup registry.gitlab.com/ix.ai/mariadb-backup:latest
```

The container will stop automatically as soon as the backup has finished.
To create more backups in the future simply start your container again:

```bash
docker start my-backup
```

To **restore a backup** into a MySQL container via `docker` CLI client:

```bash
docker run --name my-restore -e DB_HOST=mariadb -e DB_PASS=amazing_pass -e MODE=RESTORE -v /var/mysql_backups:/backup registry.gitlab.com/ix.ai/mariadb-backup:latest
```

## Script example

To back up multiple databases, all running in docker, all labeled with `mariadb-backup`:

```bash
#!/usr/bin/env bash
/bin/mkdir -p /mariadb-backup

/usr/bin/docker pull registry.gitlab.com/ix.ai/mariadb-backup:latest

for CONTAINER in $(/usr/bin/docker ps -f label=mariadb-backup --format='{{.Names}}'); do
  DB_PASS=$(/usr/bin/docker inspect ${CONTAINER}|/usr/bin/jq -r '.[0]|.Config.Env[]|select(test("^MARIADB_ROOT_PASSWORD.*"))'|/bin/sed -n 's/^MARIADB_ROOT_PASSWORD=\(.*\)/\1/p')
  DB_NAME=$(/usr/bin/docker inspect ${CONTAINER}|/usr/bin/jq -r '.[0]|.Config.Env[]|select(test("^MARIADB_DATABASE.*"))'|/bin/sed -n 's/^MARIADB_DATABASE=\(.*\)/\1/p')
  DB_NET=$(/usr/bin/docker inspect ${CONTAINER}|/usr/bin/jq -r '.[0]|.NetworkSettings.Networks|to_entries[]|.key')
  if [[ -n "${DB_PASS}" ]]; then
    /usr/bin/docker run --rm --name ${CONTAINER}-backup -e DB_PASS=${DB_PASS} -e DB_HOST=${CONTAINER} -e DB_NAME=${DB_NAME} --network ${DB_NET} -v /mariadb-backup:/backup registry.gitlab.com/ix.ai/mariadb-backup:latest
  fi
done

```

## Configuration

### Mode

By default the container backups the database.
However, you can change the mode of the container by setting the following environment variable:

* `MODE`: Sets the mode of the backup container while [`BACKUP`|`RESTORE`]

### Base directory

By default the base directory `/backup` is used.
However, you can overwrite that by setting the following environment variable:

* `BASE_DIR`: Path of the base directory (aka working directory)

### Restore directory

By default the container will automatically restore the latest backup found in `BASE_DIR`.
However, you can manually set the name of a backup directory underneath `BASE_DIR`:

* `RESTORE_DIR`: Name of a backup directory to restore

*This option is only required when the container runs in in `RESTORE` mode.*

### UID and GID

By default the backup will be written with UID and GID `666`.
However, you can overwrite that by setting the following environment variables:

* `BACKUP_UID`: UID of the backup
* `BACKUP_GID`: GID of the backup

### umask

By default a `umask` of `0022` will be used.
However, you can overwrite that by setting the following environment variable:

* `UMASK`: Umask which should be used to write the backup files

### mydumper / myloader CLI options

By default `mydumper` is invoked with the `-c` (compress backup) and `myloader` with the `-o` (overwrite tables) CLI option.
However, you can modify the CLI options by setting the following environment variable:

* `OPTIONS`: Options passed to `mydumper` (when `MODE` is `BACKUP`) or `myloader` (when `MODE` is `RESTORE`)

## Tags and Arch

Starting with version v0.0.3, the images are multi-arch, with builds for amd64, arm64, armv7. Support for armv6 has been dropped.

* `vN.N.N` - for example v0.0.2
* `latest` - always pointing to the latest version
* `dev-master` - the last build on the master branch

## Resources

* Gitlab Registry: `registry.gitlab.com/ix.ai/mariadb-backup` - [gitlab.com/ix.ai/mariadb-backup](https://gitlab.com/ix.ai/mariadb-backup)
* GitHub Registry: `ghcr.io/ix-ai/mariadb-backup` [github.com/ix-ai/mariadb-backup](https://github.com/ix-ai/mariadb-backup)
* Docker Hub: `ixdotai/mariadb-backup` - [hub.docker.com/r/ixdotai/mariadb-backup](https://hub.docker.com/r/ixdotai/mariadb-backup)

## Credits

Special thanks to [confirm/docker-mysql-backup](https://github.com/confirm/docker-mysql-backup), which this project uses heavily.
