=====================================
Motor 和 PyMongo 的区别
=====================================

主要的区别
=================

创建一个连接(connection)
---------------------

PyMongo的 :class:`~pymongo.mongo_client.MongoClient` 和
:class:`~pymongo.mongo_replica_set_client.MongoReplicaSetClient` 构造函数阻塞(block)直到它们创建了与MongoDB的连接. 然而
:class:`~motor.MotorClient` 或 :class:`~motor.MotorReplicaSetClient` 是独立创建的. 在接受请求(requests)之前, 必须在Tornado web应用程序的开头调用
:meth:`~motor.MotorClient.open_sync` :

.. code-block:: python

    import motor
    client = motor.MotorClient().open_sync()

一旦应用程序运行后马上使连接异步, 要调用 :meth:`~motor.MotorClient.open`:

.. code-block:: python

    def opened(client, error):
        if error:
            print 'Error connecting!', error
        else:
            # Use the client
            pass

    motor.MotorClient().open(opened)

Callbacks 和 Futures
---------------------

Motor支持几乎所有PyMongo方法, 但需要网络I/O的Motor方法有一个可行的回调(callback)函数. 这个回调函数必须接收两个参数:

.. code-block:: python

    def callback(result, error):
        pass

当操作结束时, Motor的异步方法马上返回一个结果(result)或者错误(error), 并执行回调函数.
例如, :meth:`~pymongo.collection.Collection.find_one` 在PyMongo中是这样使用的:

.. code-block:: python

    db = MongoClient().test
    user = db.users.find_one({'name': 'Jesse'})
    print user

但Motor的 :meth:`~motor.MotorCollection.find_one` 方法是异步的:

.. code-block:: python

    db = MotorClient().open_sync().test

    def got_user(user, error):
        if error:
            print 'error getting user!', error
        else:
            print user

    db.users.find_one({'name': 'Jesse'}, callback=got_user)

callback必须以关键字参数形式传递, 而非位置参数.

查找多个文档, Motor提供 :meth:`~motor.MotorCursor.to_list`:

.. code-block:: python

    def got_users(users, error):
        if error:
            print 'error getting users!', error
        else:
            for user in users:
                print user

    db.users.find().to_list(length=10, callback=got_users)

.. seealso:: MotorCursor's :meth:`~motor.MotorCursor.fetch_next`

If you pass no callback to an asynchronous method, it returns a Future for use
in a `coroutine`_:

.. code-block:: python

    from tornado import gen

    @gen.coroutine
    def f():
        yield motor_db.collection.insert({'name': 'Randall'})
        doc = yield motor_db.collection.find_one()

.. _coroutine: http://tornadoweb.org/en/stable/gen.html

See :ref:`coroutine-example`.

max_concurrent and max_wait_time
--------------------------------

