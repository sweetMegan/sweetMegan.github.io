---
layout: post
published: true
title: WSGI 读书笔记
description: simple reading note
---  

## WSGI的目标

WSGI 的全称是 Web Server Gateway Interface

WSGI是web server和Python web应用（web applications）或者框架之间的接口。其目的是促进web应用的可移植性。因为web server有很多种，web应用也有很多种，如果没有一个规范，那么我们在写web应用的时候只能针对某一种server来写，程序没有可移植性。如果采用了WSGI规范，那么我们的web应用可以使用任何满足WSGI的server。

现在有一个问题：在WSGI提出以前，很多的server和应用都已经在生产中狂奔了，这些server和application肯定没法遵守WSGI。所以WSGI对这些已经在使用的server和application没有太多好处。为此，WSGI必须特别简单而且容易实现，让已经存在的server和应用只需要很少的更改即可满足这些规范。这样别人才有动力遵守WSGI。

## 规范概述

WSGI接口有两端：1)server端或者说“gateway”端 2)应用端或者“框架”端。server端调用由应用端提供的可调用对象（callable object），至于调用的方式取决于server端。这里的callable指的是 **一个函数，方法，类，或者拥有call方法的对象实例**。server和应用在调用callable对象的时候，不能依赖于这个callable对象的类型（不管是函数还是类）。callable对象仅仅能够被调用，不能自省（introspect）

除了纯粹的server/gateways和applications/frameworks，还有中间件（middleware)存在于两者之间。对于server而言，中间件表现的像是一个应用，而对于应用而言，中间件表现的像是一个server。中间件很是强大，可以提供可扩展的API，内容转换和导航等等功能。

## 应用/框架端

应用的对象是一个接受两个参数的可调用对象。这里的可调用对象就是上面提到的“callable object”。一个函数、方法、类，或者带有*__call__* 方法的对象都可以用来当做应用程序对象。因为server端都会发重复的请求，所以应用的对象也必须能被多次调用，返回的结果也要是**可迭代的**。应用程序对象在这里并不是指应用程序开发者调用WSGI的API开发。 **现在假定应用程序开发者依然使用现有的、高层次的框架（如flask）来开发应用程序。WSGI其实是框架和server端的开发规范，并不直接支持应用开发。**


下面是应用程序对象的两个例子：一个是函数，一个是类：

```python
def simple_app(environ, start_response):
    """ 也许这个是最简单的application object """
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello world!\n']


class AppClass:
    """输出同样的结果，但是使用类

    注意: 'AppClass'类 在这里是一个 "application" ,所以调用它会返回一个AppClass的实例。 
    这个实例会迭代的返回结果。
    如果我们想要使用AppClass的实例作为"application"，那么我们必须要实现__call__方法。
    """

    def __init__(self, environ, start_response):
        self.environ = environ
        self.start = start_response

    def __iter__(self):
        status = '200 OK'
        response_headers = [('Content-type', 'text/plain')]
        self.start(status, response_headers)
        yield "Hello world!\n"
```

应用程序对象必须接受两个位置参数（positional arguments），分别为*environ* 和*start_response*，参数的名字不一定要是这两个单词，但是位置和意义不能变。*environ*参数是一个字典对象，也是一个有着CGI风格的环境变量。应用程序必须允许以任何它需要的方式来修改这个字典， *environ*还必须包含一些特定的WSGI所需的变量。*start_response*参数是一个可被调用的函数，它接受两个必要的位置参数和一个可选参数：status，response_headers和 exc_info 。同样的，名字可以随意取，但是位置和代表的含义不能变。status参数是一个形式如“999 Message here”这样的状态字符串。而response_headers参数是一个包含有（header_name,header_value）参数列表的元组，用来描述HTTP的响应头。可选参数exc_info主要是处理错误时使用。

start_response必须返回一个write(*body_data*)可被调用的对象，个人理解write就是一个类或者函数。write会将数据写入http的响应body里面。但是规范中也提到，新的框架应该避免使用write。

----------

## server端/网关端

server端每次接收HTTP客户端的请求就调用application一次。下面是一个简单的CGI网关，以一个函数的形式来实现。这个例子里涉及到的错误处理很少，因为默认没有被捕捉到的异常都会dump到sys.stderr流中，并且web server 打在log中。

