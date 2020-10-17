# 修改 Django ORM 底层连接数据库为连接池方式



### 目录

- [一、概述](https://blog.csdn.net/cybeyond_xuan/article/details/88641282?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf#_1)
- [二、安装 djorm-ext-pool](https://blog.csdn.net/cybeyond_xuan/article/details/88641282?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf#_djormextpool_7)
- [三、创建 APP](https://blog.csdn.net/cybeyond_xuan/article/details/88641282?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf#_APP_17)
- [四、配置 settings.py](https://blog.csdn.net/cybeyond_xuan/article/details/88641282?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf#_settingspy_151)
- [五、修改 MySQL 配置文件](https://blog.csdn.net/cybeyond_xuan/article/details/88641282?utm_medium=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf&depth_1-utm_source=distribute.pc_relevant.none-task-blog-BlogCommendFromMachineLearnPai2-1.add_param_isCf#_MySQL__174)



# 一、概述

在使用 Django 进行 Web 开发时， 我们避免不了与数据库打交道。 当并发量低的时候， 不会有任何问题。 但一旦并发量达到一定数量， 就会导致 数据库的连接数会被瞬时占满。 这将导致一个严重的后果 ———— 其他应用， 或者 Django 本身的其他服务都无法访问数据库。 这是不可容忍的！

造成这样的结果的原因之一是 Django 底层与数据库的连接方式并不是连接池。 它会为每一次的数据库访问建立连接。 这样当并发量上来以后， 数据库的连接数就不够用了。 所以本篇博客介绍一种修改 Django 底层与数据库的连接方式为连接池的形式， 避免出现这种灾难性的后果。 （本文数据库采用 MySQL）

# 二、安装 djorm-ext-pool

可以使用命令行的方式安装：

```
pip install djorm-ext-pool
```

也可以直接使用 Pycharm 进行搜索后安装。

# 三、创建 APP

创建一个名为 djorm_pool 的 APP， 在 **init**.py 文件中写如下语句（原封不动的复制进去就可以， 此 APP 中只有一个 **init**.py 文件就可以）：

```python
# -*- coding: utf-8 -*-

import logging
from functools import partial

from django.core.exceptions import ImproperlyConfigured
from django.conf import settings

from sqlalchemy import exc
from sqlalchemy import event
from sqlalchemy.pool import manage
from sqlalchemy.pool import Pool
from sqlalchemy.event import listens_for


POOL_PESSIMISTIC_MODE = getattr(settings, "DJORM_POOL_PESSIMISTIC", False)
POOL_SETTINGS = getattr(settings, 'DJORM_POOL_OPTIONS', {})
POOL_SETTINGS.setdefault("recycle", 3600)

logger = logging.getLogger('djorm.pool')


@event.listens_for(Pool, "checkout")
def _on_checkout(dbapi_connection, connection_record, connection_proxy):
    logger.debug("connection retrieved from pool")

    if POOL_PESSIMISTIC_MODE:
        cursor = dbapi_connection.cursor()
        try:
            cursor.execute("SELECT 1")
        except:
            # raise DisconnectionError - pool will try
            # connecting again up to three times before raising.
            raise exc.DisconnectionError()
        finally:
            cursor.close()


@event.listens_for(Pool, "checkin")
def _on_checkin(*args, **kwargs):
    logger.debug("connection returned to pool")


@event.listens_for(Pool, "connect")
def _on_connect(*args, **kwargs):
    logger.debug("connection created")


def patch_mysql():
    class hashabledict(dict):
        def __hash__(self):
            return hash(tuple(sorted(self.items())))

    class hashablelist(list):
        def __hash__(self):
            return hash(tuple(sorted(self)))

    class ManagerProxy(object):
        def __init__(self, manager):
            self.manager = manager

        def __getattr__(self, key):
            return getattr(self.manager, key)

        def connect(self, *args, **kwargs):
            if 'conv' in kwargs:
                conv = kwargs['conv']
                if isinstance(conv, dict):
                    items = []
                    for k, v in conv.items():
                        if isinstance(v, list):
                            v = hashablelist(v)
                        items.append((k, v))
                    kwargs['conv'] = hashabledict(items)
            if 'ssl' in kwargs:
                ssl = kwargs['ssl']
                if isinstance(ssl, dict):
                    items = []
                    for k, v in ssl.items():
                        if isinstance(v, list):
                            v = hashablelist(v)
                        items.append((k, v))
                    kwargs['ssl'] = hashabledict(items)
            return self.manager.connect(*args, **kwargs)

    try:
        from django.db.backends.mysql import base as mysql_base
    except (ImproperlyConfigured, ImportError) as e:
        return

    if not hasattr(mysql_base, "_Database"):
        mysql_base._Database = mysql_base.Database
        mysql_base.Database = ManagerProxy(manage(mysql_base._Database, **POOL_SETTINGS))


def patch_postgresql():
    try:
        from django.db.backends.postgresql_psycopg2 import base as pgsql_base
    except (ImproperlyConfigured, ImportError) as e:
        return

    if not hasattr(pgsql_base, "_Database"):
        pgsql_base._Database = pgsql_base.Database
        pgsql_base.Database = manage(pgsql_base._Database, **POOL_SETTINGS)


def patch_sqlite3():
    try:
        from django.db.backends.sqlite3 import base as sqlite3_base
    except (ImproperlyConfigured, ImportError) as e:
        return

    if not hasattr(sqlite3_base, "_Database"):
        sqlite3_base._Database = sqlite3_base.Database
        sqlite3_base.Database = manage(sqlite3_base._Database, **POOL_SETTINGS)


def patch_all():
    patch_mysql()
    patch_postgresql()
    patch_sqlite3()


patch_all()

```

接下来我们就可以配置 [settings.py](http://settings.py/) 文件了。

# 四、配置 [settings.py](http://settings.py/)

打开 [settings.py](http://settings.py/) 文件， 按照如下要求进行配置：

```python
INSTALLED_APPS = [
	...
	'djorm_pool',   # 将刚创建的项目添加进 INSTALLED_APPS
]

DJORM_POOL_OPTION = {
	'pool_size': 20,   # 本连接池的最大连接个数
	'max_overflow': 0, 
	'recycle': 3600
}

DJORM_POOL_PESSIMISTIC = True
```

这样 [settings.py](http://settings.py/) 就配置好了。

接下来需要修改一下 MySQL 的配置文件。

# 五、修改 MySQL 配置文件

将 MySQL 的配置文件 mysqld.cnf 按如下参数进行配置：

```python
#
# The MySQL database server configuration file.
#
# You can copy this to one of:
# - "/etc/mysql/my.cnf" to set global options,
# - "~/.my.cnf" to set user-specific options.
# 
# One can use all long options that the program supports.
# Run program with --help to get a list of available options and with
# --print-defaults to see which it would actually understand and use.
#
# For explanations see
# http://dev.mysql.com/doc/mysql/en/server-system-variables.html

# This will be passed to all mysql clients
# It has been reported that passwords should be enclosed with ticks/quotes
# escpecially if they contain "#" chars...
# Remember to edit /etc/mysql/debian.cnf when changing the socket location.

# Here is entries for some specific programs
# The following values assume you have at least 32M ram

[mysqld_safe]
socket		= /var/run/mysqld/mysqld.sock
nice		= 0

[mysqld]
#
# * Basic Settings
#
user		= mysql
pid-file	= /var/run/mysqld/mysqld.pid
socket		= /var/run/mysqld/mysqld.sock
port		= 3306
basedir		= /usr
datadir		= /var/lib/mysql
tmpdir		= /tmp
lc-messages-dir	= /usr/share/mysql
skip-external-locking
open_files_limit = 10240
back_log = 600 
external-locking = FALSE
#
# Instead of skip-networking the default is now to listen only on
# localhost which is more compatible and is not less secure.
#bind-address		= 127.0.0.1
#
# * Fine Tuning
#
key_buffer_size		= 32M
max_allowed_packet	= 32M
thread_stack		= 192K
thread_cache_size   = 300
# This replaces the startup script and checks MyISAM tables if needed
# the first time they are touched
myisam-recover-options  = BACKUP
max_connections = 3000
max_connect_errors = 6000
#thread_concurrency     = 10
#
# * Query Cache Configuration
#
query_cache_limit	= 4M
query_cache_size    = 64M
query_cache_min_res_unit = 2k
#default-storage-engine = MyISAM
#
# * Logging and Replication
#
# Both location gets rotated by the cronjob.
# Be aware that this log type is a performance killer.
# As of 5.1 you can enable the log at runtime!
#general_log_file        = /var/log/mysql/mysql.log
#general_log             = 1
#
# Error log - should be very few entries.
#
log_error = /var/log/mysql/error.log
#
# Here you can see queries with especially long duration
#log_slow_queries	= /var/log/mysql/mysql-slow.log
#long_query_time = 2
#log-queries-not-using-indexes
#
# The following can be used as easy to replay backup logs or for replication.
# note: if you are setting up a replication slave, see README.Debian about
#       other settings you may need to change.
#server-id		= 1
#log_bin			= /var/log/mysql/mysql-bin.log
expire_logs_days	= 10
max_binlog_size   = 100M
#binlog_do_db		= include_database_name
#binlog_ignore_db	= include_database_name
#
# * InnoDB
#
# InnoDB is enabled by default with a 10MB datafile in /var/lib/mysql/.
# Read the manual for more InnoDB related options. There are many!
#
# * Security Features
#
# Read the manual, too, if you want chroot!
# chroot = /var/lib/mysql/
#
# For generating SSL certificates I recommend the OpenSSL GUI "tinyca".
#
# ssl-ca=/etc/mysql/cacert.pem
# ssl-cert=/etc/mysql/server-cert.pem
# ssl-key=/etc/mysql/server-key.pem


transaction_isolation = READ-COMMITTED   
# 设定默认的事务隔离级别.可用的级别如下:
# READ-UNCOMMITTED, READ-COMMITTED, REPEATABLE-READ, SERIALIZABLE
# 1.READ UNCOMMITTED-读未提交2.READ COMMITTE-读已提交3.REPEATABLE READ -可重复读4.SERIALIZABLE -串行

tmp_table_size = 256M   
# tmp_table_size 的默认大小是 32M。如果一张临时表超出该大小，MySQL产生一个 The table tbl_name is full 形式的错误，如果你做很多高级 GROUP BY 查询，增加 tmp_table_size 值。如果超过该值，则会将临时表写入磁盘。
max_heap_table_size = 256M
long_query_time = 2
#log-bin = /data/3306/mysql-bin
binlog_cache_size = 4M
max_binlog_cache_size = 8M
max_binlog_size = 512M

expire_logs_days = 7
key_buffer_size = 2048M 
#批定用于索引的缓冲区大小，增加它可以得到更好的索引处理性能，对于内存在4GB左右的服务器来说，该参数可设置为256MB或384MB。

read_buffer_size = 1M  
# MySql读入缓冲区大小。对表进行顺序扫描的请求将分配一个读入缓冲区，MySql会为它分配一段内存缓冲区。read_buffer_size变量控制这一缓冲区的大小。如果对表的顺序扫描请求非常频繁，并且你认为频繁扫描进行得太慢，可以通过增加该变量值以及内存缓冲区大小提高其性能。和sort_buffer_size一样，该参数对应的分配内存也是每个连接独享。

read_rnd_buffer_size = 16M   
# MySql的随机读（查询操作）缓冲区大小。当按任意顺序读取行时(例如，按照排序顺序)，将分配一个随机读缓存区。进行排序查询时，MySql会首先扫描一遍该缓冲，以避免磁盘搜索，提高查询速度，如果需要排序大量数据，可适当调高该值。但MySql会为每个客户连接发放该缓冲空间，所以应尽量适当设置该值，以避免内存开销过大。

bulk_insert_buffer_size = 64M   
#批量插入数据缓存大小，可以有效提高插入效率，默认为8M

myisam_sort_buffer_size = 128M   
# MyISAM表发生变化时重新排序所需的缓冲

myisam_max_sort_file_size = 10G   
# MySQL重建索引时所允许的最大临时文件的大小 (当 REPAIR, ALTER TABLE 或者 LOAD DATA INFILE).
# 如果文件大小比此值更大,索引会通过键值缓冲创建(更慢)

myisam_repair_threads = 1   
# 如果一个表拥有超过一个索引, MyISAM 可以通过并行排序使用超过一个线程去修复他们.
# 这对于拥有多个CPU以及大量内存情况的用户,是一个很好的选择.

#自动检查和修复没有适当关闭的 MyISAM 表
lower_case_table_names = 1
#这个参数用来设置 InnoDB 存储的数据目录信息和其它内部数据结构的内存池大小，类似于Oracle的library cache。这不是一个强制参数，可以被突破。
innodb_buffer_pool_size = 256M   
# 这对Innodb表来说非常重要。Innodb相比MyISAM表对缓冲更为敏感。MyISAM可以在默认的 key_buffer_size 设置下运行的可以，然而Innodb在默认的 innodb_buffer_pool_size 设置下却跟蜗牛似的。由于Innodb把数据和索引都缓存起来，无需留给操作系统太多的内存，因此如果只需要用Innodb的话则可以设置它高达 70-80% 的可用内存。一些应用于 key_buffer 的规则有 — 如果你的数据量不大，并且不会暴增，那么无需把 innodb_buffer_pool_size 设置的太大了

#innodb_data_file_path = ibdata1:1024M:autoextend   
#表空间文件 重要数据


innodb_thread_concurrency = 8   
#服务器有几个CPU就设置为几，建议用默认设置，一般为8.

innodb_flush_log_at_trx_commit = 2   
# 如果将此参数设置为1，将在每次提交事务后将日志写入磁盘。为提供性能，可以设置为0或2，但要承担在发生故障时丢失数据的风险。设置为0表示事务日志写入日志文件，而日志文件每秒刷新到磁盘一次。设置为2表示事务日志将在提交时写入日志，但日志文件每次刷新到磁盘一次。

innodb_log_buffer_size = 16M  
#此参数确定些日志文件所用的内存大小，以M为单位。缓冲区更大能提高性能，但意外的故障将会丢失数据.MySQL开发人员建议设置为1－8M之间

innodb_log_file_size = 128M   
#此参数确定数据日志文件的大小，以M为单位，更大的设置可以提高性能，但也会增加恢复故障数据库所需的时间

innodb_log_files_in_group = 3   
#为提高性能，MySQL可以以循环方式将日志文件写到多个文件。推荐设置为3M

innodb_max_dirty_pages_pct = 90   
#推荐阅读 http://www.taobaodba.com/html/221_innodb_max_dirty_pages_pct_checkpoint.html
# Buffer_Pool中Dirty_Page所占的数量，直接影响InnoDB的关闭时间。参数innodb_max_dirty_pages_pct 可以直接控制了Dirty_Page在Buffer_Pool中所占的比率，而且幸运的是innodb_max_dirty_pages_pct是可以动态改变的。所以，在关闭InnoDB之前先将innodb_max_dirty_pages_pct调小，强制数据块Flush一段时间，则能够大大缩短 MySQL关闭的时间。

innodb_lock_wait_timeout = 120   
# InnoDB 有其内置的死锁检测机制，能导致未完成的事务回滚。但是，如果结合InnoDB使用MyISAM的lock tables 语句或第三方事务引擎,则InnoDB无法识别死锁。为消除这种可能性，可以将innodb_lock_wait_timeout设置为一个整数值，指示 MySQL在允许其他事务修改那些最终受事务回滚的数据之前要等待多长时间(秒数)

innodb_file_per_table = 0   
```

我们可以采用高并发的方式测试， 博主采用的是每秒写入 20 条数据。 经过一晚上的测试（大概 10 个小时）， 数据库中已存入几万条数据， 但是连接数一直保持在 13 个。

大功告成！

参考Git 地址： https://github.com/djangonauts/djorm-ext-pool.git

摘自 https://blog.csdn.net/cybeyond_xuan/article/details/88641282

