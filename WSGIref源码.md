# WSGIref源码
### simple_server.py中的示例代码如下:
```
def demo_app(environ,start_response):
    from io import StringIO
    stdout = StringIO()
    print("Hello world!", file=stdout)
    print(file=stdout)
    h = sorted(environ.items())
    for k,v in h:
        print(k,'=',repr(v), file=stdout)
    start_response("200 OK", [('Content-Type','text/plain; charset=utf-8')])
    return [stdout.getvalue().encode("utf-8")]

def make_server(
    host, port, app, server_class=WSGIServer, handler_class=WSGIRequestHandler
):
    """Create a new WSGI server listening on `host` and `port` for `app`"""
    server = server_class((host, port), handler_class)
    server.set_app(app)
    return server

if __name__ == "__name__":
    with make_server('', 8000, demo_app) as httpd:
        sa = httpd.socket.getsockname()
        print("Serving HTTP on", sa[0], "port", sa[1], "...")
        import webbrowser
        webbrowser.open('http://localhost:8000/xyz?abc')
        httpd.handle_request()  # serve one request, then exit
```
基本的调用就这样子，make\_server构造服务，默认使用WSGIServer，请求处理默认使用WSGIRequestHandler，这两个类还会涉及其他的调用，细节会慢慢展开来写  

### 基本的分层结构
![img](https://read-code.oss-cn-beijing.aliyuncs.com/Snip20210207_9.png)  
wsgi层由wsgiref模块实现  
http层由http模块实现  
tcp/udp层由socketserver模块实现

### 整体调用的结构(忽略了一些细节)，如下
![img](https://read-code.oss-cn-beijing.aliyuncs.com/wsgiref.png)
从图中可以看出WSGIServer和WSGIRequestHandler的继承关系，本质是继承于python的socketserver库的BaseServer和BaseRequestHandler


WSGIServer的细节，如下

![img](https://read-code.oss-cn-beijing.aliyuncs.com/Snip20210205_2.png)

WSGIRequestHandler的细节，如下
![img](https://read-code.oss-cn-beijing.aliyuncs.com/20210205150015.png)

### make\_server函数实现

1. 初始化WSGIServer，赋值给server变量。依据继承关系，可以看出WSGIServer的初始化其实是调用的BaseServer的\_\_init\_\_方法。而且BaseServer实现了\_\_enter\_\_和\_\_exit\_\_方法，所以这个server对象可以使用with语句进行调用
2. server调用了set\_app方法，这个方法是将demo\_app设置成server对象的属性(self.application).
3. 返回server.  

### WSGIServer初始化时的server_bind方法
依据继承关系可以看出WSGIServer->HTTPServer->TCPServer都实现了server\_bind方法，查看源码的时候发现，基本上是下边这个样子   

```
class TCPServer(BaseServer):
    def server_bind(self):
        # tcp绑定主机和端口

class HTTPServer(TCPServer):
    def server_bind(self):
        TCPServer.server_bind(self)
        # 定义server_name
        # 定义server_port

class WSGIServer(HTTPServer):
    def server_bind(self):
        HTTPServer.server_bind(self)
        # 设置wsgi环境变量
```
基本上各层干各层的工作，tcp层负责绑定地址和端口号，设置socket相关的选项；http层在tcp层的基础上再设置了server\_name和server\_port; wsgi层实现了wsgi协议相关的参数设定

### handle\_request方法
handle_request方法是由BaseServer实现的，其中主要的调用流程如下： 
![img](https://read-code.oss-cn-beijing.aliyuncs.com/Snip20210207_7.png)  
1. handler\_request中，使用了selector模块来监听文件描述符。这里不原样复制源码了，摘抄了其中的一部分:  

```
# selectors是对python select的封装，用于实现非阻塞的socket
with selectors.PollSelector as selector:
    # 注册一个文件对象，server_instance既可以是一个fd，也可以是一个实现了fileno()方法的对象，
    # server_instance是BaseServer的实例，而BaseServer实现了fileno()方法
    selector.register(server_instance, selectors.EVENT_READ)
    
    while True:
        ready = selector.select(timeout)  #  用于选择满足我们监听的event的文件对象
        if ready:
                return self._handle_request_noblock()
```
2. \_handle\_request\_noblock定义了请求的处理流程，接收请求->验证请求->处理请求
    * get\_request是对socket.accept()的封装
    * verify\_request验证请求，默认返回True
    * process\_request处理请求，主要是结束请求(用结束可能不太合适)->关闭请求
3. process\_request
    * finish\_reuqest是将BaseServer的实例作为参数传递给RequestHandlerClass进行实例化
    * shutdown\_reqeust
4. shutdown\_reqeust
    * close_request对close\_request的封装
5. close\_request 
    * 关闭请求  


### RequestHandlerClass实例化的时候发生了什么
make_server函数中传入了一个默认的handler\_class即WSGIRequestHandler类。这个类会作为BaseServer实例的RequestHandlerClass实例属性。当调用finish\_request的时候，会对WSGIRequestHandler进行实例化，依据继承关系可得，本质上是由socketserver.py中的BaseRequestHandler来实现实例化。

```
class BaseRequestHandler:
    def __init__(self, request, client_address, server):
        self.request = request
        self.client_address = client_address
        self.server = server
        self.setup()
        try:
            self.handle()
        finally:
            self.finish()
            
    def setup(self):
        pass

    def handle(self):
        pass

    def finish(self):
        pass
```
request、client_address本质上是socket.accept()的返回值，server这里就是BaseServer的实例，接下来还执行了setup()和handle()两个方法。  
**其中setup()由StreamRequestHandler实现，这个方法执行后会给BaseRequestHandler实例添加*connnection、wfile、rfile*三个实例属性**  
**handle()由WSGIRequestHandler实现，这个方法中引入了ServerHandler类**    

调用过程如下
![img](https://read-code.oss-cn-beijing.aliyuncs.com/Snip20210207_10.png)

finish\_content()，会将self.result内容输出wfile中，即
