# 解析Headers

请求头信息HttpHeader类表示，这个类在第4章会详细介绍，目前只需知道就够了：

- 可以通过HttpHeader的无参构造函数创建实例
- 有了HttpHeader实例，就可以把它作为SocketInputStream的readHeader方法的参数。若有请求头信息可以读取，readHeader方法会相应地填充HttpHeader对象，若没有请求头信息可以读取了，HttpHeader的nameEnd和valueEnd字段的值为0
- 要获取请求头的名字和值，可以使用下面的方式： 

```java
String name = new String(header.name, 0, header.nameEnd);
String value = new String(header.value, 0, header.valueEnd);
```

parseHeaders方法里有一个while循环，会不断地从SocketInputStream中读取头信息直到读完。循环从创建一个HttpHeader实例并把它传给SocketInputStream的readHeader方法开始：

```java
HttpHeader header = new HttpHeader();;
// Read the next header
input.readHeader(header);
```

然后就可以根据HttpHeader对象的nameEnd和valueEnd检查输入流中是否还有头信息可以读取：

```java
if (header.nameEnd == 0) {
    if (header.valueEnd == 0) {
        return;
    }
    else {
        throw new ServletException (sm.getString("httpProcessor.parseHeaders.colon"));
    }
}
```

如果还有头信息可以读取，就获取头信息的名字和值：

```java
String name = new String(header.name, 0, header.nameEnd);
String value = new String(header.value, 0, header.valueEnd);
```

一旦得到了头信息的名字和值，就可以调用HttpRequest的addHeader方法把头信息添加到HttpRequest实例的headers中去，headers是一个HashMap。

```java
request.addHeader(name, value);
```

一些头信息还需要设置一些其他属性：

```java
// do something for some headers, ignore others.
if (name.equals("cookie")) {
    Cookie cookies[] = RequestUtil.parseCookieHeader(value);
    for (int i = 0; i < cookies.length; i++) {
        if (cookies[i].getName().equals("jsessionid")) {
            // Override anything requested in the URL
            if (!request.isRequestedSessionIdFromCookie()) {
                // Accept only the first session id cookie
                request.setRequestedSessionId(cookies[i].getValue());
                request.setRequestedSessionCookie(true);
                request.setRequestedSessionURL(false);
            }
        }
        request.addCookie(cookies[i]);
    }
}
else if (name.equals("content-length")) {
    int n = -1;
    try {
        n = Integer.parseInt(value);
    }
    catch (Exception e) {
        throw new ServletException(sm.getString("httpProcessor.parseHeaders.contentLength"));
    }
    request.setContentLength(n);
}
else if (name.equals("content-type")) {
    request.setContentType(value);
}
```



