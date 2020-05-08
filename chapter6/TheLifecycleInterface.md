# The Lifecycle Interface

Catalina在设计上允许一个组件包含其他的组件，比如，一个container可以包含一个loader，一个manager等等。父组件负责启动和停止它的子组件。Catalina的设计使得除了一个组件外，所有的组件都被托管在父组件中，这样启动类就只需要启动一个组件就行了。这种单一启动/停止的机制是通过Lifecycle接口来实现的，现在来看看这个接口。

```java
package org.apache.catalina;

public interface Lifecycle {
    public static final String START_EVENT = "start";
    public static final String BEFORE_START_EVENT = "before_start";
    public static final String AFTER_START_EVENT = "after_start";
    public static final String STOP_EVENT = "stop";
    public static final String BEFORE_STOP_EVENT = "before_stop";
    public static final String AFTER_STOP_EVENT = "after_stop";
    
    public void addLifecycleListener(LifecycleListener listener);
    public LifecycleListener[] findLifecycleListeners();
    public void removeLifecycleListener(LifecycleListener listener);
    public void start() throws LifecycleException;
    public void stop() throws LifecycleException;
}
```

这里面最重要的方法是start()和stop()。每个组件都会提供这两个方法的实现，这样它的父组件就可以启动和停止它。其他的三个方法：addLifecycleListener、findLifecycleListeners和removeLifecycleListener，它们是和listener想关联的。组件中可以注册多个监听器来对发生在该组件中的某些事件进行监听。当事件发生时，对这个事件感兴趣的监听器会收到通知。Lifecycle接口在public static final String中定义了6个能够被Lifecycle实例触发的事件的名字。

