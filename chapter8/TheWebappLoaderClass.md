# The WebappLoader Class

org.apache.catalina.loader.WebappLoader是Loader接口的实现，表示一个web应用加载器，负责加载web应用所需的类。WebappLoader创建了一个org.apache.catalina.loader.WebappClassLoader实例作为它的类加载器。和其他Catalina组件一样，WebappLoader实现了Lifecycle接口，关联的容器负责启动或者停止它。WebappLoader还实现了Runnable接口，能够指定一条线程来不断地调用它的类加载器的modified方法，如果modified返回true，就会通知关联的容器（本例中就是context），然后由Context而不是WebappLoader来重新加载类。在第12章会讨论Context是如何做的。

当WebappLoader的start方法被调用，会执行几个重要的任务：

- 创建一个类加载器
- 设置仓库
- 设置类路径
- 设置权限
- 为自动重载启动一个新的线程

在接下来的小节中会讨论每个任务。

## 创建一个类加载器

为了加载类，WebappLoader使用了一个内部的类加载器。Loader接口提供了一个getClassLoader方法但是并没有提供setClassLoader方法，因此不能实例化一个类加载器并把它传给WebappLoader，那WebappLoader就只能使用默认的类加载器了吗？

答案是no。WebappLoader提供了getLoaderClass和setLoaderClass方法来获取或者改变它的私有变量loaderClass的值，这个变量是一个字符串，表示类加载器的名字，它的默认值是org.apache.catalina.loader.WebappClassLoader。如果需要，你可以编写自己的类加载器来继承WebappClassLoader，并调用setLoaderClass方法来强制WebappLoader使用你自定义的类加载器。否则，当启动的时候，WebappLoader会调用它的createClassLoader方法来创建一个WebappClassLoader实例。下面是createClassLoader方法的代码。

```java
private WebappClassLoader createClassLoader() throws Exception {
    Class clazz = Class.forName(loaderClass);
    WebappClassLoader classLoader = null;

    if (parentClassLoader == null) {
        // Will cause a ClassCast is the class does not extend WCL, but
        // this is on purpose (the exception will be caught and rethrown)
        classLoader = (WebappClassLoader) clazz.newInstance();
    } else {
        Class[] argTypes = { ClassLoader.class };
        Object[] args = { parentClassLoader };
        Constructor constr = clazz.getConstructor(argTypes);
        classLoader = (WebappClassLoader) constr.newInstance(args);
    }

    return classLoader;

}
```

能够使用WebappClassLoader以外的类加载器。但是，这个createClassLoader方法返回的是一个WebappClassLoader，所以，如果你自定义的类加载器没有继承WebappClassLoader的话，就会抛出异常。

## 设置仓库

WebappLoader的start方法会调用setRepositories方法来为它的类加载添加仓库。WEB-INF/classes目录会传递给类加载器的addRepository方法，WEB-INF/lib目录传递给类加载器的setJarPath方法。这样，类加载器就能够从WEB-INF/classes目录或者部署到WEB-INF/lib目录的类库中加载类了。

## 设置类路径

start方法调用setClassPath方法来执行此任务，这个方法会将一个包含了Jasper JSP编译器类路径的字符串赋值给servletContext中的属性，这里先不讨论。

## 设置权限

如果Tomcat运行的时候使用了安全管理器，setPermissions方法就会为类加载器添加访问必要目录的权限，比如WEB-INF/classes和WEB-INF/lib。如果没有使用安全管理器，这个方法会立即返回。

## 为自动重载启动一个新的线程

WebappLoader支持自动重载。当WEB-INF/classes或者WEB-INF/lib目录中的类被重新编译了，这些类会自动重新加载，而不必重新启动Tomcat。为了实现这个目的，WebappLoader会有一条线程每隔几秒就检查每个资源的时间戳，间隔时间由变量checkInterval的值决定，默认15，意味着每隔15秒就会执行检查，getCheckInterval和setCheckInterval方法用来访问这个变量。

Tomcat 4里，WebappLoader实现了Runnable接口来支持自动重载。下面是它的run方法。

```java
public void run() {
    if (debug >= 1)
        log("BACKGROUND THREAD Starting");

    // Loop until the termination semaphore is set
    while (!threadDone) {
        // Wait for our check interval
        threadSleep();
        if (!started)
            break;
        try {
            // Perform our modification check
            if (!classLoader.modified())
                continue;
        } catch (Exception e) {
            log(sm.getString("webappLoader.failModifiedCheck"), e);
            continue;
        }
        // Handle a need for reloading
        notifyContext();
        break;
    }
    
    if (debug >= 1)
        log("BACKGROUND THREAD Stopping");
}
```

注：在Tomcat 5中，检查类修改的任务是由org.apache.catalina.core.StandardContext的backgroundProcess方法来完成的，org.apache.catalina.core.ContainerBase的一条独立线程会周期性地调用这个方法，ContainerBase是StandardContext的父类。ContainerBase的内部类ContainerBackgroundProcessor实现了Runnable接口。

run方法里有一个while循环，它会一直循环知道变量started被设置为false(即WebappLoader还未启动或者已经被停止)，while循环里做了几件事：

- 根据变量checkInterval的值周期性地休眠
- 通过调用WebappLoader里的类加载器的modified方法来检查已经加载的类是否有修改，如果没有，继续下一次循环
- 如果有类被修改了，就调用私有方法notifyContext请求与这个WebappLoader关联的Context进行重新加载

notifyContext方法如下：

```java
private void notifyContext() {
    WebappContextNotifier notifier = new WebappContextNotifier();
    (new Thread(notifier)).start();
}
```

notifyContext没有调用Context接口的reload方法，而是实例化了一个内部类WebappContextNotifier，并把它传给一条新的线程，启动新线程。这样，重载的任务就交给了一条新线程来完成。WebappContextNotifier类代码如下：

```java
protected class WebappContextNotifier implements Runnable {
    /**
     * Perform the requested notification.
     */
    public void run() {
        ((Context) container).reload();
    }
}
```

可以看到，run方法调用了Context接口的reload方法，在第12章的org.apache.catalina.core.StandardContext类中会看到reload方法的实现。