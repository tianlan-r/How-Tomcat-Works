# 解析Request Line

HttpProcessor的process方法调用了私有方法parseRequest来解析请求行（即请求的第一行），下面是一个请求行的例子：

```
GET /myApp/ModernServlet?userName=tarzan&password=pwd HTTP/1.1 
```

请求行的第二部分是URI加上query string。本例中，URI为

```
/myApp/ModernServlet
```

后面的就是query string

```
userName=tarzan&password=pwd
```

query string可以包含0个或者多个参数，上面的例子中它就包含2个参数。在servlet/JSP程序中，参数jsessionid用于携带会话标识符，会话标识符通常是嵌入到cookies当中，但是当浏览器禁用了cookie，你就可以把会话标识符嵌入到query string中。

当调用parseRequest方法时，变量request已经指向了新创建的HttpRequest对象，parseRequest方法会解析请求行并把解析后的值关联到这个HttpRequest对象上去。让我们来看看parseRequest方法：

```java
private void parseRequest(SocketInputStream input, OutputStream output)
    throws IOException, ServletException {

    // Parse the incoming request line
    input.readRequestLine(requestLine);
    String method =
      new String(requestLine.method, 0, requestLine.methodEnd);
    String uri = null;
    String protocol = new String(requestLine.protocol, 0, requestLine.protocolEnd);

    // Validate the incoming request line
    if (method.length() < 1) {
      throw new ServletException("Missing HTTP request method");
    }
    else if (requestLine.uriEnd < 1) {
      throw new ServletException("Missing HTTP request URI");
    }
    // Parse any query parameters out of the request URI
    int question = requestLine.indexOf("?");
    if (question >= 0) {
      request.setQueryString(new String(requestLine.uri, question + 1,
        requestLine.uriEnd - question - 1));
      uri = new String(requestLine.uri, 0, question);
    }
    else {
      request.setQueryString(null);
      uri = new String(requestLine.uri, 0, requestLine.uriEnd);
    }


    // Checking for an absolute URI (with the HTTP protocol)
    if (!uri.startsWith("/")) {
      int pos = uri.indexOf("://");
      // Parsing out protocol and host name
      if (pos != -1) {
        pos = uri.indexOf('/', pos + 3);
        if (pos == -1) {
          uri = "";
        }
        else {
          uri = uri.substring(pos);
        }
      }
    }

    // Parse any requested session ID out of the request URI
    String match = ";jsessionid=";
    int semicolon = uri.indexOf(match);
    if (semicolon >= 0) {
      String rest = uri.substring(semicolon + match.length());
      int semicolon2 = rest.indexOf(';');
      if (semicolon2 >= 0) {
        request.setRequestedSessionId(rest.substring(0, semicolon2));
        rest = rest.substring(semicolon2);
      }
      else {
        request.setRequestedSessionId(rest);
        rest = "";
      }
      request.setRequestedSessionURL(true);
      uri = uri.substring(0, semicolon) + rest;
    }
    else {
      request.setRequestedSessionId(null);
      request.setRequestedSessionURL(false);
    }

    // Normalize URI (using String operations at the moment)
    String normalizedUri = normalize(uri);

    // Set the corresponding request properties
    ((HttpRequest) request).setMethod(method);
    request.setProtocol(protocol);
    if (normalizedUri != null) {
      ((HttpRequest) request).setRequestURI(normalizedUri);
    }
    else {
      ((HttpRequest) request).setRequestURI(uri);
    }

    if (normalizedUri == null) {
      throw new ServletException("Invalid URI: " + uri + "'");
    }
  }
```

parseRequest方法首先会调用SocketInputStream的readRequestLine方法

```java
input.readRequestLine(requestLine);
```

requestLine是HttpProcessor里的一个HttpRequestLine实例

```java
private HttpRequestLine requestLine = new HttpRequestLine();
```

调用readRequestLine方法就是告诉SocketInputStream，让它去填充这个HttpRequestLine实例。

然后，parseRequest会获取请求行里的请求方法、URI和协议信息：

```java
String method =
      new String(requestLine.method, 0, requestLine.methodEnd);
String uri = null;
String protocol = new String(requestLine.protocol, 0, requestLine.protocolEnd);
```

然而，URI后面可能会有query string，它们之间以 ”?" 分隔，所以parseRequest方法会先获取query string并调用setQueryString方法将值设置到HttpRequest对象上去

```java
// Parse any query parameters out of the request URI
int question = requestLine.indexOf("?");
if (question >= 0) {
  request.setQueryString(new String(requestLine.uri, question + 1,
    requestLine.uriEnd - question - 1));
  uri = new String(requestLine.uri, 0, question);
}
else {
  request.setQueryString(null);
  uri = new String(requestLine.uri, 0, requestLine.uriEnd);
}
```

虽然大部分URI是指向相对路径的资源，但也有写URI是指向绝对路径，就像这样：

```
http://www.brainysoftware.com/index.html?name=Tarzan 
```

parseRequest方法会进行检查：

```java
// Checking for an absolute URI (with the HTTP protocol)
if (!uri.startsWith("/")) {
  int pos = uri.indexOf("://");
  // Parsing out protocol and host name
  if (pos != -1) {
    pos = uri.indexOf('/', pos + 3);
    if (pos == -1) {
      uri = "";
    }
    else {
      uri = uri.substring(pos);
    }
  }
}
```

接着，query string里可能会包含会话标识符，参数名为jsessionid ，因此，parseRequest也需要检查会话标识符，如果在query string中发现了jsessionid，就会获取它的值并调用setRequestedSessionId将值设置到HttpRequest。

```java
// Parse any requested session ID out of the request URI
    String match = ";jsessionid=";
    int semicolon = uri.indexOf(match);
    if (semicolon >= 0) {
        String rest = uri.substring(semicolon + match.length());
        int semicolon2 = rest.indexOf(';');
        if (semicolon2 >= 0) {
            request.setRequestedSessionId(rest.substring(0, semicolon2));
            rest = rest.substring(semicolon2);
        }
        else {
            request.setRequestedSessionId(rest);
            rest = "";
        }
        request.setRequestedSessionURL(true);
        uri = uri.substring(0, semicolon) + rest;
    }
    else {
        request.setRequestedSessionId(null);
        request.setRequestedSessionURL(false);
    }
```

如果发现了jsessionid，就意味着是query string中携带了会话标识符，而不是在cookies中，因此setRequestedSessionURL为true，否则setRequestedSessionURL为false，setRequestedSessionId为null。

此时，uri的值就不包含jsessionid了。

然后，parseRequest方法将uri的值传给normalize方法进行修正，例如，"\\"会被替换为"/"。若uri本身是正常的或者可以被修正，normalize返回原来的uri或者修正后的uri，如果uri不能被修正，就会被认为是无效的，返回null。如果返回null，parseRequest会在方法的最后抛出异常。

最后，设置HttpRequest对象的其他属性：

```java
// Set the corresponding request properties
    ((HttpRequest) request).setMethod(method);
    request.setProtocol(protocol);
    if (normalizedUri != null) {
      ((HttpRequest) request).setRequestURI(normalizedUri);
    }
    else {
      ((HttpRequest) request).setRequestURI(uri);
    }
```

如果normalize方法返回null，抛出异常：

```java
if (normalizedUri == null) {
      throw new ServletException("Invalid URI: " + uri + "'");
    }
```





