---
title: Python Uvicorn Asyncio探索
date: 2024-01-09 21:00:00
tags: 
  - python
  - learning
  - uvicorn
  - asyncio
categories:
  - 學習
---

![](images/2024-01-09Python_Uvicorn_Asyncio探索/0_INO7Jg0oBMnOleAX.webp)

# 起因
在純函式咖啡每周三的podcast中講到的web framework benchmarks有著很多web framework的排行榜，那就好奇我大Python哪一個web framework是榜首呢～於是乎ctrl + F 搜尋下去就看到 uvicorn 228名，而我常用的django則是462名，那我就好奇uvicorn究竟是怎麼當python中的榜首的呢？就稍為的去了解了一下～

# Uvicorn
[Uvicorn  The lightning-fast ASGI server.](https://www.uvicorn.org/)


他是一個ASGI的web server，異步編程的server，所以嚴格說起來他並不是web framework，而且很容易迷路，且看到ASGI就會想到常用的WSGI，這兩者的差異就之後再說吧～

而要了解uvicorn呢需要知道的項目有

+ asyncio
+ H11Protocol

預設uvicorn main:app 是使用到H11Protocol，也支援

+ HttpTools
+ WS
+ WebSocket

而實際要起一個hello world的網站只需要簡單的

```py
# main.py

async def app(scope, receive, send):
    assert scope['type'] == 'http'

    await send({
        'type': 'http.response.start',
        'status': 200,
        'headers': [
            [b'content-type', b'text/plain'],
        ],
    })
    await send({
        'type': 'http.response.body',
        'body': b'Hello, world!',
    })
```

之後執行 uvicorn main:app 就可以起一個網站了

![](images/2024-01-09Python_Uvicorn_Asyncio探索/1_6M705790uavMCE1bGNQwcg.webp)


在看看uvicorn之前我們先了解一下asyncio要起server怎麼做！

# Asyncio
[Streams](https://docs.python.org/3/library/asyncio-stream.html)

跟著官方文件的範例來看看～可以稍微理解asyncio的server是怎麼運行的！

```py
# server.py

import asyncio


class EchoServerProtocol(asyncio.Protocol):
    def connection_made(self, transport):
        peername = transport.get_extra_info('peername')
        print('Connection from {}'.format(peername))
        self.transport = transport

    def data_received(self, data):
        message = data.decode()
        print('Data received: {!r}'.format(message))

        print('Send: {!r}'.format(message))
        self.transport.write(data)

        print('Close the client socket')
        self.transport.close()


async def main():
    # Get a reference to the event loop as we plan to use
    # low-level APIs.
    loop = asyncio.get_running_loop()

    server = await loop.create_server(
        lambda: EchoServerProtocol(),
        '127.0.0.1', 8888)

    async with server:
        await server.serve_forever()


asyncio.run(main())
```
```py
# client.py

import asyncio


class EchoClientProtocol(asyncio.Protocol):
    def __init__(self, message, on_con_lost):
        self.message = message
        self.on_con_lost = on_con_lost

    def connection_made(self, transport):
        transport.write(self.message.encode())
        print('Data sent: {!r}'.format(self.message))

    def data_received(self, data):
        print('Data received: {!r}'.format(data.decode()))

    def connection_lost(self, exc):
        print('The server closed the connection')
        self.on_con_lost.set_result(True)


async def main():
    # Get a reference to the event loop as we plan to use
    # low-level APIs.
    loop = asyncio.get_running_loop()

    on_con_lost = loop.create_future()
    message = 'Hello World!'

    transport, protocol = await loop.create_connection(
        lambda: EchoClientProtocol(message, on_con_lost),
        '127.0.0.1', 8888)

    # Wait until the protocol signals that the connection
    # is lost and close the transport.
    try:
        await on_con_lost
    finally:
        transport.close()


asyncio.run(main())
```

分別在不同的terminal執行程式 python XXX.py就能看到他們之間的互動

![](images/2024-01-09Python_Uvicorn_Asyncio探索/1_p33U5AmXiDjEH2Krkiq6WA.webp)

![](images/2024-01-09Python_Uvicorn_Asyncio探索/1_sGPdz9OxyA0oS119NgfeHA.webp)


我們可以看到server和client都是跑在loop上面的，而這個EventLoop會是整個asyncio server中很重要的一個部分！

loop分為 windows_events 和 unix_events，我是在windows上測試的所以所有物件都會由windows_events那邊生成～可以在asyncio \__init__.py中看到

```py
# asyncio.__init__.py

if sys.platform == 'win32':  # pragma: no cover
    from .windows_events import *
    __all__ += windows_events.__all__
else:
    from .unix_events import *  # pragma: no cover
    __all__ += unix_events.__all__
```

而我們剛剛照著官網範例實作的loop會是ProactorEventLoop，可以在windows_events.py中找到該class，那就先來看看 `create_server()`的過程吧！

## create_server()
先找到ProactorEventLoop發現沒有create_server在往繼承上去找BaseProactorEventLoop，也發現沒有在往繼承上去找BaseEventLoop，找到create_server了

```py
async def create_server(
        self, protocol_factory, host=None, port=None,
        *,
        family=socket.AF_UNSPEC,
        flags=socket.AI_PASSIVE,
        sock=None,
        backlog=100,
        ssl=None,
        reuse_address=None,
        reuse_port=None,
        ssl_handshake_timeout=None,
        ssl_shutdown_timeout=None,
        start_serving=True):
    """Create a TCP server.

    The host parameter can be a string, in that case the TCP server is
    bound to host and port.

    The host parameter can also be a sequence of strings and in that case
    the TCP server is bound to all hosts of the sequence. If a host
    appears multiple times (possibly indirectly e.g. when hostnames
    resolve to the same IP address), the server is only bound once to that
    host.

    Return a Server object which can be used to stop the service.

    This method is a coroutine.
    """
    if isinstance(ssl, bool):
        raise TypeError('ssl argument must be an SSLContext or None')

    if ssl_handshake_timeout is not None and ssl is None:
        raise ValueError(
            'ssl_handshake_timeout is only meaningful with ssl')

    if ssl_shutdown_timeout is not None and ssl is None:
        raise ValueError(
            'ssl_shutdown_timeout is only meaningful with ssl')

    if sock is not None:
        _check_ssl_socket(sock)

    if host is not None or port is not None:
        if sock is not None:
            raise ValueError(
                'host/port and sock can not be specified at the same time')

        if reuse_address is None:
            reuse_address = os.name == "posix" and sys.platform != "cygwin"
        sockets = []
        if host == '':
            hosts = [None]
        elif (isinstance(host, str) or
              not isinstance(host, collections.abc.Iterable)):
            hosts = [host]
        else:
            hosts = host

        fs = [self._create_server_getaddrinfo(host, port, family=family,
                                              flags=flags)
              for host in hosts]
        infos = await tasks.gather(*fs)
        infos = set(itertools.chain.from_iterable(infos))

        completed = False
        try:
            for res in infos:
                af, socktype, proto, canonname, sa = res
                try:
                    sock = socket.socket(af, socktype, proto)
                except socket.error:
                    # Assume it's a bad family/type/protocol combination.
                    if self._debug:
                        logger.warning('create_server() failed to create '
                                       'socket.socket(%r, %r, %r)',
                                       af, socktype, proto, exc_info=True)
                    continue
                sockets.append(sock)
                if reuse_address:
                    sock.setsockopt(
                        socket.SOL_SOCKET, socket.SO_REUSEADDR, True)
                if reuse_port:
                    _set_reuseport(sock)
                # Disable IPv4/IPv6 dual stack support (enabled by
                # default on Linux) which makes a single socket
                # listen on both address families.
                if (_HAS_IPv6 and
                        af == socket.AF_INET6 and
                        hasattr(socket, 'IPPROTO_IPV6')):
                    sock.setsockopt(socket.IPPROTO_IPV6,
                                    socket.IPV6_V6ONLY,
                                    True)
                try:
                    sock.bind(sa)
                except OSError as err:
                    raise OSError(err.errno, 'error while attempting '
                                  'to bind on address %r: %s'
                                  % (sa, err.strerror.lower())) from None
            completed = True
        finally:
            if not completed:
                for sock in sockets:
                    sock.close()
    else:
        if sock is None:
            raise ValueError('Neither host/port nor sock were specified')
        if sock.type != socket.SOCK_STREAM:
            raise ValueError(f'A Stream Socket was expected, got {sock!r}')
        sockets = [sock]

    for sock in sockets:
        sock.setblocking(False)

    server = Server(self, sockets, protocol_factory,
                    ssl, backlog, ssl_handshake_timeout,
                    ssl_shutdown_timeout)
    if start_serving:
        server._start_serving()
        # Skip one loop iteration so that all 'loop.add_reader'
        # go through.
        await tasks.sleep(0)

    if self._debug:
        logger.info("%r is serving", server)
    return server
```

當中有很多判斷點我們先略過，看他return的server為何～？

```py
server = Server(self, sockets, protocol_factory,
                      ssl, backlog, ssl_handshake_timeout,
                      ssl_shutdown_timeout)
```

一個Server物件，再回到server.py看拿到server後的動作為

## server_forever()

```py
async def main():
    # Get a reference to the event loop as we plan to use
    # low-level APIs.
    loop = asyncio.get_running_loop()

    server = await loop.create_server(
        lambda: EchoServerProtocol(),
        '127.0.0.1', 8888)

    async with server:
        await server.serve_forever()
```

```py
# base_events.Server
class Server(events.AbstractServer):

# ...

async def serve_forever(self):
    if self._serving_forever_fut is not None:
        raise RuntimeError(
            f'server {self!r} is already being awaited on serve_forever()')
    if self._sockets is None:
        raise RuntimeError(f'server {self!r} is closed')

    self._start_serving()
    self._serving_forever_fut = self._loop.create_future()

    try:
        await self._serving_forever_fut
    except exceptions.CancelledError:
        try:
            self.close()
            await self.wait_closed()
        finally:
            raise
    finally:
        self._serving_forever_fut = None

# ...
```

關注在self._start_serving()上

```py
# base_events.Server
class Server(events.AbstractServer):

# ...

def _start_serving(self):
    if self._serving:
        return
    self._serving = True
    for sock in self._sockets:
        sock.listen(self._backlog)
        self._loop._start_serving(
            self._protocol_factory, sock, self._ssl_context,
            self, self._backlog, self._ssl_handshake_timeout,
            self._ssl_shutdown_timeout)
# ...
```

接著看self._loop._start_serving()為何？別忘記這邊的self._loop唯一開始的ProactorEventLoop，所以要從這邊開始找，沒有在往繼承上面找，最後會在BaseProactorEventLoop中找到 _start_serving()

```py
# proactor_events.py BaseProactorEventLoop

class BaseProactorEventLoop(base_events.BaseEventLoop):

# ...

def _start_serving(self, protocol_factory, sock,
                   sslcontext=None, server=None, backlog=100,
                   ssl_handshake_timeout=None,
                   ssl_shutdown_timeout=None):

    def loop(f=None):
        try:
            if f is not None:
                conn, addr = f.result()
                if self._debug:
                    logger.debug("%r got a new connection from %r: %r",
                                 server, addr, conn)
                protocol = protocol_factory()
                if sslcontext is not None:
                    self._make_ssl_transport(
                        conn, protocol, sslcontext, server_side=True,
                        extra={'peername': addr}, server=server,
                        ssl_handshake_timeout=ssl_handshake_timeout,
                        ssl_shutdown_timeout=ssl_shutdown_timeout)
                else:
                    self._make_socket_transport(
                        conn, protocol,
                        extra={'peername': addr}, server=server)
            if self.is_closed():
                return
            f = self._proactor.accept(sock)
        except OSError as exc:
            if sock.fileno() != -1:
                self.call_exception_handler({
                    'message': 'Accept failed on a socket',
                    'exception': exc,
                    'socket': trsock.TransportSocket(sock),
                })
                sock.close()
            elif self._debug:
                logger.debug("Accept failed on socket %r",
                             sock, exc_info=True)
        except exceptions.CancelledError:
            sock.close()
        else:
            self._accept_futures[sock.fileno()] = f
            f.add_done_callback(loop)

    self.call_soon(loop)
# ...
```

這邊最後的self.call_soon()會把該function物件丟到events.Handle並生成Handle物件，然後加到self._ready中，self._ready為collections.deque()，這邊我還沒有搞清楚什麼時候self._ready會被觸發！server繞了一圈被丟到self._ready中～

## asyncio.run()
接著再切回來server.py中最後一行

```py
# server.py

asyncio.run(main())
```

asyncio啟動式

```py
def run(main, *, debug=None, loop_factory=None):
    """Execute the coroutine and return the result.

    This function runs the passed coroutine, taking care of
    managing the asyncio event loop, finalizing asynchronous
    generators and closing the default executor.

    This function cannot be called when another asyncio event loop is
    running in the same thread.

    If debug is True, the event loop will be run in debug mode.

    This function always creates a new event loop and closes it at the end.
    It should be used as a main entry point for asyncio programs, and should
    ideally only be called once.

    The executor is given a timeout duration of 5 minutes to shutdown.
    If the executor hasn't finished within that duration, a warning is
    emitted and the executor is closed.

    Example:

        async def main():
            await asyncio.sleep(1)
            print('hello')

        asyncio.run(main())
    """
    if events._get_running_loop() is not None:
        # fail fast with short traceback
        raise RuntimeError(
            "asyncio.run() cannot be called from a running event loop")

    with Runner(debug=debug, loop_factory=loop_factory) as runner:
        return runner.run(main)
```

會交由Runner去執行

```py
# runners.py Runner.run

class Runner:

# ...

def run(self, coro, *, context=None):
    """Run a coroutine inside the embedded event loop."""
    if not coroutines.iscoroutine(coro):
        raise ValueError("a coroutine was expected, got {!r}".format(coro))

    if events._get_running_loop() is not None:
        # fail fast with short traceback
        raise RuntimeError(
            "Runner.run() cannot be called from a running event loop")

    self._lazy_init()

    if context is None:
        context = self._context
    task = self._loop.create_task(coro, context=context)

    if (threading.current_thread() is threading.main_thread()
        and signal.getsignal(signal.SIGINT) is signal.default_int_handler
    ):
        sigint_handler = functools.partial(self._on_sigint, main_task=task)
        try:
            signal.signal(signal.SIGINT, sigint_handler)
        except ValueError:
            # `signal.signal` may throw if `threading.main_thread` does
            # not support signals (e.g. embedded interpreter with signals
            # not registered - see gh-91880)
            sigint_handler = None
    else:
        sigint_handler = None

    self._interrupt_count = 0
    try:
        return self._loop.run_until_complete(task)
    except exceptions.CancelledError:
        if self._interrupt_count > 0:
            uncancel = getattr(task, "uncancel", None)
            if uncancel is not None and uncancel() == 0:
                raise KeyboardInterrupt()
        raise  # CancelledError
    finally:
        if (sigint_handler is not None
            and signal.getsignal(signal.SIGINT) is sigint_handler
        ):
            signal.signal(signal.SIGINT, signal.default_int_handler)
# ...
```

把main()丟到task裡面後，交由self._loop.run_until_complete(task)，create_task和run_until_complete都在BaseEventLoop中被定義！

```py
# base_events.py BaseEventLoop

class BaseEventLoop(events.AbstractEventLoop):

# ...

def run_until_complete(self, future):
    """Run until the Future is done.

    If the argument is a coroutine, it is wrapped in a Task.

    WARNING: It would be disastrous to call run_until_complete()
    with the same coroutine twice -- it would wrap it in two
    different Tasks and that can't be good.

    Return the Future's result, or raise its exception.
    """
    self._check_closed()
    self._check_running()

    new_task = not futures.isfuture(future)
    future = tasks.ensure_future(future, loop=self)
    if new_task:
        # An exception is raised if the future didn't complete, so there
        # is no need to log the "destroy pending task" message
        future._log_destroy_pending = False

    future.add_done_callback(_run_until_complete_cb)
    try:
        self.run_forever()
    except:
        if new_task and future.done() and not future.cancelled():
            # The coroutine raised a BaseException. Consume the exception
            # to not log a warning, the caller doesn't have access to the
            # local task.
            future.exception()
        raise
    finally:
        future.remove_done_callback(_run_until_complete_cb)
    if not future.done():
        raise RuntimeError('Event loop stopped before Future completed.')

    return future.result()
# ...
```

關注self.run_forever()，要再回頭看ProactorEventLoop中有沒有該function，有！

```py
# windowns_events.py ProactorEventLoop

class ProactorEventLoop(proactor_events.BaseProactorEventLoop):
    """Windows version of proactor event loop using IOCP."""

    def __init__(self, proactor=None):
        if proactor is None:
            proactor = IocpProactor()
        super().__init__(proactor)

    def run_forever(self):
        try:
            assert self._self_reading_future is None
            self.call_soon(self._loop_self_reading)
            super().run_forever()
        finally:
            if self._self_reading_future is not None:
                ov = self._self_reading_future._ov
                self._self_reading_future.cancel()
                # self_reading_future was just cancelled so if it hasn't been
                # finished yet, it never will be (it's possible that it has
                # already finished and its callback is waiting in the queue,
                # where it could still happen if the event loop is restarted).
                # Unregister it otherwise IocpProactor.close will wait for it
                # forever
                if ov is not None:
                    self._proactor._unregister(ov)
                self._self_reading_future = None
```

super到BaseEventLoop的run_forever()

```py
# base_events.py BaseEventLoop

class BaseEventLoop(events.AbstractEventLoop):

# ...

def run_forever(self):
    """Run until stop() is called."""
    self._check_closed()
    self._check_running()
    self._set_coroutine_origin_tracking(self._debug)

    old_agen_hooks = sys.get_asyncgen_hooks()
    try:
        self._thread_id = threading.get_ident()
        sys.set_asyncgen_hooks(firstiter=self._asyncgen_firstiter_hook,
                               finalizer=self._asyncgen_finalizer_hook)

        events._set_running_loop(self)
        while True:
            self._run_once()
            if self._stopping:
                break
    finally:
        self._stopping = False
        self._thread_id = None
        events._set_running_loop(None)
        self._set_coroutine_origin_tracking(False)
        sys.set_asyncgen_hooks(*old_agen_hooks)
# ...
```

這邊會設定events loop並一直執行裡面該有的任務，前面不知道的self._ready也會在self._run_once()當中被執行！

到這邊大概能有個對asyncio server的一個概觀了～

最後回頭來看看 uvicorn是怎麼運作的吧！

## uvicorn server
先找到uvicorn的位置吧，找到之後看看main.py

參數有點多就不貼上來了，主要看看Server物件和該物件的run function！

```py
# uvicorn server.py Server

class Server:
    def __init__(self, config: Config) -> None:
        self.config = config
        self.server_state = ServerState()

        self.started = False
        self.should_exit = False
        self.force_exit = False
        self.last_notified = 0.0

    def run(self, sockets: Optional[List[socket.socket]] = None) -> None:
        self.config.setup_event_loop()
        return asyncio.run(self.serve(sockets=sockets))
```

可以看到一開始先準備好event_loop後執行asyncio.run()，看看其中的self.serve()

```py
async def serve(self, sockets: Optional[List[socket.socket]] = None) -> None:
    process_id = os.getpid()

    config = self.config
    if not config.loaded:
        config.load()

    self.lifespan = config.lifespan_class(config)

    self.install_signal_handlers()

    message = "Started server process [%d]"
    color_message = "Started server process [" + click.style("%d", fg="cyan") + "]"
    logger.info(message, process_id, extra={"color_message": color_message})

    await self.startup(sockets=sockets)
    if self.should_exit:
        return
    await self.main_loop()
    await self.shutdown(sockets=sockets)

    message = "Finished server process [%d]"
    color_message = "Finished server process [" + click.style("%d", fg="cyan") + "]"
    logger.info(message, process_id, extra={"color_message": color_message})
```

在往下看self.startup()就能看到剛剛asyncio server.py那些部分啦！

# 感想
uvicorn設定的部分還有很多，還沒有很熟悉就先到這邊了，對asyncio沒有很熟悉來看這些原始碼真的挺容易迷路的，不過稍微找一下還是能摸清楚大概的來龍去脈，也可以大致了解運行流程以及接收request的過程(這邊今天沒說到)，還可以跟django比較看看兩邊最大的差異點，之後再來看看FastAPI的原始碼吧！！
