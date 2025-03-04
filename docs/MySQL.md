# WAL-G for MySQL

**Interface of MySQL and MariaDB now is unstable**

You can use wal-g as a tool for encrypting, compressing MySQL backups and push/fetch them to/from storage without saving it on your filesystem.

Development
-----------
### Installing
To compile and build the binary for MySQL:

Optional:

- To build with libsodium, just set `USE_LIBSODIUM` environment variable.
- To build with lzo decompressor, just set `USE_LZO` environment variable.
```plaintext
go get github.com/wal-g/wal-g
cd $GOPATH/src/github.com/wal-g/wal-g
make install
make deps
make mysql_build
```
Users can also install WAL-G by using `make install`. Specifying the GOBIN environment variable before installing allows the user to specify the installation location. On default, `make install` puts the compiled binary in `go/bin`.
```plaintext
export GOBIN=/usr/local/bin
cd $GOPATH/src/github.com/wal-g/wal-g
make install
make deps
make mysql_install
```

Configuration
-------------

* `WALG_MYSQL_DATASOURCE_NAME`

To configure the connection string for MySQL. Required. Format ```user:password@host/dbname```

* `WALG_MYSQL_SSL_CA`

To use SSL, a path to file with certificates should be set to this variable.

*  `WALG_STREAM_CREATE_COMMAND`

Command to create MySQL backup, should return backup as single stream to STDOUT. Requried.

*  `WALG_STREAM_RESTORE_COMMAND`

Command to unpack MySQL backup, should take backup (created by `WALG_STREAM_CREATE_COMMAND`) 
to STDIN and unpack it to MySQL datadir. Required.

* `WALG_MYSQL_BACKUP_PREPARE_COMMAND`

Command to prepare MySQL backup after restoring. Optional. Needed for xtrabackup case.

* `WALG_MYSQL_BINLOG_REPLAY_COMMAND`

Command to replay binlog on runing MySQL. Required for binlog-fetch command.

* `WALG_MYSQL_BINLOG_DST`

To place binlogs in the specified directory during binlog-fetch or binlog-replay

* `WALG_MYSQL_TAKE_BINLOGS_FROM_MASTER`

Set this variable to True if you are planning to take base backup from replica and binlog backup from master.
If base and binlogs backups are taken from the same host, this variable should be left False (default).

