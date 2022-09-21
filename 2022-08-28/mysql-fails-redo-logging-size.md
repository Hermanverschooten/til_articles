visible
-- TITLE --
Mysql 8.0.30 fails with redo logfile size is not a multiple of innodb_page_size
-- TAGS --
mysql
ubuntu
-- TLDR --
This morning I reboot one of my VPS's, and MySQL does not start anymore.
It complains that the redo logfile is not a multiple of innodb_page_size.
Read the full article for how i solved this.
-- CONTENT --
# MySQL fails to start after auto-upgrading to 8.0.30.

Most of my VPS run on Ubuntu (20.04/22.04), and a couple of weeks ago, one of my customers had an issue.
The system is - by default - set to unattended upgrades. One of the upgrades it did was upgrading MySQL
server.  This had no effect, until the VPS was rebooted. Now it will not start anymore.
We restored from backup, took a backup from 8.0.29, dropped the database, recreated it and did a restore.

## When disaster strikes...

<span>This morning I had the same issue. &#128563;</span>

```elixir
2022-08-28T05:59:41.270083Z 0 [System] [MY-010116] [Server] /usr/sbin/mysqld (mysqld 8.0.30-0ubuntu0.20.04.2) starting as process 814
2022-08-28T05:59:41.280784Z 1 [System] [MY-013576] [InnoDB] InnoDB initialization has started.
2022-08-28T05:59:41.426273Z 1 [ERROR] [MY-012962] [InnoDB] The redo log file ./#innodb_redo/#ib_redo7 size 2449408 is not a multiple of innodb_page_size
2022-08-28T05:59:41.426359Z 1 [ERROR] [MY-012930] [InnoDB] Plugin initialization aborted with error Generic error.
2022-08-28T05:59:41.808322Z 1 [ERROR] [MY-010334] [Server] Failed to initialize DD Storage Engine
2022-08-28T05:59:41.808629Z 0 [ERROR] [MY-010020] [Server] Data Dictionary initialization failed.
2022-08-28T05:59:41.808673Z 0 [ERROR] [MY-010119] [Server] Aborting
2022-08-28T05:59:41.809287Z 0 [System] [MY-010910] [Server] /usr/sbin/mysqld: Shutdown complete (mysqld 8.0.30-0ubuntu0.20.04.2)  (Ubuntu).
```

So I restored my backup from last night, only to have the same issue reappearing.
OMG, the unattended upgrade happened on July 29th, a month ago. I do not have a backup, nor could I restore it, this server runs PowerDNS.

## Not to put a blame on...

The information I found back in July was that this was caused by running MySQL on top of ZFS on Ubuntu.
A bit of a strange conclusion. And since all my VPS run on ProxMox on ZFS volumes, not something I can easily remedy.

Looking at the release notes - _kind of_ - on [the MySQL dev site](https://dev.mysql.com/doc/relnotes/mysql/8.0/en/news-8-0-30.html), I read under `InnoDB` that it now supports dynamic configuration of redo logging.

But much more to the point is the single sentence almost at the end.

> As is generally required for any upgrade, this change requires a clean shutdown before upgrading.

This is much more likely the culprit.

## Not all is lost...

On [StackOverflow](https://stackoverflow.com/questions/73186360/innodb-redo0-is-not-a-multiple-of-innodb-page-size) I found an article with a possible solution.
The file should be a multiple of `innodb_page_size`, by default that would be `16384`. So how much too small is my file?

```elixir
2449408 % 16384 = 8192
```

Let's create a zeroed-file of that size and add it to the logfile.

```elixir
dd if=/dev/zero bs=1 count=8192 of=./zeros

# Append zeroes to invalid file
cat zeros >> /var/lib/mysql/#innodb_redo/#ib_redo0

# Restart MySQL
systemctl restart mysql.service
```

And luckily for me this worked as advertised.