PyMongo allows the number of connections to MongoDB to grow to match the number
of threads performing concurrent operations. (PyMongo's ``max_pool_size``
merely caps the number of *idle* sockets kept open. [#max_pool_size]_)
:class:`~motor.MotorClient` and :class:`~motor.MotorReplicaSetClient` provide
an additional option, ``max_concurrent``, which caps the total number of
sockets per host, per client. The default is 100. Once the cap is reached,
operations yield to the IOLoop while waiting for a free socket. The optional
``max_wait_time`` allows operations to raise a :exc:`~motor.MotorPoolTimeout`
if they can't acquire a socket before the deadline.

Timeouts
--------

In PyMongo, you can set a network timeout which causes an
:exc:`~pymongo.errors.AutoReconnect` exception if an operation does not complete
in time::

    db = MongoClient(socketTimeoutMS=500).test
    try:
        user = db.users.find_one({'name': 'Jesse'})
        print user
    except AutoReconnect:
        print 'timed out'

:class:`~motor.MotorClient` and :class:`~motor.MotorReplicaSetClient`
support the same options::

    db = MotorClient(socketTimeoutMS=500).open_sync().test

    @gen.coroutine
    def f():
        try:
            user = yield db.users.find_one({'name': 'Jesse'})
            print user
        except AutoReconnect:
            print 'timed out'

As in PyMongo, the default ``connectTimeoutMS`` is 20 seconds, and the default
``socketTimeoutMS`` is no timeout.

Requests
--------

PyMongo provides "requests" to ensure that a series of operations are performed
in order by the MongoDB server, even with unacknowledged writes (writes with
``w=0``). Motor does not support requests, so the only way to guarantee order
is by doing acknowledged writes. Register a callback for each operation and
perform the next operation in the callback::

    def inserted(result, error):
        if error:
            raise error

        db.users.find_one({'name': 'Ben'}, callback=found_one)

    def found_one(result, error):
        if error:
            raise error

        print result

    # Acknowledged insert:
    db.users.insert({'name': 'Ben', 'maintains': 'Tornado'}, callback=inserted)

This ensures ``find_one`` isn't run until ``insert`` has been acknowledged by
the server. Obviously, this code is improved by `tornado.gen`_::

    @gen.coroutine
    def f():
        yield db.users.insert({'name': 'Ben', 'maintains': 'Tornado'})
        result = yield db.users.find_one({'name': 'Ben'})
        print result

Motor ignores the ``auto_start_request`` parameter to
:class:`~motor.MotorClient` or :class:`~motor.MotorReplicaSetClient`.

.. _tornado.gen: http://tornadoweb.org/en/stable/gen.html

Threading and forking
---------------------

Multithreading and forking are not supported; Motor is intended to be used in
a single-threaded Tornado application. See Tornado's documentation on
`running Tornado in production`_ to take advantage of multiple cores.

.. _`running Tornado in production`: http://tornadoweb.org/en/stable/overview.html#running-tornado-in-production

Minor differences
=================

Deprecated classes and options
------------------------------

PyMongo deprecated the ``slave_okay`` / ``slaveok`` option in favor of
`read preferences`_ in version 2.3. It deprecated
:class:`~pymongo.connection.Connection` and
:class:`~pymongo.replica_set_connection.ReplicaSetConnection` in favor of
:class:`~pymongo.mongo_client.MongoClient` and
:class:`~pymongo.mongo_replica_set_client.MongoReplicaSetClient` in version
2.4, as well as deprecating the ``safe`` option in favor of `write concerns`_.
Motor supports none of PyMongo's deprecated options and classes at all, and
will raise :exc:`~pymongo.errors.ConfigurationError` if you use them.

.. _read preferences: http://api.mongodb.org/python/current/examples/high_availability.html#secondary-reads

.. _write concerns: http://api.mongodb.org/python/current/api/pymongo/mongo_client.html#pymongo.mongo_client.MongoClient.write_concern

MasterSlaveConnection
---------------------

PyMongo's :class:`~pymongo.master_slave_connection.MasterSlaveConnection`
offers a few conveniences when connected to a MongoDB `master-slave pair`_.
Master-slave replication has long been superseded by `replica sets`_, so Motor
has no equivalent to MasterSlaveConnection.

.. _master-slave pair: http://docs.mongodb.org/manual/administration/master-slave/

.. _replica sets: http://docs.mongodb.org/manual/core/replication/

.. _gridfs-differences:

GridFS
------

- File-like

    PyMongo's :class:`~gridfs.grid_file.GridIn` and
    :class:`~gridfs.grid_file.GridOut` strive to act like Python's built-in
    file objects, so they can be passed to many functions that expect files.
    But the I/O methods of :class:`~motor.MotorGridIn` and
    :class:`~motor.MotorGridOut` are asynchronous, so they cannot obey the
    file API and aren't suitable in the same circumstances as files.

- Iteration

    It's convenient in PyMongo to iterate a :class:`~gridfs.grid_file.GridOut`::

        fs = gridfs.GridFS(db)
        grid_out = fs.get(file_id)
        for chunk in grid_out:
            print chunk

    :class:`~motor.MotorGridOut` cannot support this API asynchronously.
    To read a ``MotorGridOut`` use the non-blocking
    :meth:`~motor.MotorGridOut.read` method. For convenience ``MotorGridOut``
    provides :meth:`~motor.MotorGridOut.stream_to_handler`.

    .. seealso:: :ref:`reading-from-gridfs` and :doc:`../api/web`

- Setting properties

    In PyMongo, you can set arbitrary attributes on
    a :class:`~gridfs.grid_file.GridIn` and they're stored as metadata on
    the server, even after the ``GridIn`` is closed::

        grid_in = fs.new_file()
        grid_in.close()
        grid_in.my_field = 'my_value'  # Sends update to server.

    Updating metadata on a :class:`~motor.MotorGridIn` is asynchronous, so
    the API is different::

        @gen.coroutine
        def f():
            fs = motor.MotorGridFS(db)
            yield fs.open()
            grid_in = yield fs.new_file()
            yield grid_in.close()

            # Sends update to server.
            yield grid_in.set('my_field', 'my_value')

    .. seealso:: :ref:`setting-attributes-on-a-motor-gridin`

- The "with" statement

    :class:`~gridfs.grid_file.GridIn` is a context manager--you can use it in a
    "with" statement and it is closed on exit::

        with fs.new_file() as grid_in:
            grid_in.write('data')

    But ``MotorGridIn``'s :meth:`~motor.MotorGridIn.close` method is
    asynchronous, so it must be called explicitly.

is_locked
---------

In PyMongo ``is_locked`` is a property of
:class:`~pymongo.mongo_client.MongoClient`. Since determining whether the
server has been fsyncLocked requires I/O, Motor has no such convenience method.
The equivalent in Motor is::

    result = yield client.admin.current_op()
    locked = bool(result.get('fsyncLock', None))

system_js
---------

PyMongo supports Javascript procedures stored in MongoDB with syntax like:

.. code-block:: python

    >>> db.system_js.my_func = 'function(x) { return x * x; }'
    >>> db.system_js.my_func(2)
    4.0

Motor does not.

Cursor slicing
--------------

In Pymongo, the following raises an ``IndexError`` if the collection has fewer
than 101 documents:

.. code-block:: python

    # Can raise IndexError.
    doc = db.collection.find()[100]

In Motor, however, no exception is raised. The query simply has no results:

.. code-block:: python

    @gen.coroutine
    def f():
        cursor = db.collection.find()[100]

        # Iterates zero or one times.
        while (yield cursor.fetch_next):
            doc = cursor.next_object()

The difference arises because the PyMongo :class:`~pymongo.cursor.Cursor`'s
slicing operator blocks until it has queried the MongoDB server, and determines
if a document exists at the desired offset; Motor simply returns a new
:class:`~motor.MotorCursor` with a skip and limit applied.

.. [#max_pool_size] See `PyMongo's max_pool_size
  <http://api.mongodb.org/python/current/api/pymongo/mongo_client.html#pymongo.mongo_client.MongoClient.max_pool_size>`_
