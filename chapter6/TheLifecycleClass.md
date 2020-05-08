# The LifecycleEvent Class

org.apache.catalina.LifecycleEvent类表示一个生命周期事件，下面是它的代码：

```java
package org.apache.catalina;

import java.util.EventObject;

public final class LifecycleEvent extends EventObject {

    public LifecycleEvent(Lifecycle lifecycle, String type) {
        this(lifecycle, type, null);
    }

    public LifecycleEvent(Lifecycle lifecycle, String type, Object data) {
        super(lifecycle);
        this.lifecycle = lifecycle;
        this.type = type;
        this.data = data;
    }

    private Object data = null;
    private Lifecycle lifecycle = null;
    private String type = null;

    public Object getData() {
        return (this.data);
    }

    public Lifecycle getLifecycle() {
        return (this.lifecycle);
    }

    public String getType() {
        return (this.type);
    }
}
```

