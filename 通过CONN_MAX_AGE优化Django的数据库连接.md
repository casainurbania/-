我们用Django+Django-rest-framework提供的一套接口进行了压力测试。压测的过程中，收到DBA通知——数据库连接数过多，希望我们优化下程序。具体症状就是，如果设置mysql的最大连接数为1000，压测过程中，很快连接数就会达到上限，调整上限到2000，依然如此。

# Django的数据库连接

Django对数据库的链接处理是这样的，Django程序接受到请求之后，在第一访问数据库的时候会创建一个数据库连接,直到请求结束，关闭连接。下次请求也是如此。因此，这种情况下，随着访问的并发数越来越高，就会产生大量的数据库连接。也就是我们在压测时出现的情况。

关于Django每次接受到请求和处理完请求时对数据库连接的操作，最后会从源码上来看看。

# 使用CONN_MAX_AGE减少数据库请求

上面说了，每次请求都会创建新的数据库连接，这对于高访问量的应用来说完全是不可接受的。因此在Django1.6时，提供了持久的数据库连接，通过DATABASE配置上添加CONN_MAX_AGE来控制每个连接的最大存活时间。具体使用可以参考最后的链接。

这个参数的原理就是在每次创建完数据库连接之后，把连接放到一个Theard.local的实例中。在request请求开始结束的时候，打算关闭连接时会判断是否超过CONN_MAX_AGE设置这个有效期。这是关闭。每次进行数据库请求的时候其实只是判断local中有没有已存在的连接，有则复用。

基于上述原因，Django中对于CONN_MAX_AGE的使用是有些限制的，使用不当，会事得其反。因为保存的连接是基于线程局部变量的，因此如果你部署方式采用多线程，必须要注意保证你的最大线程数不会多余数据库能支持的最大连接数。另外，如果使用开发模式运行程序（直接runserver的方式），建议不要设置CONN_MAX_AGE，因为这种情况下，每次请求都会创建一个Thread。同时如果你设置了CONN_MAX_AGE，将会导致你创建大量的不可复用的持久的连接。

# CONN_MAX_AGE设置多久

CONN_MAX_AGE的时间怎么设置主要取决于数据库对空闲连接的管理，比如你的MySQL设置了空闲1分钟就关闭连接，那你的CONN_MAX_AGE就不能大于一分钟，不过DBA已经习惯了程序中的线程池的概念，会在数据库中设置一个较大的值。

# 优化结果

了解了上述过程之后，配置了CONN_MAX_AGE参数，再次测试，终于没有接到DBA通知，查看数据库连接数，最大700多。

# 最好的文档是代码

Django的文档上只是简单得介绍了原理和使用方式，对于好奇的同学来说，这个显然是不够的。于是我也好奇的看了下代码，把相关的片段贴到这里。

## 首先是一次请求开始和结束时对连接的处理



#### 请求开始
# django.core.handlers.wsgi.py
class WSGIHandler(base.BaseHandler):
    initLock = Lock()
    request_class = WSGIRequest

    def __call__(self, environ, start_response):
        #  ..... 省略若干代码
        # 触发request_started这个Signal
        signals.request_started.send(sender=self.__class__, environ=environ)
        try:
            request = self.request_class(environ)
        except UnicodeDecodeError:
            logger.warning('Bad Request (UnicodeDecodeError)',
                exc_info=sys.exc_info(),
                extra={
                    'status_code': 400,
                }
            )

> # 请求结束
>

class HttpResponseBase(six.Iterator):
    """
    An HTTP response base class with dictionary-accessed headers.

```python
This class doesn't handle content. It should not be used directly.
Use the HttpResponse and StreamingHttpResponse subclasses instead.
"""

def close(self):
    for closable in self._closable_objects:
        try:
            closable.close()
        except Exception:
            pass
    # 请求结束时触发request_finished这个触发器
    signals.request_finished.send(sender=self._handler_class)
