# The WebappClassLoader Class

org.apache.catalina.loader.WebappClassLoader表示一个类加载器，负责加载web应用要用到的类。WebappClassLoader继承于java.net.URLClassLoader，前面的章节中我们用它来加载java类。

WebappClassLoader的设计方案考虑了优化和安全两个方面。例如，它会缓存已经加载过的类以提高性能，它也会缓存没有找到的类的名字，这样，下一次请求加载这个类的时候就可以直接抛出ClassNotFoundException异常，而不必再去查找一次了。WebappClassLoader会在仓库列表以及指定的jar包里面查找类。考虑到安全，WebappClassLoader不允许载入某些类，这些类存放在一个字符串数组triggers里面，当前这个数组里面只有一个元素：

```java
private static final String[] triggers = {
    "javax.servlet.Servlet"                     // Servlet API
};
```

另外，你必须先委派给系统类加载器，才能加载某些包及其子包下面的类，这些包是：

```java
private static final String[] packageTriggers = {
    "javax",                                     // Java extensions
    "org.xml.sax",                               // SAX 1 & 2
    "org.w3c.dom",                               // DOM 1 & 2
    "org.apache.xerces",                         // Xerces 1 & 2
    "org.apache.xalan"                           // Xalan
};
```

下面让我们来看看它是怎样缓存和加载类的。

## 缓存

为了更好的性能，会缓存已经加载过的类，这样下一次需要这个类的时候就可以直接从缓存中取。缓存可以在本地执行，即由WebappClassLoader实例来管理缓存。java.lang.ClassLoader维护了一个Vector，保存着已经加载过的类，防止他们被当做垃圾回收。在本例中，是由父类来管理缓存。

每一个由 WebappClassLoader 加载的类都被视作一个资源，org.apache.catalina.loader.ResourceEntry 表示资源，一个ResourceEntry实例保存了该类的字节数组形式，最后一次修改时间以及Manifest(如果这个资源来源于一个jar文件的话)等等。

```java
package org.apache.catalina.loader;

import java.net.URL;
import java.security.cert.Certificate;
import java.util.jar.Manifest;

public class ResourceEntry {
    public long lastModified = -1;
    
    // Binary content of the resource.
    public byte[] binaryContent = null;

    // Loaded class.
    public Class loadedClass = null;

    // URL source from where the object was loaded.
    public URL source = null;

    // URL of the codebase from where the object was loaded.
    public URL codeBase = null;

    // Manifest (if the resource was loaded from a JAR).
    public Manifest manifest = null;

    // Certificates (if the resource was loaded from a JAR).
    public Certificate[] certificates = null;
}
```

所有缓存的资源都放在一个名为resourceEntries的HashMap中，key是这个资源的名称。所有未找到的资源存放在另一个HashMap中——notFoundResources。

## 加载类

当加载一个类时，WebappClassLoader会遵循一些规则：

- 所有已经加载过的类都会被缓存，所以首先应该检查本地缓存。
- 若本地缓存没有，则检查上一级缓存，即调用java.lang.ClassLoader的findLoadedClass方法。
- 若缓存中还是没有，则使用系统的类加载器，防止web应用程序的类覆盖J2EE的类。
- 如果使用了SecurityManager，则检查这个类是否允许被载入，如果不允许，则抛出ClassNotFoundException异常。
- 如果delegate变量为true，或者要加载的类属于packageTriggers中的包，则使用父类加载器来加载。如果父加载器是null，则使用system类加载器。
- 从当前仓库加载类。
- 如果当前仓库里面没有找到这个类且delegate为false，则使用父加载器，如果父加载器为null，则使用system类加载器。
- 如果还是没有找到这个类，就抛出ClassNotFoundException异常。



