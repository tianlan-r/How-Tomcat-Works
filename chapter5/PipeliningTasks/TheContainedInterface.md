# The Contained Interface

阀可以选择是否实现该接口，该接口的实现类可以与至多一个容器关联。接口如下：

```java
package org.apache.catalina;

public interface Contained {
    public Container getContainer();
    public void setContainer(Container container);
}
```

