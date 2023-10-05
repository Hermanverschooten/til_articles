visible
-- TITLE --
Mounting MySQL in Docker
-- TAGS --
mysql
database
docker
-- TLDR --
Mounting a copy of a MySQL data-directory in Docker
-- CONTENT --
# Mounting a copy of a MySQL data-directory in Docker

A customer called this morning, they had deleted a large part of their records in the database, can you please restore them?
Oh yeah, and don't touch the current records as we have made sales this morning.

The instance runs under ProxMox and is daily backuped to the Proxmox Backup Server.
I downloaded the `/var/lib/mysql` and `/etc/mysql` as a `zip` to my machine, and unzipped them in a working directory.

Initially I tried to run this using docker:
```elixir
docker run --rm -v $(pwd)/mysql:/var/lib/mysql -v $(pwd)/conf:/etc/mysql  mysql:8.0
```
Sadly this failed with:
```elixir
2023-10-05T12:38:43.399765Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.34) starting as process 1
2023-10-05T12:38:43.403373Z 0 [Warning] [MY-010159] [Server] Setting lower_case_table_names=2 because file system for /var/lib/mysql/ is case insensitive
2023-10-05T12:38:43.412236Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2023-10-05T12:38:44.351480Z 1 [System] [MY-013577] [InnoDB] InnoDB initialization has ended.
2023-10-05T12:38:44.366718Z 1 [ERROR] [MY-011087] [Server] Different lower_case_table_names settings for server ('2') and data dictionary ('0').
2023-10-05T12:38:44.366834Z 0 [ERROR] [MY-010020] [Server] Data Dictionary initialization failed.
2023-10-05T12:38:44.366843Z 0 [ERROR] [MY-010119] [Server] Aborting
2023-10-05T12:38:44.958349Z 0 [System] [MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.34)  MySQL Community Server - GPL.
```
This is caused by my machine being MacOS and the original server being a Linux, Linus has a case-sentive filesystem, Mac does not by default.

To resolve this you can create a sparse-file that has a case-sensitive filesystem and mount it.
```elixir
hdiutil create -type SPARSE -fs hfsx -size 4g -volname linux linux.hfsx.dmg.sparseimage
hdiutil attach -nobrowse -mountpoint linux linux.hfsx.dmg.sparseimage
```
Now move the unzipped data into this linux folder, and try again.

```elixir
docker run -v $(pwd)/linux:/var/lib/mysql -v $(pwd)/conf:/etc/mysql --name mysql  mysql:8.0
```
This works, now you can access your database again.

After you've finished, stop docker and eject.

```elixir
docker stop mysql
hdiutil eject linux
```

And finally, delete the sparse file.

To restore the data, we exported the 2 tables to new tables and re-added the recors with a `select into` from the old where the id's were missing.

A happy customer!
