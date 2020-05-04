# 处理请求

现在，我们已经了解了请求和响应对象，也知道了HttpConnector对象是怎么创建它们。现在是处理的最后一步，本节我们聚焦HttpProcessor的process()方法，run()方法在被指派了一个socket后就会调用process()。process()会做这几件事：

- 解析连接
- 解析请求
- 解析头信息

我们会在解释完process()方法后讨论这些步骤。

process()使用一个布尔值变量ok来表示在处理过程中没有错误，用finishResponse来表示是否应该调用Response接口的finishResponse()方法。

```java
boolean ok = true;
boolean finishResponse = true;
```

process()还使用了几个布尔值变量：keepAlive、stopped和http11。keepAlive表示这个连接是持久的，stopped表示HttpProcessor实例已经被连接器停止所以这个process()方法也应该停止，http11表示这个请求是由支持HTTP 1.1的客户端发起的。

和第3章一样，socket的输入流会包裹在一个SocketInputStream实例中。注意，构建SocketInputStream时传入的buffer大小是来自于连接器，而不是HttpProcessor类的局部变量。这是因为默认连接器的用户无法访问HttpProcessor，而将buffer size放在连接器里面，这样任何人都可以通过这个连接器来设置buffer的大小。

```java
SocketInputStream input = null;
OutputStream output = null;
// Construct and initialize the objects we will need
try {
    input = new SocketInputStream(socket.getInputStream(), connector.getBufferSize());
} catch (Exception e) {
    log("process.create", e);
    ok = false;
}
```

然后，会有一个while循环持续读取输入流，直到HttpProcessor被关闭或者抛出异常，或者连接被关闭。

```java
keepAlive = true;
while (!stopped && ok && keepAlive) {
    ...
}
```

在while循环里，会先设置finishResponse为true，获取输出流并对请求和响应对象做一些初始化。

```java
finishResponse = true;
try {
    request.setStream(input);
    request.setResponse(response);
    output = socket.getOutputStream();
    response.setStream(output);
    response.setRequest(request);
    ((HttpServletResponse) response.getResponse()).setHeader("Server", SERVER_INFO);
} catch (Exception e) {
    log("process.create", e);
    ok = false;
}
```

接着，就开始解析请求，会调用parseConnection、parseRequest、parseHeaders方法，这些都会在下一小节中讨论。

```java
// Parse the incoming request
try {
    if (ok) {
        parseConnection(socket);
        parseRequest(input, output);
        if (!request.getRequest().getProtocol().startsWith("HTTP/0"))
            parseHeaders(input);
```

parseConnection方法会获取请求协议，可能是HTTP 0.9、HTTP 1.0或者HTTP 1.1。如果协议是HTTP 1.0，就将keepAlive设置为false，因为HTTP 1.0不支持持久连接。如果在请求中发现100-continue的头信息，parseHeaders会将sendAck设置为true。

如果协议是HTTP 1.1，客户端发送的头信息里有100-continue，就会调用ackRequest方法响应请求。也会检查是否允许分块。

```java
if (http11) {
    // Sending a request acknowledge back to the client if requested.
    ackRequest(output);
    // If the protocol is HTTP/1.1, chunking is allowed.
    if (connector.isChunkingAllowed())
        response.setAllowChunking(true);
}
```

ackRequest()方法会检查sendAck变量的值，如果sendAck为true，它就会发送以下字符串：

```java
HTTP/1.1 100 Continue \r\n\r\n
```

在解析请求的过程中，可能会抛出很多异常，遇到任何异常都会将ok或者finishResponse变量设置为false。解析之后，就把请求和响应对象传给容器的invoke()方法。

```java
// Ask our Container to process this request
try {
    ((HttpServletResponse) response).setHeader("Date", FastHttpDateFormat.getCurrentDate());
    if (ok) {
        connector.getContainer().invoke(request, response);
    }
}
```

之后，如果finishResponse还是为true，就会调用响应对象的finishResponse()方法和请求对象的finishRequest()方法，并且刷新输出流。

```java
// Finish up the handling of the request
if (finishResponse) {
    ...
    response.finishResponse();
    ...
    request.finishRequest();
    ...
    output.flush();
}
```

循环的最后检查响应的Connection头是否被servlet设置为“close”或者协议是否是HTTP 1.0，如果是这种情况，就设置keepAlive为false。并回收请求和响应对象。

