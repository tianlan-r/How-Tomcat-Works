# 获取参数

在调用avax.servlet.http.HttpServletRequest的getParameter，getParameterMap，getParameterNames，getParameterValues的这些方法来读取参数之前，我们还没有解析query string或者请求体中的参数，所以，在HttpRequest的这四个方法中，都会首先调用parseParameter()方法先解析参数。

参数只需要解析一次，而且只会解析一次，这是因为如果请求体里包含参数，解析参数时SocketInputStream就会读完整个字节流。HttpRequest用一个布尔值变量parsed来标记是否解析过。

参数可以包含在query string或者请求体中，如果是GET请求，所有的参数都在query string中，如果是POST请求，请求体中也会有参数。所有的键值对都会放到一个HashMap中，servlet程序员可以获得这些参数的Map（通过调用HttpServletRequest的getParamterMap()方法），但有一点需要注意，程序员不能修改这些参数的值，所以这里用了一个特殊的HashMap：org.apache.catalina.util.ParameterMap。

ParameterMap继承自HashMap，它有一个布尔值变量locked，只有在locked为false，才可以添加、修改或者删除键值对，否则会抛出一个IllegalStateException异常，读取的话则不受限制。下面会给出ParameterMap的源码，它覆写了add()、update()、remove()方法，这些方法只有在locked为false时才能调用。

```java
package org.apache.catalina.util;
import java.util.HashMap;
import java.util.Map;


public final class ParameterMap extends HashMap {

    public ParameterMap() {
        super();
    }

    public ParameterMap(int initialCapacity) {
        super(initialCapacity);
    }

    public ParameterMap(int initialCapacity, float loadFactor) {
        super(initialCapacity, loadFactor);
    }

    public ParameterMap(Map map) {
        super(map);
    }

    private boolean locked = false;

    public boolean isLocked() {
        return (this.locked);
    }

    public void setLocked(boolean locked) {
        this.locked = locked;
    }

    private static final StringManager sm =
        StringManager.getManager("org.apache.catalina.util");

    public void clear() {
        if (locked)
            throw new IllegalStateException
                (sm.getString("parameterMap.locked"));
        super.clear();
    }

    public Object put(Object key, Object value) {
        if (locked)
            throw new IllegalStateException
                (sm.getString("parameterMap.locked"));
        return (super.put(key, value));
    }

    public void putAll(Map map) {
        if (locked)
            throw new IllegalStateException
                (sm.getString("parameterMap.locked"));
        super.putAll(map);
    }

    public Object remove(Object key) {
        if (locked)
            throw new IllegalStateException
                (sm.getString("parameterMap.locked"));
        return (super.remove(key));
    }
}
```

现在让我们来看看parseParameters()方法是怎样工作的。

因为参数可以包含在query string或者请求体中，所以parseParameters()方法会对两者都进行检查。一旦解析过，参数就会存储在变量parameters中，方法会先判断parsed变量的值，如果parsed为true则表示已经解析过了。

```java
if (parsed)
    return;
```

然后，声明一个ParameterMap变量results，它指向parameters变量。如果parameters为null，则创建一个ParameterMap对象。

```java
ParameterMap results = parameters;
if (results == null)
    results = new ParameterMap();
```

打开ParameterMap的锁，使得能够往里写入数据：

```java
results.setLocked(false);
```

检查编码，如果编码为null则设置默认编码：

```java
String encoding = getCharacterEncoding();
if (encoding == null)
    encoding = "ISO-8859-1";
```

调用org.apache.Catalina.util.RequestUtil的parseParameters()方法对query string进行解析：

```java
// Parse any parameters specified in the query string
String queryString = getQueryString();
try {
    RequestUtil.parseParameters(results, queryString, encoding);
}
catch (UnsupportedEncodingException e) {
    ;
}
```

检查请求体是否包含参数，如果用户发送的是POST请求，content的长度大于0，类型为application/x-www-form-urlencoded，则请求体中包含参数。

```java
// Parse any parameters specified in the input stream
String contentType = getContentType();
if (contentType == null)
    contentType = "";
int semicolon = contentType.indexOf(';');
if (semicolon >= 0) {
    contentType = contentType.substring(0, semicolon).trim();
}
else {
    contentType = contentType.trim();
}
if ("POST".equals(getMethod()) && (getContentLength() > 0) && "application/x-www-form-urlencoded".equals(contentType)) {
    try {
        int max = getContentLength();
        int len = 0;
        byte buf[] = new byte[getContentLength()];
        ServletInputStream is = getInputStream();
        while (len < max) {
            int next = is.read(buf, len, max - len);
            if (next < 0 ) {
                break;
            }
            len += next;
        }
        is.close();
        if (len < max) {
            throw new RuntimeException("Content length mismatch");
        }
        RequestUtil.parseParameters(results, buf, encoding);
    }
    catch (UnsupportedEncodingException ue) {
        ;
    }
    catch (IOException e) {
        throw new RuntimeException("Content read fail");
    }
}
```

最后，ParameterMap上锁，parsed设为true，解析结果赋值给parameters变量。

```java
// Store the final results
results.setLocked(true);
parsed = true;
parameters = results;
```