```python
import os, sys

def run_with_cgi(application):

    environ = dict(os.environ.items())
    environ['wsgi.input']        = sys.stdin
    environ['wsgi.errors']       = sys.stderr
    environ['wsgi.version']      = (1, 0)
    environ['wsgi.multithread']  = False
    environ['wsgi.multiprocess'] = True
    environ['wsgi.run_once']     = True

    if environ.get('HTTPS', 'off') in ('on', '1'):
        environ['wsgi.url_scheme'] = 'https'
    else:
        environ['wsgi.url_scheme'] = 'http'

    headers_set = []
    headers_sent = []

    def write(data):
        if not headers_set:
             raise AssertionError("write() before start_response()")

        elif not headers_sent:
             # 在发送数据之前，先把存储的头发送出去
             status, response_headers = headers_sent[:] = headers_set
             sys.stdout.write('Status: %s\r\n' % status)
             for header in response_headers:
                 sys.stdout.write('%s: %s\r\n' % header)
             sys.stdout.write('\r\n')

        sys.stdout.write(data)
        sys.stdout.flush()

    def start_response(status, response_headers, exc_info=None):
        if exc_info:
            try:
                if headers_sent:
                    # 如果报头已经被发送，重新抛出异常
                    raise exc_info[0], exc_info[1], exc_info[2]
            finally:
                exc_info = None     # 避免死循环
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]
        return write

    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # 如果消息主体还没有，不能发送头
                write(data)
        if not headers_sent:
            write('')   # 主体为空，发送头
    finally:
        if hasattr(result, 'close'):
            result.close()
```

服务端在在传输给HTTP客户端数据时，是不用缓冲的。在传输完当前的字符串才会请求下一个。这也就意味着应用必须有自己的缓存。如果application返回的可迭代对象有close()方法，就是上面代码中的result.close()。那么无论请求是否正常结束，最后都要调用这个方法，来释放application的资源。

----------

## 中间件

在服务端，中间件表现的像一个应用。但是在应用端，中间件表现的像一个服务端。中间件有很多作用：
>* 根据不同的URL将请求路由到不同的应用
>* 允许不同的应用或者框架在同一个进程中并行运行
>* 通过在网络中转发请求和应答，实现负载均衡和远程处理
>* 对于内容进行后加工，比如应用xsl样式表等

中间件比较特殊，必须满足server端和应用端的双重的规定。而且中间件可以有很多层，每一层中间件可以提供不同的功能。

下面是一个中间件的例子。这个中间件将英语词尾改成拉丁语式。

```python 
from piglatin import piglatin

class LatinIter:

    """如果可以转换，就把可迭代对象转换为拉丁语。

     Note that the "okayness" can change until the application yields its first non-empty string, so 'transform_ok' has to be a mutable truth value.
    """

    def __init__(self, result, transform_ok):
        if hasattr(result, 'close'):
            self.close = result.close
        self._next = iter(result).next
        self.transform_ok = transform_ok

    def __iter__(self):
        return self

    def next(self):
        if self.transform_ok:
            return piglatin(self._next())
        else:
            return self._next()

class Latinator:

    # 默认情况下不传送输出。
    transform = False

    def __init__(self, application):
        self.application = application

    def __call__(self, environ, start_response):

        transform_ok = []

        def start_latin(status, response_headers, exc_info=None):

            # 重置ok标志位，以防这是一个重复的调用。 
            del transform_ok[:]

            for name, value in response_headers:
                if name.lower() == 'content-type' and value == 'text/plain':
                    transform_ok.append(True)
                    # Strip content-length if present, else it'll be wrong
                    response_headers = [(name, value)
                        for name, value in response_headers
                            if name.lower() != 'content-length'
                    ]
                    break

            write = start_response(status, response_headers, exc_info)

            if transform_ok:
                def write_latin(data):
                    write(piglatin(data))
                return write_latin
            else:
                return write

        return LatinIter(self.application(environ, start_latin), transform_ok)
# 在Latinator's控制下运行foo_app, 使用示例的CGI网关例子。
from foo_app import foo_app
run_with_cgi(Latinator(foo_app))
```

----------

## 其他学习资料

WSGI还有其他的一个规定，比如代码中一直出现的environ变量，还有输入和错误流，编码，url构建的问题等等。这些都比较细碎，不再这里总结了。这里有一些比较好的学习资源：

 1. [pep-0333官方网站](https://www.python.org/dev/peps/pep-0333/)
 2. [中文WSGI翻译版](https://github.com/mainframer/PEP333-zh-CN)
 3. [http://blog.kenshinx.me/blog/wsgi-research/](http://blog.kenshinx.me/blog/wsgi-research/)
 4. [csdn博客](http://blog.csdn.net/on_1y/article/details/18803563)