> **Operations with binlogs**: If you'd like to do binlog operations with wal-g don't forget to [activate the binary log](https://mariadb.com/kb/en/activating-the-binary-log/) by starting mysql/mariadb with [--log-bin](https://mariadb.com/kb/en/replication-and-binary-log-server-system-variables/#log_bin) and [--log-basename](https://mariadb.com/kb/en/mysqld-options/#-log-basename)=\[name\].


Usage
-----

WAL-G mysql extension currently supports these commands:

### ``backup-push``

Creates new backup and send it to storage. Runs `WALG_STREAM_CREATE_COMMAND` to create backup.

```bash
wal-g backup-push
```

### ``backup-list``

Lists currently available backups in storage

```bash
wal-g backup-list
```

### ``backup-fetch``

Fetches backup from storage and restores it to datadir.
Runs `WALG_STREAM_RESTORE_COMMAND` to restore backup.
User should specify the name of the backup to fetch.

```bash
wal-g backup-fetch example_backup
```

WAL-G can also fetch the latest backup using:

```bash
wal-g backup-fetch  LATEST
```

### ``binlog-push``

Sends (not yet archived) binlogs to storage. Typically run in CRON.

```bash
wal-g binlog-push
```


When `WALG_MYSQL_CHECK_GTIDS` is set wal-g will try to be upload only binlogs which GTID sets contains events that
wasn't seen before. This is done by parsing binlogs and peeking first PREVIOUS_GTIDS_EVENT that holds GTID set of
all executed transactions at the moment this particular binlog file created.
This feature may be useful when you are uploading binlogs from different hosts (e.g. after master switchower)
Note: Don't use `WALG_MYSQL_CHECK_GTIDS` when GTIDs are not used - it will slow down binlog upload.

### ``binlog-fetch``

Fetches binlogs from storage and saves them to `WALG_MYSQL_BINLOG_DST` folder.
User should specify the name of the backup starting with which to fetch an binlog.
User may also specify time in  RFC3339 format until which should be fetched (used for PITR).
User have to replay binlogs manually in that case.

```bash
wal-g binlog-fetch --since "backupname"
```
or
```bash
wal-g binlog-fetch --since "backupname" --until "2006-01-02T15:04:05Z07:00"
```
or
```bash
wal-g binlog-fetch --since LATEST --until "2006-01-02T15:04:05Z07:00"
```

You can stop wal-g from fetching newly created/modified binlogs by specifying `--until-binlog-last-modified-time` option.
This may be useful to achieve exact clones of the same database in scenarios when new binlogs are uploaded concurrently whith your restore process.

```bash
wal-g binlog-replay --since LATEST --until "2006-01-02T15:04:05Z07:00" --until-binlog-last-modified-time "2006-01-02T15:04:05Z07:00"
```

### ``binlog-replay``

Fetches binlogs from storage and passes them to `WALG_MYSQL_BINLOG_REPLAY_COMMAND` to replay on running MySQL server.
User should specify the name of the backup starting with which to fetch an binlog.
User may also specify time in  RFC3339 format until which should be fetched (used for PITR).
If `until` timestamp is in the future, wal-g will search for newly uploaded binlogs until no new found.
Binlogs are temporarily save in `WALG_MYSQL_BINLOG_DST` folder.
Replay command gets name of binlog to replay via environment variable `WALG_MYSQL_CURRENT_BINLOG` and stop-date via `WALG_MYSQL_BINLOG_END_TS`, which are set for each invocation.

```bash
wal-g binlog-replay --since "backupname"
```
or
```bash
wal-g binlog-replay --since "backupname" --until "2006-01-02T15:04:05Z07:00"
```
or
```bash
wal-g binlog-replay --since LATEST --until "2006-01-02T15:04:05Z07:00"
```

You can stop wal-g from applying newly created/modified  binlogs by specifying `--until-binlog-last-modified-time` option.
This may be useful to achieve exact clones of the same database in scenarios when new binlogs are uploaded concurrently whith your restore process.

```bash
wal-g binlog-replay --since LATEST --until "2006-01-02T15:04:05Z07:00" --until-binlog-last-modified-time "2006-01-02T15:04:05Z07:00"
```

Typical configurations
-----

### MySQL - using with `xtrabackup`


It's recommended to use wal-g with xtrabackup tool in case of MySQL for creating lock-less backups.
Here's typical wal-g configuration for that case:
```bash
 WALG_MYSQL_DATASOURCE_NAME=user:pass@tcp(localhost:3306)/mysql                                                                                                                                      
 WALG_STREAM_CREATE_COMMAND="xtrabackup --backup --stream=xbstream --datadir=/var/lib/mysql"                                                                                                                               
 WALG_STREAM_RESTORE_COMMAND="xbstream -x -C /var/lib/mysql"                                                                                                                       
 WALG_MYSQL_BACKUP_PREPARE_COMMAND="xtrabackup --prepare --target-dir=/var/lib/mysql"                                                                                              
 WALG_MYSQL_BINLOG_REPLAY_COMMAND='mysqlbinlog --stop-datetime="$WALG_MYSQL_BINLOG_END_TS" "$WALG_MYSQL_CURRENT_BINLOG" | mysql'
```

Restore procedure is a bit tricky:
* stop mysql
* clean a datadir (typically `/var/lib/mysql`)
* fetch and prepare desired backup using `wal-g backup-fetch "backup_name"`
* start mysql
* in case of you have replication and GTID enabled: set mysql GTID_PURGED variable to value from `/var/lib/mysql/xtrabackup_binlog_info`, using
```bash
gtids=$(tr -d '\n' < /var/lib/mysql/xtrabackup_binlog_info | awk '{print $3}')
mysql -e "RESET MASTER; SET @@GLOBAL.GTID_PURGED='$gtids';"
```
* for PITR, replay binlogs with
```bash
wal-g binlog-replay --since "backup_name" --until "2006-01-02T15:04:05Z07:00"
```

### MySQL - using with `mysqldump`


It's possible to use wal-g with standard mysqldump/mysql tools.
In that case MySQL mysql backup is a plain SQL script.
Here's typical wal-g configuration for that case:

```bash
 WALG_MYSQL_DATASOURCE_NAME=user:pass@localhost/mysql                                                                                                               
 WALG_STREAM_CREATE_COMMAND="mysqldump --all-databases --single-transaction --set-gtid-purged=ON"                                                                                                                               
 WALG_STREAM_RESTORE_COMMAND="mysql"
 WALG_MYSQL_BINLOG_REPLAY_COMMAND='mysqlbinlog --stop-datetime="$WALG_MYSQL_BINLOG_END_TS" "$WALG_MYSQL_CURRENT_BINLOG" | mysql'
```

Restore procedure is straightforward:
* start mysql (it's recommended to create new mysql instance)
* fetch and apply desired backup using `wal-g backup-fetch "backup_name"`


### MariaDB - using with `mariabackup`

It's recommended to use wal-g with `mariabackup` tool in case of MariaDB for creating lock-less backups.
Here's typical wal-g configuration for that case:
```bash
 WALG_MYSQL_DATASOURCE_NAME=user:pass@tcp(localhost:3305)/mysql                                                                                                                                      
 WALG_STREAM_CREATE_COMMAND="mariabackup --backup --stream=xbstream --datadir=/var/lib/mysql"                                                                                                                               
 WALG_STREAM_RESTORE_COMMAND="mbstream -x -C /var/lib/mysql"                                                                                                                       
 WALG_MYSQL_BACKUP_PREPARE_COMMAND="mariabackup --prepare --target-dir=/var/lib/mysql"                                                                                              
 WALG_MYSQL_BINLOG_REPLAY_COMMAND='mysqlbinlog --stop-datetime="$WALG_MYSQL_BINLOG_END_TS" "$WALG_MYSQL_CURRENT_BINLOG" | mysql'
```

For the restore procedure you have to do similar things to [what the offical docs says about full backup and restore](https://mariadb.com/kb/en/full-backup-and-restore-with-mariabackup/):
* stop mariadb
* clean a datadir (typically `/var/lib/mysql`)
* fetch and prepare desired backup using `wal-g backup-fetch "backup_name"`
* after the previous step you might have to fix file permissions: `chown -R mysql:mysql /var/lib/mysql`
* start mariadb
* WAL-G doesn't support automatic PITR for MariaDB. There are 2 possible workarounds:
  * You can configure restored database to replicate from your master, so it will be able to catch up (follow the [official docs](https://mariadb.com/kb/en/setting-up-a-replication-slave-with-mariabackup/#gtids))
  * You can manually replay events with binlog and position:
```bash
wal-g binlog-fetch --since [backup name | LATEST]
# Get binlog-name, position and GTIDs:
tail -n 1 < /var/lib/mysql/xtrabackup_binlog_info
# eg 'mysql-bin.000005	385	0-1-5763'
# then replay it manually:
mysql -e "STOP ALL SLAVES; SET GLOBAL gtid_slave_pos='$gtids';"
mysqlbinlog --stop-datetime="some point in time" --start-position [position above] [all binlogs starting from thouse we seen above] | mysql --user XXX --host YYY [other options]
```

### MariaDB - using with `mysqldump`

The procedure is same as in case of [MySQL. You can follow the instructions from the previous section.](#mysql---using-with-mysqldump)
