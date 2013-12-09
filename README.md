# fluent-plugin-mysql-replicator [![Build Status](https://travis-ci.org/y-ken/fluent-plugin-mysql-replicator.png?branch=master)](https://travis-ci.org/y-ken/fluent-plugin-mysql-replicator)

## Overview

Fluentd input plugin to track insert/update/delete event from MySQL database server.

## Installation

`````
### native gem
gem install fluent-plugin-mysql-replicator

### td-agent gem
/usr/lib64/fluent/ruby/bin/fluent-gem install fluent-plugin-mysql-replicator
`````

## Tutorial for Quickstart

It is useful for these purpose.

* try it on this plugin.
* replicate small record under a millons table.

**Note:**  
On syncing 300 million rows table, it will consume around 800MB of memory with ruby 1.9.3 environment.

#### configuration

`````
<source>
  type mysql_replicator
  host localhost
  username your_mysql_user
  password your_mysql_password
  database myweb
  interval 5s
  tag replicator
  query SELECT id, text from search_test
</source>

<match replicator.*>
  type stdout
</match>
`````

#### sample query

`````
$ mysql -e "create database myweb"
$ mysql myweb -e "create table search_test(id int auto_increment, text text, PRIMARY KEY (id))"
$ sleep 10
$ mysql myweb -e "insert into search_test(text) values('aaa')"
$ sleep 10
$ mysql myweb -e "update search_test set text='bbb' where text = 'aaa'"
$ sleep 10
$ mysql myweb -e "delete from search_test where text='bbb'"
`````

#### result

`````
$ tail -f /var/log/td-agent/td-agent.log
2013-11-25 18:22:25 +0900 replicator.insert: {"id":"1","text":"aaa"}
2013-11-25 18:22:35 +0900 replicator.update: {"id":"1","text":"bbb"}
2013-11-25 18:22:45 +0900 replicator.delete: {"id":"1"}
`````

## Tutorial for Production

It is very useful to replicate a millions of records and/or multiple tables with multiple threads.  
This architecture is storing hash table in mysql management table instead of ruby internal memory.  

**Note:**  
On syncing 300 million rows table, it will consume around 20MB of memory with ruby 1.9.3 environment.

#### prepare

* create database and tables.
* add replicator configuration.

```
$ cat setup_mysql_replicator_multi.sql
CREATE DATABASE replicator_manager;
USE replicator_manager;

CREATE TABLE `hash_tables` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `setting_name` varchar(255) NOT NULL,
  `setting_query_pk` int(11) NOT NULL,
  `setting_query_hash` varchar(255) NOT NULL,
  PRIMARY KEY (`id`),
  UNIQUE KEY `setting_query_pk` (`setting_query_pk`,`setting_name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;

CREATE TABLE `settings` (
  `id` int(11) NOT NULL AUTO_INCREMENT,
  `name` varchar(255) NOT NULL,
  `host` varchar(255) NOT NULL DEFAULT 'localhost',
  `port` int(11) NOT NULL DEFAULT '3306',
  `username` varchar(255) NOT NULL,
  `password` varchar(255) NOT NULL,
  `database` varchar(255) NOT NULL,
  `query` TEXT NOT NULL,
  `interval` int(11) NOT NULL,
  `tag` varchar(255) NOT NULL,
  `primary_key` varchar(11) DEFAULT 'id',
  `enable_delete` int(11) DEFAULT '1',
  PRIMARY KEY (`id`),
  UNIQUE KEY `name` (`name`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8;
```

```
$ mysql
mysql> source /path/to/setup_mysql_replicator_multi.sql
mysql> insert into source ...snip...;
```

#### configuration

`````
<source>
  type mysql_replicator_multi
  manager_host localhost
  manager_username your_mysql_user
  manager_password your_mysql_password
  manager_database replicator_manager
</source>

<match replicator.*>
  type stdout
</match>
`````

## TODO

Pull requests are very welcome!!

## Copyright

Copyright © 2013- Kentaro Yoshida ([@yoshi_ken](https://twitter.com/yoshi_ken))

## License

Apache License, Version 2.0
