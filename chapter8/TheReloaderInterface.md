# The Reloader Interface

为了支持自动重载，类加载器必须实现org.apache.catalina.loader.Reloader接口。

```java
package org.apache.catalina.loader;

public interface Reloader {
    public void addRepository(String repository);
    public String[] findRepositories();
    public boolean modified();
}
```

这个接口最重要的方法是modified()，如果这个web应用中的servlet及其相关类发生了改变，方法会返回true。addRepository方法添加仓库，findRepositories方法返回仓库的数组。