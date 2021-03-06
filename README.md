# SFTPGo

[![Build Status](https://travis-ci.org/drakkan/sftpgo.svg?branch=master)](https://travis-ci.org/drakkan/sftpgo) [![Code Coverage](https://codecov.io/gh/drakkan/sftpgo/branch/master/graph/badge.svg)](https://codecov.io/gh/drakkan/sftpgo/branch/master) [![Go Report Card](https://goreportcard.com/badge/github.com/drakkan/sftpgo)](https://goreportcard.com/report/github.com/drakkan/sftpgo) [![License: GPL v3](https://img.shields.io/badge/License-GPLv3-blue.svg)](https://www.gnu.org/licenses/gpl-3.0) [![Mentioned in Awesome Go](https://awesome.re/mentioned-badge.svg)](https://github.com/avelino/awesome-go)

Fully featured and highly configurable SFTP server, written in Go

## Features

- Each account is chrooted to its home directory.
- SFTP accounts are virtual accounts stored in a "data provider".
- SQLite, MySQL, PostgreSQL, bbolt (key/value store in pure Go) and in-memory data providers are supported.
- Public key and password authentication. Multiple public keys per user are supported.
- SSH user [certificate authentication](https://cvsweb.openbsd.org/src/usr.bin/ssh/PROTOCOL.certkeys?rev=1.8).
- Keyboard interactive authentication. You can easily setup a customizable multi-factor authentication.
- Partial authentication. You can configure multi-step authentication requiring, for example, the user password after successful public key authentication.
- Per user authentication methods. You can configure the allowed authentication methods for each user.
- Custom authentication via external programs/HTTP API is supported.
- Dynamic user modification before login via external programs/HTTP API is supported.
- Quota support: accounts can have individual quota expressed as max total size and/or max number of files.
- Bandwidth throttling is supported, with distinct settings for upload and download.
- Per user maximum concurrent sessions.
- Per user and per directory permission management: list directory contents, upload, overwrite, download, delete, rename, create directories, create symlinks, change owner/group and mode, change access and modification times.
- Per user files/folders ownership mapping: you can map all the users to the system account that runs SFTPGo (all platforms are supported) or you can run SFTPGo as root user and map each user or group of users to a different system account (\*NIX only).
- Per user IP filters are supported: login can be restricted to specific ranges of IP addresses or to a specific IP address.
- Per user and per directory file extensions filters are supported: files can be allowed or denied based on their extensions.
- Virtual folders are supported: directories outside the user home directory can be exposed as virtual folders.
- Configurable custom commands and/or HTTP notifications on file upload, download, delete, rename, on SSH commands and on user add, update and delete.
- Automatically terminating idle connections.
- Atomic uploads are configurable.
- Support for Git repositories over SSH.
- SCP and rsync are supported.
- Support for serving local filesystem, S3 Compatible Object Storage and Google Cloud Storage over SFTP/SCP.
- [Prometheus metrics](./docs/metrics.md) are exposed.
- Support for HAProxy PROXY protocol: you can proxy and/or load balance the SFTP/SCP service without losing the information about the client's address.
- [REST API](./docs/rest-api.md) for users management, backup, restore and real time reports of the active connections with possibility of forcibly closing a connection.
- [Web based administration interface](./docs/web-admin.md) to easily manage users and connections.
- Easy [migration](./examples/rest-api-cli#convert-users-from-other-stores) from Linux system user accounts.
- [Portable mode](./docs/portable-mode.md): a convenient way to share a single directory on demand.
- Performance analysis using built-in [profiler](./docs/profiling.md).
- Configuration format is at your choice: JSON, TOML, YAML, HCL, envfile are supported.
- Log files are accurate and they are saved in the easily parsable JSON format ([more information](./docs/logs.md)).

## Platforms

SFTPGo is developed and tested on Linux. After each commit, the code is automatically built and tested on Linux and macOS using Travis CI.
The test cases are regularly manually executed and passed on Windows. Other UNIX variants such as \*BSD should work too.

## Requirements

- Go 1.13 or higher as build only dependency.
- A suitable SQL server or key/value store to use as data provider: PostgreSQL 9.4+ or MySQL 5.6+ or SQLite 3.x or bbolt 1.3.x

## Installation

Binary releases for Linux, macOS, and Windows are available. Please visit the [releases](https://github.com/drakkan/sftpgo/releases "releases") page.

Sample Dockerfiles for [Debian](https://www.debian.org) and [Alpine](https://alpinelinux.org) are available inside the source tree [docker](./docker) directory.

Some Linux distro packages are available:

- For Arch Linux via AUR:
  - [sftpgo](https://aur.archlinux.org/packages/sftpgo/). This package follows stable releases. It requires `git`, `gcc` and `go` to build.
  - [sftpgo-bin](https://aur.archlinux.org/packages/sftpgo-bin/). This package follows stable releases downloading the prebuilt linux binary from GitHub. It does not require `git`, `gcc` and `go` to build.
  - [sftpgo-git](https://aur.archlinux.org/packages/sftpgo-git/). This package builds and installs the latest git master. It requires `git`, `gcc` and `go` to build.

Alternately, you can [build from source](./docs/build-from-source.md).

## Configuration

A full explanation of all configuration methods can be found [here](./docs/full-configuration.md).

Please make sure to [initialize the data provider](#data-provider-initialization) before running the daemon!

To start the SFTP server with default settings, simply run:

```bash
sftpgo serve
```

Check out [this documentation](./docs/service.md) if you want to run SFTPGo as a service.

### Data provider initialization

Before starting the SFTPGo server, please ensure that the configured data provider is properly initialized.

SQL based data providers (SQLite, MySQL, PostgreSQL) require the creation of a database containing the required tables. Memory and bolt data providers do not require an initialization.

After configuring the data provider using the configuration file, you can create the required database structure using the `initprovider` command.
For SQLite provider, the `initprovider` command will auto create the database file, if missing, and the required tables.
For PostgreSQL and MySQL providers, you need to create the configured database, and the `initprovider` command will create the required tables.

For example, you can simply execute the following command from the configuration directory:

```bash
sftpgo initprovider
```

Take a look at the CLI usage to learn how to specify a different configuration file:

```bash
sftpgo initprovider --help
```

The `initprovider` command is enough for new installations. From now on, the database structure will be automatically checked and updated, if required, at startup.

#### Upgrading

If you are upgrading from version 0.9.5 or before, you have to manually execute the SQL scripts to create the required database structure. These scripts can be found inside the source tree [sql](./sql "sql") directory. The SQL scripts filename is, by convention, the date as `YYYYMMDD` and the suffix `.sql`. You need to apply all the SQL scripts for your database ordered by name. For example, `20190828.sql` must be applied before `20191112.sql`, and so on.
Example for SQLite: `find sql/sqlite/ -type f -iname '*.sql' -print | sort -n | xargs cat | sqlite3 sftpgo.db`.
After applying these scripts, your database structure is the same as the one obtained using `initprovider` for new installations, so from now on, you don't have to manually upgrade your database anymore.

## Authentication options

### External Authentication

Custom authentication methods can easily be added. SFTPGo supports external authentication modules, and writing a new backend can be as simple as a few lines of shell script. More information can be found [here](./docs/external-auth.md).

### Keyboard Interactive Authentication

Keyboard interactive authentication is, in general, a series of questions asked by the server with responses provided by the client.
This authentication method is typically used for multi-factor authentication.

More information can be found [here](./docs/keyboard-interactive.md).

## Dynamic user creation or modification

A user can be created or modified by an external program just before the login. More information about this can be found [here](./docs/dynamic-user-mod.md).

## Custom Actions

SFTPGo allows to configure custom commands and/or HTTP notifications on file upload, download, delete, rename, on SSH commands and on user add, update and delete.

More information about custom actions can be found [here](./docs/custom-actions.md).

## Storage backends

### S3 Compabible Object Storage backends

Each user can be mapped to whole bucket or to a bucket virtual folder. This way, the mapped bucket/virtual folder is exposed over SFTP/SCP. More information about S3 integration can be found [here](./docs/s3.md).

### Google Cloud Storage backend

Each user can be mapped with a Google Cloud Storage bucket or a bucket virtual folder. This way, the mapped bucket/virtual folder is exposed over SFTP/SCP. More information about Google Cloud Storage integration can be found [here](./docs/google-cloud-storage.md).

### Other Storage backends

Adding new storage backends is quite easy:

- implement the [Fs interface](./vfs/vfs.go#L18 "interface for filesystem backends").
- update the user method `GetFilesystem` to return the new backend
- update the web interface and the REST API CLI
- add the flags for the new storage backed to the `portable` mode

Anyway, some backends require a pay per use account (or they offer free account for a limited time period only). To be able to add support for such backends or to review pull requests, please provide a test account. The test account must be available for enough time to be able to maintain the backend and do basic tests before each new release.

## Brute force protection

The [connection failed logs](./docs/logs.md) can be used for integration in tools such as [Fail2ban](http://www.fail2ban.org/). Example of [jails](./fail2ban/jails) and [filters](./fail2ban/filters) working with `systemd`/`journald` are available in fail2ban directory.

## Account's configuration properties

Details information about account configuration properties can be found [here](./docs/account.md).

## Performance

SFTPGo can easily saturate a Gigabit connection on low end hardware with no special configuration, this is generally enough for most use cases.

More in-depth analysis of performance can be found [here](./docs/performance.md).

## Acknowledgements

- [pkg/sftp](https://github.com/pkg/sftp)
- [go-chi](https://github.com/go-chi/chi)
- [zerolog](https://github.com/rs/zerolog)
- [lumberjack](https://gopkg.in/natefinch/lumberjack.v2)
- [argon2id](https://github.com/alexedwards/argon2id)
- [go-sqlite3](https://github.com/mattn/go-sqlite3)
- [go-sql-driver/mysql](https://github.com/go-sql-driver/mysql)
- [bbolt](https://github.com/etcd-io/bbolt)
- [lib/pq](https://github.com/lib/pq)
- [viper](https://github.com/spf13/viper)
- [cobra](https://github.com/spf13/cobra)
- [xid](https://github.com/rs/xid)
- [nathanaelle/password](https://github.com/nathanaelle/password)
- [PipeAt](https://github.com/eikenb/pipeat)
- [ZeroConf](https://github.com/grandcat/zeroconf)
- [SB Admin 2](https://github.com/BlackrockDigital/startbootstrap-sb-admin-2)
- [shlex](https://github.com/google/shlex)
- [go-proxyproto](https://github.com/pires/go-proxyproto)

Some code was initially taken from [Pterodactyl sftp server](https://github.com/pterodactyl/sftp-server)

## License

GNU GPLv3
