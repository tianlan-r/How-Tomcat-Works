# The Logger Interface

日志记录器必须实现org.apache.catalina.Logger接口。

```java
package org.apache.catalina;

import java.beans.PropertyChangeListener;

public interface Logger {
    public static final int FATAL = Integer.MIN_VALUE;
    public static final int ERROR = 1;
    public static final int WARNING = 2;
    public static final int INFORMATION = 3;
    public static final int DEBUG = 4;

    public Container getContainer();
    public void setContainer(Container container);
    public String getInfo();
    public int getVerbosity();
    public void setVerbosity(int verbosity);
    public void addPropertyChangeListener(PropertyChangeListener listener);
    public void log(String message);
    public void log(Exception exception, String msg);
    public void log(String message, Throwable throwable);
    public void log(String message, int verbosity);
    public void log(String message, Throwable throwable, int verbosity);
    public void removePropertyChangeListener(PropertyChangeListener listener);
}
```

Logger接口提供了一些log方法，实现类可以选择调用。这些方法里面最简单的一个是只接收一个字符串并把它作为记录的信息。

另外有两个log方法会接收一个日志级别参数。如果传的数字比实例对象里面设置的级别小，那么这条信息就会被记录，否则这条信息会被忽略。5个日志级别被定义为pulic Static变量：FATAL、ERROR、WARNING、INFORMATION和DEBUG。getVerbosity()和setVerbosity()方法用来获取和设置日志级别。

另外，Logger接口有getContainer()和setContainer()方法用来关联container。它还提供了addPropertyChangeListener()和removePropertyChangeListener()方法来添加或者移除一个PropertyChangeListener。

当你在Tomcat的logger类里面看到这些方法的实现，你就会更了解它们了。

