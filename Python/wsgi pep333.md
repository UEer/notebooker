#wsgi介绍
-----------------------
[参考:pep 333](https://www.python.org/dev/peps/pep-0333/)

wsgi 有两个interface:server/gateway和application/framework。server会调用一个由app提供的可callable 对象。而app怎么提供这个对象给server,就由server决定了。有些server需要app写一个script 来创建一个对象给server,有些server就直接根据config或者其它机制来调用这个对象。

除了纯粹的server和app，wsgi还可以加入midlleware部件。middleware组件对于server就相当于一个app，对于app来说就是个server，可以提供可扩展的api，导航等其它有用的功能。

##app
-----------------------
app 就是一个可调用的接受两个参数的对象。它可以是一个函数，一个类，一个带有__call\__的对象。app可以多次被server调用。
下面两个是app的例子，一个是函数，一个是类。
```python
def simple_app(environ, start_response):
    """Simplest possible application object"""
    status = '200 OK'
    response_headers = [('Content-type', 'text/plain')]
    start_response(status, response_headers)
    return ['Hello world!\n']

class AppClass:
    """Produce the same output, but using a class

    (Note: 'AppClass' is the "application" here, so calling it
    returns an instance of 'AppClass', which is then the iterable
    return value of the "application callable" as required by
    the spec.

    If we wanted to use *instances* of 'AppClass' as application
    objects instead, we would have to implement a '__call__'
    method, which would be invoked to execute the application,
    and we would need to create an instance for use by the
    server or gateway.
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

##server
-----------------------------
server每次从client接受一个http请求就调用一次app。为了说明，下面一个简单的调用一个app对象的cgi gateway。但是下面这个例子对于错误处理是不足的，正常情况错误要输出到server log 上。
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
             # Before the first output, send the stored headers
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
                    # Re-raise original exception if headers sent
                    raise exc_info[0], exc_info[1], exc_info[2]
            finally:
                exc_info = None     # avoid dangling circular ref
        elif headers_set:
            raise AssertionError("Headers already set!")

        headers_set[:] = [status, response_headers]
        return write

    result = application(environ, start_response)
    try:
        for data in result:
            if data:    # don't send headers until body appears
                write(data)
        if not headers_sent:
            write('')   # send headers now if body was empty
    finally:
        if hasattr(result, 'close'):
            result.close()
```

##middleware
--------------
middleware 可以提供如下列几种功能：
- 根据url，把请求转发到不同app处理
- 允许多个app在同一进程使用
- 通过网络中转发请求，实现负载均衡
- 实现内容预处理，例如添加XSL 样式

middleware其实对于server就是个app 对于app就是一个server。middlerware可以写得跟app一样接受两个同样的参数，env和start_response，这样middleware就对于server相当于一个app，只要在middleware像server调用app一样再调用app就可以了，而对于app，middleware就相当于server调用它。

下面是一个用Joe Strout's piglatin.py将text/plain的response转化为pig Latin的middleware
```python
from piglatin import piglatin

class LatinIter:

    """Transform iterated output to piglatin, if it's okay to do so

    Note that the "okayness" can change until the application yields
    its first non-empty string, so 'transform_ok' has to be a mutable
    truth value.
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

    # by default, don't transform output
    transform = False

    def __init__(self, application):
        self.application = application

    def __call__(self, environ, start_response):

        transform_ok = []

        def start_latin(status, response_headers, exc_info=None):

            # Reset ok flag, in case this is a repeat call
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


# Run foo_app under a Latinator's control, using the example CGI gateway
from foo_app import foo_app
run_with_cgi(Latinator(foo_app))
```

##细节说明
------------------------
app必须是接受两个参数，environ和start_response(参数名可以另取，就是一定要代入两个参数)

environ是一个类dict的对象，里面包含cgi-style的环境变量。这个对象必须是使用Python内置的dict对象（不能是子对象或其它），而app能根据自身需求修改这个environ。这个dict也必须包含一些wsgi需要的变量和server需要确认的变量。

start_response是一个可以调用的接受两个必须参数和一个可选参数的对象，这三个参数分别是：status,response_header和exc_info。但是参数不一定要取这些名称。app一定可以用这些参数调用start_response。

satus是一种以"999 Message here"字符串形式给出的。response_header是一个以(header_name, header_value)tuple形式组成的list，用来描述http reaponse header。可选的exc_info是用来处理错误信息的，错误信息通过exc_info展现给浏览器。

start_response一定要返回一个write(body_data)可调用的需要一个参数的对象：一个写进http response body的字符串

当app被server调用的时候，app一定要返回一个可iterable yielding 0或者字符串。这样的app可以通过很多方式去实现，例如返回一个list的字符串或者app写成一个返回字符串的生成器，或者app写成一个可以iterable 的类。

server一定要以unbuffered形式把生成的字符串发送出去后，再去生成另外的。

server一定要把生成的字符串当做是二进制队列。app负责要把所有的字符串转化成对client可读的形式。

如果app返回的iterable有close()方法，那么server在处理完一个请求后一定要调用这个方法，不管这个请求是否正确处理完或者因为错误停止。

最后，server不能直接使用由app提供的iterable中的所有参数，除非参数是指定给server使用的。

##environ variables
---------------------
environ是根据cgi需要的定义的，下面这些参数是一定要提供的，除非他们的值为空。
***
REQUEST_METHOD
***
	The HTTP request method, such as "GET" or "POST" . This cannot ever be an empty string, and so is always required.
***
SCRIPT_NAME
***
	The initial portion of the request URL's "path" that corresponds to the application object, so that the application knows its virtual "location". This may be an empty string, if the application corresponds to the "root" of the server.
***
PATH_INFO
***
	The remainder of the request URL's "path", designating the virtual "location" of the request's target within the application. This may be an empty string, if the request URL targets the application root and does not have a trailing slash.
***
QUERY_STRING
***
	The portion of the request URL that follows the "?" , if any. May be empty or absent.
CONTENT_TYPE
***
	The contents of any Content-Type fields in the HTTP request. May be empty or absent.
***
CONTENT_LENGTH
***
	The contents of any Content-Length fields in the HTTP request. May be empty or absent.
SERVER_NAME , SERVER_PORT
***
	When combined with SCRIPT_NAME and PATH_INFO , these variables can be used to complete the URL. Note, however, that HTTP_HOST , if present, should be used in preference to SERVER_NAME for reconstructing the request URL. See the URL Reconstruction section below for more detail. SERVER_NAME and SERVER_PORT can never be empty strings, and so are always required.
***
SERVER_PROTOCOL
***
	The version of the protocol the client used to send the request. Typically this will be something like "HTTP/1.0" or "HTTP/1.1" and may be used by the application to determine how to treat any HTTP request headers. (This variable should probably be called REQUEST_PROTOCOL , since it denotes the protocol used in the request, and is not necessarily the protocol that will be used in the server's response. However, for compatibility with CGI we have to keep the existing name.)
***
HTTP_ Variables
***
Variables corresponding to the client-supplied HTTP request headers (i.e., variables whose names begin with "HTTP_" ). The presence or absence of these variables should correspond with the presence or absence of the appropriate HTTP header in the request.
***

除了要提供cgi需要的参数，另外还有一些系统的环境变量和一些wsgi需要的参数也要包含在里面。
|Variable|Value|
|--------|-------|
|wsgi.version|wsgi.version	The tuple (1, 0) , representing WSGI version 1.0.|
|wsgi.url_scheme|A string representing the "scheme" portion of the URL at which the application is being invoked. Normally, this will have the value "http" or "https" , as appropriate.|
|wsgi.input|An input stream (file-like object) from which the HTTP request body can be read. (The server or gateway may perform reads on-demand as requested by the application, or it may pre- read the client's request body and buffer it in-memory or on disk, or use any other technique for providing such an input stream, according to its preference.)|
|wsgi.errors|An output stream (file-like object) to which error output can be written, for the purpose of recording program or other errors in a standardized and possibly centralized location. This should be a "text mode" stream; i.e., applications should use "\n" as a line ending, and assume that it will be converted to the correct line ending by the server/gateway.For many servers, wsgi.errors will be the server's main error log. Alternatively, this may be sys.stderr , or a log file of some sort. The server's documentation should include an explanation of how to configure this or where to find the recorded output. A server or gateway may supply different error streams to different applications, if this is desired.|
|wsgi.multithread|This value should evaluate true if the application object may be simultaneously invoked by another thread in the same process, and should evaluate false otherwise.|
|wsgi.multiprocess|This value should evaluate true if an equivalent application object may be simultaneously invoked by another process, and should evaluate false otherwise.|
|wsgi.run_once|This value should evaluate true if the server or gateway expects (but does not guarantee!) that the application will only be invoked this one time during the life of its containing process. Normally, this will only be true for a gateway based on CGI (or something similar).|