```
这里只是触发，那么在哪对这些signal进行处理呢？

```python
# django.db.__init__.py
from django.db.utils import ConnectionHandler
connections = ConnectionHandler()

# Register an event to reset saved queries when a Django request is started.
def reset_queries(**kwargs):
    for conn in connections.all():
        conn.queries_log.clear()
signals.request_started.connect(reset_queries)

# Register an event to reset transaction state and close connections past
# their lifetime.
def close_old_connections(**kwargs):
    for conn in connections.all():
        conn.close_if_unusable_or_obsolete()
signals.request_started.connect(close_old_connections)
signals.request_finished.connect(close_old_connections)
```

在这里对触发的signal进行了处理，从代码上看，逻辑就是，遍历所有已存在的链接，关闭不可用的连接。

再来看ConnectionHandler代码:

```
class ConnectionHandler(object):

    def __init__(self, databases=None):
        """
        databases is an optional dictionary of database definitions (structured
        like settings.DATABASES).
        """
        # databases来自settings对数据库的配置
        self._databases = databases
        self._connections = local()

    @cached_property
    def databases(self):
        if self._databases is None:
            self._databases = settings.DATABASES
        if self._databases == {}:
            self._databases = {
                DEFAULT_DB_ALIAS: {
                    'ENGINE': 'django.db.backends.dummy',
                },
            }
        if DEFAULT_DB_ALIAS not in self._databases:
            raise ImproperlyConfigured("You must define a '%s' database" % DEFAULT_DB_ALIAS)
        return self._databases

    def __iter__(self):
        return iter(self.databases)

    def all(self):
        # 调用__iter__和__getitem__
        return [self[alias] for alias in self]

    def __getitem__(self, alias):
        if hasattr(self._connections, alias):
            return getattr(self._connections, alias)

        self.ensure_defaults(alias)
        self.prepare_test_settings(alias)
        db = self.databases[alias]
        backend = load_backend(db['ENGINE'])
        # 关键在这了，这个就是conn
        conn = backend.DatabaseWrapper(db, alias)
        # 放到 local里
        setattr(self._connections, alias, conn)
        return conn
```

这个代码的关键就是生成对于backend的conn，并且放到local中。backend.DatabaseWrapper继承了db.backends.__init__.BaseDatabaseWrapper类的 `close_if_unusable_or_obsolete()` 的方法，来直接看下这个方法

```python
class BaseDatabaseWrapper(object):
    """
    Represents a database connection.
    """

    def connect(self):
        """Connects to the database. Assumes that the connection is closed."""
        # 连接数据库时读取配置中的CONN_MAX_AGE
        max_age = self.settings_dict['CONN_MAX_AGE']
        self.close_at = None if max_age is None else time.time() + max_age

    def close_if_unusable_or_obsolete(self):
        """
        Closes the current connection if unrecoverable errors have occurred,
        or if it outlived its maximum age.
        """
        if self.connection is not None:
            # If the application didn't restore the original autocommit setting,
            # don't take chances, drop the connection.
            if self.get_autocommit() != self.settings_dict['AUTOCOMMIT']:
                self.close()
                return

            # If an exception other than DataError or IntegrityError occurred
            # since the last commit / rollback, check if the connection works.
            if self.errors_occurred:
                if self.is_usable():
                    self.errors_occurred = False
                else:
                    self.close()
                    return

            if self.close_at is not None and time.time() >= self.close_at:
                self.close()
                return
```

参考<https://docs.djangoproject.com/en/1.6/ref/databases/#persistent-database-connections>

<https://github.com/django/django/blob/master/django/core/handlers/wsgi.py#L164> <https://github.com/django/django/blob/master/django/http/response.py#L310><https://github.com/django/django/blob/master/django/db/__init__.py#L62> <https://github.com/django/django/blob/master/django/db/utils.py#L252><https://github.com/django/django/blob/master/django/db/backends/__init__.py#L383>