```java
// We have to check if the connection closure has been requested
// by the application or the response stream (in case of HTTP/1.0
// and keep-alive).
if ( "close".equals(response.getHeader("Connection")) ) {
    keepAlive = false;
}
// End of request processing
status = Constants.PROCESSOR_IDLE;
// Recycling the request and the response objects
request.recycle();
response.recycle();
```

如果keepAlive为true，在之前的解析和调用容器的invoke()过程中也没有错误，HttpProcessor实例也没有被停止，这个while循环就会重新开始。否则就会调用shutdownInput()方法并关闭这个套接字。

```java
try {
    shutdownInput(input);
    socket.close();
}
...
```

shutdownInput()方法会检查是否还有未读的字节，如果有，就跳过这些字节。

## 解析连接

parseConnection()方法从套接字中获取网络地址并设置到HttpRequestImpl对象上去。它也会检查是否使用了代理并把socket设置到请求对象上。

```java
private void parseConnection(Socket socket) throws IOException, ServletException {
    if (debug >= 2)
        log("  parseConnection: address=" + socket.getInetAddress() + ", port=" + connector.getPort());
    ((HttpRequestImpl) request).setInet(socket.getInetAddress());
    if (proxyPort != 0)
        request.setServerPort(proxyPort);
    else
        request.setServerPort(serverPort);
    request.setSocket(socket);
}
```

## 解析请求

parseRequest()方法是第3章中的完整版，如果你理解了第3章中的内容，你就能理解它是怎样工作的。

## 解析头信息

默认连接器中的parseHeaders()方法使用了org.apache.catalina.connector.http包中的HttpHeader类和DefaultHeaders类。HttpHeader代表一个请求头，它并没有像第3章一样使用字符串，而是使用字符数组来避免了字符串操作的昂贵开销。DefaultHeaders是一个final类，它的字符串数组里面包含了标准的请求头：

```java
static final char[] AUTHORIZATION_NAME = "authorization".toCharArray();
static final char[] ACCEPT_LANGUAGE_NAME = "accept-language".toCharArray();
static final char[] COOKIE_NAME = "cookie".toCharArray();
...
```

parseHeaders()方法里有一个while循环，它会持续读取请求直到没有更多的头信息可读。循环首先调用request对象的allocateHeader()方法来获取一个空的HttpHeader对象，然后把这个对象传给SocketInputStream的readHeader()方法。

```java
HttpHeader header = request.allocateHeader();
// Read the next header
input.readHeader(header);
```

如果所有的头信息都已经被读取了，readHeader()方法会把HttpHeader对象设置为空name，parseHeaders()方法会直接返回。

```java
if (header.nameEnd == 0) {
    if (header.valueEnd == 0) {
        return;
    } else {
        throw new ServletException(sm.getString("httpProcessor.parseHeaders.colon"));
    }
}
```

如果有头信息的name，就必须有value。

```java
String value = new String(header.value, 0, header.valueEnd);
```

接着，像第3章一样，parseHeaders()方法会比较这个头信息的name和DefaultHeaders里的标准name。注意，这是比较两个字符数组，而不是比较两个字符串。

```java
if (header.equals(DefaultHeaders.AUTHORIZATION_NAME)) {
    request.setAuthorization(value);
} else if (header.equals(DefaultHeaders.ACCEPT_LANGUAGE_NAME)) {
    parseAcceptLanguage(value);
} else if (header.equals(DefaultHeaders.COOKIE_NAME)) {
    // parse cookie
} else if (header.equals(DefaultHeaders.CONTENT_LENGTH_NAME)) {
    // get content length
} else if (header.equals(DefaultHeaders.CONTENT_TYPE_NAME)) {
    request.setContentType(value);
} else if (header.equals(DefaultHeaders.HOST_NAME)) {
    // get host name
} else if (header.equals(DefaultHeaders.CONNECTION_NAME)) {
    if (header.valueEquals(DefaultHeaders.CONNECTION_CLOSE_VALUE)) {
        keepAlive = false;
        response.setHeader("Connection", "close");
    }
} else if (header.equals(DefaultHeaders.EXPECT_NAME)) {
    if (header.valueEquals(DefaultHeaders.EXPECT_100_VALUE))
        sendAck = true;
    else
        throw new ServletException(sm.getString("httpProcessor.parseHeaders.unknownExpectation"));
} else if (header.equals(DefaultHeaders.TRANSFER_ENCODING_NAME)) {
    //request.setTransferEncoding(header);
}
request.nextHeader();
```

