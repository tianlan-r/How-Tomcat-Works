# The LifecycleListener Interface

这个接口代表一个生命周期监听器，如下所示：

```java
package org.apache.catalina;

import java.util.EventObject;

public interface LifecycleListener{
    public void lifecycleEvent(LifecycleEvent event);
}
```

这个接口只有一个方法——lifecycleEvent，当这个监听器监听到某一事件发生时，这个方法就会被调用。