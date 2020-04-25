第3章中连接器已经可以很好的工作了，也可以再完善完善实现更多的功能。然而，它终究只是一个学习工具，是为了介绍Tomcat 4的默认连接器而编写的，理解第3章中的连接器是理解Tomcat 4默认连接器的关键。本章会通过剖析Tomcat 4默认连接器的代码来讨论如何构建一个真正的Tomcat连接器。

注：本章中的默认连接器是指Tomcat 4的默认连接器，尽管这个默认连接器已经废弃了，被一个更快的连接器Coyote取代，它仍然是一个好的教材。

Tomcat的连接器是一个独立的模块，这个模块可以插入到servlet容器里。已经存在了很多的连接器，像Coyote、mod_jk、mod_jk2和mod_webapp。一个Tomcat连接器必须满足以下条件：

- 必须实现org.apache.catalina.Connector接口
- 必须创建实现了org.apache.catalina.Request接口的请求对象
- 必须创建实现了org.apache.catalina.Response接口的响应对象

Tomcat 4默认的连接器的工作原理和第3章中的简单连接器类似。它等待HTTP请求，创建请求和响应对象，然后把请求和响应对象传递给容器。连接器通过调用org.apache.catalina.Container接口的invoke()方法将请求和响应对象传递给容器，invoke()的方法签名如下：

```java
public void invoke(Request request, Response response) throws IOException, ServletException;
```

在invoke()方法中，容器会加载servlet，调用它的service()方法，管理会话，记录错误信息日志等。

默认连接器在第3章的连接器的基础上进行了一些优化，第一个就是提供了多种对象池来避免昂贵的对象创建，另一个是大量地使用字符数组来代替字符串。

本章的应用程序时一个简单的容器，它会和这个默认的连接器关联起来。但是本章的重点不是这个简单的容器而是默认的连接器，容器会在第5章讨论。尽管如此，为了展示如何使用默认的连接器，本章的最后一节也会讨论下这个简单的容器。

另外一个需要注意的点是默认连接器不光可以为HTTP 0.9和1.0服务，也实现了1.1的新特性。为了理解这些新特性，你需要好好读一读本章的第一节。在那之后，我们会讨论org.apache.catalina.Connector接口和怎样创建请求和响应对象。如果你理解了第3章中连接器的工作方式，理解这个默认连接器就不会有任何困难了。

本章从HTTP 1.1的3个新特性开始，了解这些特性对于我们理解默认连接器的内部工作原理是很重要的。然后会介绍所有连接器都必须实现的org.apache.catalina.Connector接口。你会看到一些在第3章出现过的类，比如HttpConnector、HttpProcessor等，这次它们会更复杂一些。

