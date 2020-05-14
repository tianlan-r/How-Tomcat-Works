# 应用程序

本章的应用程序示范了如何使用WebappLoader。org.apache.catalina.core.StandardContext是Context的标准实现，该应用程序就实例化了一个StandardContext，但是StandardContext的讨论会留到第12章，这里你不必详细理解这个类，你需要知道的就是它会和一个监听器协同工作监听事件的触发，例如START_EVENT和STOP_EVENT。这个监听器必须实现org.apache.catalina.lifecycle.LifecycleListener接口并调用StandardContextde的setConfigured方法。在本例中，ex08.pyrmont.core.SimpleContextConfig代表了这个监听器，下面是它的代码：

```java
package ex08.pyrmont.core;

import org.apache.catalina.Context;
import org.apache.catalina.Lifecycle;
import org.apache.catalina.LifecycleEvent;
import org.apache.catalina.LifecycleListener;

public class SimpleContextConfig implements LifecycleListener {

  public void lifecycleEvent(LifecycleEvent event) {
    if (Lifecycle.START_EVENT.equals(event.getType())) {
      Context context = (Context) event.getLifecycle();
      context.setConfigured(true);
    }
  }
}
```

你需要做的就是实例化StandardContext和SimpleContextConfig，并调用org.apache.catalina.Lifecycle接口的addLifecycleListener方法注册后者，这个接口已经在第6章讨论过了。

另外，本例中还使用到了一些前面章节中的类：SimplePipeline、SimpleWrapper和SimpleWrapperValve。

可以使用PrimitiveServlet和ModernServlet来测试这个应用程序，但是这次使用到了StandardContext，这些servlet要保存到应用程序目录的WEB-INF/classes目录下。myApp就是应用程序目录，当第一次部署下载的ZIP文件时应该创建此目录。为了告诉StandardContext到哪查找应用程序目录，声明了一个名为catalina.base的系统属性，它的值设置为user.dir属性的值

```java
System.setProperty("catalina.base", System.getProperty("user.dir"));
```

实际上，这就是Bootstrap类的main方法的第一行。然后，main方法会实例化一个默认的连接器。

```java
Connector connector = new HttpConnector();
```

然后，为两个servlet创建两个wrapper并初始化它们，就像以前的章节中的一样。

```java
Wrapper wrapper1 = new SimpleWrapper();
wrapper1.setName("Primitive");
wrapper1.setServletClass("PrimitiveServlet");
Wrapper wrapper2 = new SimpleWrapper();
wrapper2.setName("Modern");
wrapper2.setServletClass("ModernServlet");
```

创建StandardContext实例，并设置它的path和document base。

```java
    Context context = new StandardContext();
    // StandardContext's start method adds a default mapper
    context.setPath("/myApp");
    context.setDocBase("myApp");
```

这等同于在server.xml文件中配置：

```xml
<Context path="/myApp" docBase="myApp"/> 
```

然后，把两个wrapper添加到context，也会添加映射关系使得context能够定位到wrapper。

```java
    context.addChild(wrapper1);
    context.addChild(wrapper2);

    // context.addServletMapping(pattern, name);
    context.addServletMapping("/Primitive", "Primitive");
    context.addServletMapping("/Modern", "Modern");
```

下一步，初始化一个监听器，并把它注册到context

```java
    LifecycleListener listener = new SimpleContextConfig();
    ((Lifecycle) context).addLifecycleListener(listener);
```

实例化WebappLoader并把它关联到context

```java
    // here is our loader
    Loader loader = new WebappLoader();
    // associate the loader with the Context
    context.setLoader(loader);
```

将context关联到默认的连接器，调用连接器的初始化和start方法，接着调用context的start方法，servlet容器就可以使用了

```java
    connector.setContainer(context);
    try {
      connector.initialize();
      ((Lifecycle) connector).start();
      ((Lifecycle) context).start();
```

下一行简单地显示了资源的docBase和类加载器的所有仓库

```java
      // now we want to know some details about WebappLoader
      WebappClassLoader classLoader = (WebappClassLoader) loader.getClassLoader();
      System.out.println("Resources' docBase: " + ((ProxyDirContext)classLoader.getResources()).getDocBase());
      String[] repositories = classLoader.findRepositories();
      for (int i=0; i<repositories.length; i++) {
        System.out.println("  repository: " + repositories[i]);
      }
```

