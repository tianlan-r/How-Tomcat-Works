# The LifecycleSupport Class

一个实现了Lifecycle接口并且允许注册监听器的组件必须提供和监听器相关的三个方法的实现代码（addLifecycleListener、findLifecycleListener和removeLifecycleListener），这个组件还必须把这些添加的监听器存到一个数组或者ArrayList或者其他类似的集合里面。Catalina提供了一个通用的类来简化组件对于监听器的处理和生命周期事件的触发，这个类就是org.apache.catalina.util.LifecycleSupport。

```java
package org.apache.catalina.util;

import org.apache.catalina.Lifecycle;
import org.apache.catalina.LifecycleEvent;
import org.apache.catalina.LifecycleListener;

public final class LifecycleSupport {

    public LifecycleSupport(Lifecycle lifecycle) {
        super();
        this.lifecycle = lifecycle;
    }

    private Lifecycle lifecycle = null;
	private LifecycleListener listeners[] = new LifecycleListener[0];

    public void addLifecycleListener(LifecycleListener listener) {
      	synchronized (listeners) {
          	LifecycleListener results[] = new LifecycleListener[listeners.length + 1];
          	for (int i = 0; i < listeners.length; i++)
              	results[i] = listeners[i];
          	results[listeners.length] = listener;
          	listeners = results;
      	}
    }

    public LifecycleListener[] findLifecycleListeners() {
        return listeners;
    }

    public void fireLifecycleEvent(String type, Object data) {
        LifecycleEvent event = new LifecycleEvent(lifecycle, type, data);
        LifecycleListener interested[] = null;
        synchronized (listeners) {
            interested = (LifecycleListener[]) listeners.clone();
        }
        for (int i = 0; i < interested.length; i++)
            interested[i].lifecycleEvent(event);
    }

    public void removeLifecycleListener(LifecycleListener listener) {
        synchronized (listeners) {
            int n = -1;
            for (int i = 0; i < listeners.length; i++) {
                if (listeners[i] == listener) {
                    n = i;
                    break;
                }
            }
            if (n < 0)
                return;
            LifecycleListener results[] =
              new LifecycleListener[listeners.length - 1];
            int j = 0;
            for (int i = 0; i < listeners.length; i++) {
                if (i != n)
                    results[j++] = listeners[i];
            }
            listeners = results;
        }
    }
}
```

如你所见，LifecycleSupport把所有监听器存到一个数组listeners里面，它初始化时没有元素。

```java
private LifecycleListener listeners[] = new LifecycleListener[0];
```

当addLifecycleListener方法添加一个监听器时，创建一个容量等于原来容量加一的新数组，然后把原来数组里面的监听器复制到新数组并把新监听器添加进去。当用removeLifecycleListener方法移除一个监听器时，也会创建一个新的数组，新的容量是原来的容量减一，然后把原来数组里除了要移除的那个监听器之外的监听器全部复制到新数组里。

fireLifecycleEvent方法触发事件。首先，克隆原来的监听器数组，然后调用每一个成员的lifecycleEvent方法，传入触发的事件。

实现了Lifecycle接口的组件都可以使用LifecycleSupport。举个例子，本章应用程序里的SimpleContext类就声明了一个变量：

```java
protected LifecycleSupport lifecycle = new LifecycleSupport(this);
```

添加监听器时，SimpleContext就调用LifecycleSupport的addLifecycleListener方法：

```java
public void addLifecycleListener(LifecycleListener listener){
    lifecycle.addLifecycleListener(listener);
}
```

移除监听器时，SimpleContext调用LifecycleSupport的removeLifecycleListener方法：

```java
public void removeLifecycleListener(LifecycleListener listener){
    lifecycle.removeLifecycleListener(listener);
}
```

触发事件时，SimpleContext调用LifecycleSupport的fireLifecycleEvent方法，就像下面这样：

```java
lifecycle.fireLifecycleEvent(START_EVENT, null);
```

