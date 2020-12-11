---
title: 在python multiprocessing情况下使用SQLAlchemy
date: 2017-09-20 11:06:51
tags: SQLAlchemy, multiprocessing
---

最近有个ETL任务，单独跑的时候很正常，在使用多进程并发执行时，出现了各种 mysql 客户端的问题，如：

```
(_mysql_exceptions.ProgrammingError) (2014, "Commands out of sync; you can't run this command now")

NoSuchColumnError: "Could not locate column in row for column 'config_powersaving.id'"
```

在初始化的时候，已经使用 scoped_session 包装了数据库的连接，使每个线程独立创建一个数据库连接，应该没有并发问题。

在google上翻阅各类解决方案都无法解决之后，发现官网本身提供了解决方案，原文如下：

> The key goal with multiple python processes is to prevent any database connections from being shared across processes. Depending on specifics of the driver and OS, the issues that arise here range from non-working connections to socket connections that are used by multiple processes concurrently, leading to broken messaging (the latter case is typically the most common).
>
> The SQLAlchemy [`Engine`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Engine) object refers to a connection pool of existing database connections. So when this object is replicated to a child process, the goal is to ensure that no database connections are carried over. There are three general approaches to this:
>
> 1. Disable pooling using [`NullPool`](http://docs.sqlalchemy.org/en/rel_1_0/core/pooling.html#sqlalchemy.pool.NullPool). This is the most simplistic, one shot system that prevents the [`Engine`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Engine) from using any connection more than once.
>
> 2. Call [`Engine.dispose()`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Engine.dispose) on any given [`Engine`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Engine) as soon one is within the new process. In Python multiprocessing, constructs such as`multiprocessing.Pool` include “initializer” hooks which are a place that this can be performed; otherwise at the top of where `os.fork()` or where the `Process` object begins the child fork, a single call to [`Engine.dispose()`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Engine.dispose) will ensure any remaining connections are flushed.
>
> 3. An event handler can be applied to the connection pool that tests for connections being shared across process boundaries, and invalidates them. This looks like the following:
>
>    ```
>    import os
>    import warnings
>
>    from sqlalchemy import event
>    from sqlalchemy import exc
>
>    def add_engine_pidguard(engine):
>        """Add multiprocessing guards.
>
>        Forces a connection to be reconnected if it is detected
>        as having been shared to a sub-process.
>
>        """
>
>        @event.listens_for(engine, "connect")
>        def connect(dbapi_connection, connection_record):
>            connection_record.info['pid'] = os.getpid()
>
>        @event.listens_for(engine, "checkout")
>        def checkout(dbapi_connection, connection_record, connection_proxy):
>            pid = os.getpid()
>            if connection_record.info['pid'] != pid:
>                # substitute log.debug() or similar here as desired
>                warnings.warn(
>                    "Parent process %(orig)s forked (%(newproc)s) with an open "
>                    "database connection, "
>                    "which is being discarded and recreated." %
>                    {"newproc": pid, "orig": connection_record.info['pid']})
>                connection_record.connection = connection_proxy.connection = None
>                raise exc.DisconnectionError(
>                    "Connection record belongs to pid %s, "
>                    "attempting to check out in pid %s" %
>                    (connection_record.info['pid'], pid)
>                )
>    ```
>
>    These events are applied to an [`Engine`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Engine) as soon as its created:
>
>    ```
>    engine = create_engine("...")
>
>    add_engine_pidguard(engine)
>    ```
>
> The above strategies will accommodate the case of an [`Engine`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Engine) being shared among processes. However, for the case of a transaction-active [`Session`](http://docs.sqlalchemy.org/en/rel_1_0/orm/session_api.html#sqlalchemy.orm.session.Session) or [`Connection`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Connection) being shared, there’s no automatic fix for this; an application needs to ensure a new child process only initiate new [`Connection`](http://docs.sqlalchemy.org/en/rel_1_0/core/connections.html#sqlalchemy.engine.Connection) objects and transactions, as well as ORM [`Session`](http://docs.sqlalchemy.org/en/rel_1_0/orm/session_api.html#sqlalchemy.orm.session.Session) objects. For a [`Session`](http://docs.sqlalchemy.org/en/rel_1_0/orm/session_api.html#sqlalchemy.orm.session.Session) object, technically this is only needed if the session is currently transaction-bound, however the scope of a single [`Session`](http://docs.sqlalchemy.org/en/rel_1_0/orm/session_api.html#sqlalchemy.orm.session.Session) is in any case intended to be kept within a single call stack in any case (e.g. not a global object, not shared between processes or threads).

资料来源：

[http://docs.sqlalchemy.org/en/rel_1_0/faq/connections.html#how-do-i-use-engines-connections-sessions-with-python-multiprocessing-or-os-fork](http://docs.sqlalchemy.org/en/rel_1_0/faq/connections.html#how-do-i-use-engines-connections-sessions-with-python-multiprocessing-or-os-fork)