# 连接器

HttpConnector创建了一个ServerSocket，等待连接请求。

HttpConnector实现了Runnable接口，当你启动应用程序的时候，就会创建一个HttpConnector实例并另起一条线程执行run方法。

run方法里有一个while循环，这个循环做了几件事：

- 等待HTTP请求
- 对于每个请求创建一个HttpProcessor实例
- 调用HttpProcessor的process方法

这个run方法和第二章HttpServer1中的await方法很类似，但是从ServerSocket的accept方法获得获得一个socket之后，它创建了一个HttpProcessor实例并调用它的process方法，socket会作为process方法的参数。

**注：**HttpConnector还有一个getScheme方法，它会返回scheme（http）。

- HttpProcessor的process方法接收到socket后，会做这么几件事：
- 创建一个HttpRequest对象。
- 创建一个HttpResponse对象。
- 解析请求的第一行和headers并设置到HttpRequest对象上去。
- 将HttpRequest和HttpResponse对象传递给ServletProcessor或者StaticResourceProcessor。和第二章一样，ServletProcessor会调用请求的servlet的service方法，StaticResourceProcessor会返回静态资源。

HttpProcessor的process方法如下：

```java
public void process(Socket socket) {
    SocketInputStream input = null;
    OutputStream output = null;
    try {
      input = new SocketInputStream(socket.getInputStream(), 2048);
      output = socket.getOutputStream();

      // create HttpRequest object and parse
      request = new HttpRequest(input);

      // create HttpResponse object
      response = new HttpResponse(output);
      response.setRequest(request);

      response.setHeader("Server", "Pyrmont Servlet Container");

      parseRequest(input, output);
      parseHeaders(input);

      //check if this is a request for a servlet or a static resource
      //a request for a servlet begins with "/servlet/"
      if (request.getRequestURI().startsWith("/servlet/")) {
        ServletProcessor processor = new ServletProcessor();
        processor.process(request, response);
      }
      else {
        StaticResourceProcessor processor = new StaticResourceProcessor();
        processor.process(request, response);
      }

      // Close the socket
      socket.close();
      // no shutdown for this application
    }
    catch (Exception e) {
      e.printStackTrace();
    }
  }
```

process方法先从socket中获取输入流和输出流，我们这里使用的是SocketInputStream，它继承与InputStream。

```java
SocketInputStream input = null;
    OutputStream output = null;
    try {
      input = new SocketInputStream(socket.getInputStream(), 2048);
      output = socket.getOutputStream();
```

然后，它创建HttpRequest和HttpResponse对象，并把request设置到response中去。

```java
// create HttpRequest object and parse
request = new HttpRequest(input);

// create HttpResponse object
response = new HttpResponse(output);
response.setRequest(request);
```

HttpResponse比第2章中Response复杂一些，其中一点就是你可以通过调用它的setHeader方法将headers发送给客户端。

```java
response.setHeader("Server", "Pyrmont Servlet Container");
```

下一步，process方法会调用HttpProcessor的两个私有方法来解析请求。

```java
parseRequest(input, output);
parseHeaders(input);
```

然后，根据请求的URI将HttpRequest和HttpResponse对象交给ServletProcessor或者StaticResourceProcessor进行处理。

```java
//check if this is a request for a servlet or a static resource 
//a request for a servlet begins with "/servlet/"
if (request.getRequestURI().startsWith("/servlet/")) {
    ServletProcessor processor = new ServletProcessor();
    processor.process(request, response);
}
else {
    StaticResourceProcessor processor = new StaticResourceProcessor();
    processor.process(request, response);
}
```

最后关闭socket

```java
socket.close();
```

HttpProcessor使用org.apache.catalina.util.StringManager来发送错误信息

```java
protected StringManager sm = StringManager.getManager("ex03.pyrmont.connector.http");
```

parseRequst和parseHeaders方法被用来填充HttpRequest，我们会在下一节中讨论他们。



