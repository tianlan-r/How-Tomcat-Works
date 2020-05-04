# 简单的Container应用程序

本章应用程序的主要目的是展示该怎样使用默认的连接器。它由两个类组成：ex04.pyrmont.core.SimpleContainer和ex04 pyrmont.startup.Bootstrap。SimpleContainer实现了org.apache.catalina.container，它可以关联到连接器上。Bootstrap用来启动应用程序。和第3章相比，这里移除了连接器模块、ServletProcessor和StaticResourceProcessor，所以不能请求静态页面。

SimpleContainer如下：

```java
package ex04.pyrmont.core;

import java.beans.PropertyChangeListener;
import java.net.URL;
import java.net.URLClassLoader;
import java.net.URLStreamHandler;
import java.io.File;
import java.io.IOException;
import javax.naming.directory.DirContext;
import javax.servlet.Servlet;
import javax.servlet.ServletException;
import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpServletResponse;

import org.apache.catalina.Cluster;
import org.apache.catalina.Container;
import org.apache.catalina.ContainerListener;
import org.apache.catalina.Loader;
import org.apache.catalina.Logger;
import org.apache.catalina.Manager;
import org.apache.catalina.Mapper;
import org.apache.catalina.Realm;
import org.apache.catalina.Request;
import org.apache.catalina.Response;

public class SimpleContainer implements Container {

    public static final String WEB_ROOT =System.getProperty("user.dir") + File.separator + "webroot";

    public SimpleContainer() {
    }

    public String getInfo() {
        return null;
    }

    public Loader getLoader() {
        return null;
    }

    public void setLoader(Loader loader) {
    }

    public Logger getLogger() {
        return null;
    }

    public void setLogger(Logger logger) {
    }

    public Manager getManager() {
        return null;
    }

    public void setManager(Manager manager) {
    }

    public Cluster getCluster() {
        return null;
    }

    public void setCluster(Cluster cluster) {
    }

    public String getName() {
        return null;
    }

    public void setName(String name) {
    }

    public Container getParent() {
        return null;
    }

    public void setParent(Container container) {
    }

    public ClassLoader getParentClassLoader() {
        return null;
    }

    public void setParentClassLoader(ClassLoader parent) {
    }

    public Realm getRealm() {
        return null;
    }

    public void setRealm(Realm realm) {
    }

    public DirContext getResources() {
        return null;
    }

    public void setResources(DirContext resources) {
    }

    public void addChild(Container child) {
    }

    public void addContainerListener(ContainerListener listener) {
    }

    public void addMapper(Mapper mapper) {
    }

    public void addPropertyChangeListener(PropertyChangeListener listener) {
    }

    public Container findChild(String name) {
        return null;
    }

    public Container[] findChildren() {
        return null;
    }

    public ContainerListener[] findContainerListeners() {
        return null;
    }

    public Mapper findMapper(String protocol) {
        return null;
    }

    public Mapper[] findMappers() {
        return null;
    }

    public void invoke(Request request, Response response) throws IOException, ServletException {
        String servletName = ( (HttpServletRequest) request).getRequestURI();
        servletName = servletName.substring(servletName.lastIndexOf("/") + 1);
        URLClassLoader loader = null;
        try {
            URL[] urls = new URL[1];
            URLStreamHandler streamHandler = null;
            File classPath = new File(WEB_ROOT);
            String repository = (new URL("file", null, classPath.getCanonicalPath() + File.separator)).toString() ;
            urls[0] = new URL(null, repository, streamHandler);
            loader = new URLClassLoader(urls);
        }
        catch (IOException e) {
            System.out.println(e.toString() );
        }
        Class myClass = null;
        try {
            myClass = loader.loadClass(servletName);
        }
        catch (ClassNotFoundException e) {
            System.out.println(e.toString());
        }

        Servlet servlet = null;

        try {
            servlet = (Servlet) myClass.newInstance();
            servlet.service((HttpServletRequest) request, (HttpServletResponse) response);
        }
        catch (Exception e) {
            System.out.println(e.toString());
        }
        catch (Throwable e) {
            System.out.println(e.toString());
        }
    }

    public Container map(Request request, boolean update) {
        return null;
    }

    public void removeChild(Container child) {
    }

    public void removeContainerListener(ContainerListener listener) {
    }

    public void removeMapper(Mapper mapper) {
    }

    public void removePropertyChangeListener(PropertyChangeListener listener) {
    }
}
```

SimpleContainer中我们只提供了invoke()方法的实现，因为默认的连接器会调用此方法。此方法创建了一个类加载器，加载servlet并调用它的service()方法。这个方法和第3章ServletProcessor里的process()方法很像。

下面是Bootstrap的代码：

```java
/* explains Tomcat's default container */
package ex04.pyrmont.startup;

import ex04.pyrmont.core.SimpleContainer;
import org.apache.catalina.connector.http.HttpConnector;

public final class Bootstrap {
    public static void main(String[] args) {
        HttpConnector connector = new HttpConnector();
        SimpleContainer container = new SimpleContainer();
        connector.setContainer(container);
        try {
            connector.initialize();
            connector.start();

            // make the application wait until we press any key.
            System.in.read();
        }
        catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

main方法创建了一个HttpConnector实例和一个SimpleContainer实例，然后调用connector的setContainer方法将两者关联起来。调用连接器的initialize()方法和start()方法，这会连接器就可以在8080端口接受并处理请求了。

