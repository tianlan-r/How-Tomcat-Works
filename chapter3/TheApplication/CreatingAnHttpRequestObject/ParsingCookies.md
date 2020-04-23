# 解析Cookies

浏览器将Cookies作为一个header发送，这样的header以“cookie"作为名字，值是cookie的键值对。这有一个cookie的示例，它包含两个cookie：userName和password

```
Cookie: userName=budi; password=pwd;
```

cookie的解析是由org.apache.catalina.util.RequestUtil的parseCookieHeader方法完成，该方法接收cookie header，返回一个javax.servlet.http.Cookie数组，数组的长度等于header中cookie键值对的个数。parseCookieHeader方法如下：

public static Cookie[] parseCookieHeader(String header) {

    public static Cookie[] parseCookieHeader(String header) {
    
        if ((header == null) || (header.length() < 1))
            return (new Cookie[0]);
    
        ArrayList cookies = new ArrayList();
        while (header.length() > 0) {
            int semicolon = header.indexOf(';');
            if (semicolon < 0)
                semicolon = header.length();
            if (semicolon == 0)
                break;
            String token = header.substring(0, semicolon);
            if (semicolon < header.length())
                header = header.substring(semicolon + 1);
            else
                header = "";
            try {
                int equals = token.indexOf('=');
                if (equals > 0) {
                    String name = token.substring(0, equals).trim();
                    String value = token.substring(equals+1).trim();
                    cookies.add(new Cookie(name, value));
                }
            } catch (Throwable e) {
                ;
            }
        }
    
        return ((Cookie[]) cookies.toArray(new Cookie[cookies.size()]));
    
    }
HttpProcessor的parseHeader方法中对于cookies的处理部分的代码如下：

```java
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
```

