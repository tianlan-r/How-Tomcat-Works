# 第9章 Session管理

Catalina通过一个叫做manager的组件来支持session管理，org.apache.catalina.Manager接口表示该组件。一个manager总是会与一个context关联，它负责创建、更新和销毁session对象，以及返回一个有效的session对象给请求的组件。

servlet可以通过调用javax.servlet.http.HttpServletRequest接口的getSession方法获得session对象，在默认的连接器中，这个方法是由org.apache.catalina.connector.HttpRequestBase实现的，下面是HttpRequestBase类中一些相关的方法：

```java
    public HttpSession getSession() {
        return (getSession(true));
    }

    public HttpSession getSession(boolean create) {
        if( System.getSecurityManager() != null ) {
            PrivilegedGetSession dp = new PrivilegedGetSession(create);
            return (HttpSession)AccessController.doPrivileged(dp);
        }
        return doGetSession(create);
    }

    private HttpSession doGetSession(boolean create) {
        // There cannot be a session if no context has been assigned yet
        if (context == null)
            return (null);

        // Return the current session if it exists and is valid
        if ((session != null) && !session.isValid())
            session = null;
        if (session != null)
            return (session.getSession());


        // Return the requested session if it exists and is valid
        Manager manager = null;
        if (context != null)
            manager = context.getManager();

        if (manager == null)
            return (null);      // Sessions are not supported

        if (requestedSessionId != null) {
            try {
                session = manager.findSession(requestedSessionId);
            } catch (IOException e) {
                session = null;
            }
            if ((session != null) && !session.isValid())
                session = null;
            if (session != null) {
                return (session.getSession());
            }
        }

        // Create a new session if requested and the response is not committed
        if (!create)
            return (null);
        if ((context != null) && (response != null) &&
            context.getCookies() &&
            response.getResponse().isCommitted()) {
            throw new IllegalStateException
              (sm.getString("httpRequestBase.createCommitted"));
        }

        session = manager.createSession();
        if (session != null)
            return (session.getSession());
        else
            return (null);

    }
```

默认情况下，manager会在内存中保存它的session对象，但是，Tomcat也允许manager保存它的session对象到文件中或者数据库中（通过JDBC）。Catalina提供了org.apache.catalina.session包，这个包中包含了和session对象以及session管理相关的类。

本章分三小节来介绍Catalina中的session管理，“Sessions”、“Managers”和“Stores”。最后一小节会介绍一个应用程序，这个程序使用了context和一个关联的Manager。

