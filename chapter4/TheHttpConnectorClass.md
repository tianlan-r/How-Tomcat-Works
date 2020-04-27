# The HttpConnector Class

你已经知道了这个类是怎么工作的，因为它的简化版已经在第3章中解析过了。它实现了org.apache.catalina.Connector、java.lang.Runnable和org.apache.catalina.Lifecycle接口。Lifecycle接口用于维护每个实现了这个接口的Catalina组件的生命周期。

Lifecycle会在第6章中解释，现在你只要知道，因为实现了这个接口，当你创建了一个HttpConnector实例后，你都应该调用它的initialize()和start()方法。在这个组件的生命周期内，这两个方法只能调用一次。现在我们会讲一些它和第3章的HttpConnector不同的地方：怎样创建一个服务器套接字，怎样维护一个HttpProcessor池以及怎样处理HTTP请求。

## 创建服务器套接字

HttpConnector的initialize()方法调用了私有的open()方法，open()会返回一个java.net.ServerSocket实例，并把它赋值给serverSocket变量。然而，open()不是通过调用ServerSocket的构造器而是通过一个server socket工厂来获得实例的。如果你想要了解这个工厂的详情，可以读读org.apache.catalina.net包里的ServerSocketFactory接口和DefaultServerSocketFactory类，它们都很好理解。

## 维护HttpProcessor实例

在第3章中，HttpConnector在某一时间只会有一个HttpProcessor实例，因此，它每次只能处理一个请求。而在默认连接器中，有一个HttpProcessor对象池，每一个HttpProcessor都有它自己的线程，因此，它能同时处理多个请求。

HttpConnector维护一个HttpProcessor对象池，避免了每次都要新建HttpProcessor对象。HttpProcessor对象存放在一个名为processors的java.io.Stack中：

```java
private Stack processors = new Stack();
```

HttpProcessor实例的数量由两个变量决定：minProcessors和maxProcessors。默认的minProcessors=5，maxProcessors=20，你可以通过setMinProcessors和setMaxProcessors方法来设置它们的值：

```java
protected int minProcessors = 5;
private int maxProcessors = 20;
```

最开始，HttpConnector会创建minProcessors个HttpProcessor实例，如果请求的数量超过了minProcessors，HttpConnector就会创建更多的HttpProcessor实例，直到它的数量到达maxProcessors。如果已经达到了最大数量，再进来的请求就会被忽略。如果想要持续地创建HttpProcessor实例，可以把maxProcessors设置为负数。另外，curProcessors变量表示当前HttpProcessor实例的数量。

在HttpConnector的start()方法中，创建初始数量的HttpProcessor对象：

```java
// Create the specified minimum number of processors
while (curProcessors < minProcessors) {
    if ((maxProcessors > 0) && (curProcessors >= maxProcessors))
        break;
    HttpProcessor processor = newProcessor();
    recycle(processor);
}
```

newProcessor()方法创建一个HttpProcessor对象并把curProcessors加一。recycle()方法把这个HttpProcessor对象入栈。

每一个HttpProcessor对象都会负责解析请求行和请求头信息并填充一个请求对象，因此，每一个HttpProcessor实例都会与一个请求对象和响应对象关联。HttpProcessor的构造器会调用HttpConnector的createRequest()和createResponse()方法。

## 处理请求

HttpConnector的主要逻辑都在run()方法里，就像第3章中的一样。run()方法里有一个while循环，服务器套接字会一直等待HTTP请求，直到连接被关闭。

```java
while (!stopped) {
    Socket socket = null;
    try {
        socket = serverSocket.accept();
    ...
```

每收到一个请求，就会调用createProcessor()方法获得一个HttpProcessor实例：

```java
HttpProcessor processor = createProcessor();
```

大多数时间里，createProcessor()方法并不会创建一个新的HttpProcessor实例，而是从HttpProcessor池中取一个。如果栈中还有HttpProcessor实例，就弹出一个给它。如果栈已经空了而且还没有超过最大数量，就会新建一个实例。如果栈中为空且当前实例数量已经达到了最大，createProcessor()就会返回null。如果返回null，套接字就会关闭，这个请求就不会被处理。

```java
if (processor == null) {
    try {
        log(sm.getString("httpConnector.noProcessor"));
        socket.close();
    } catch (IOException e) {
        ;
    }
    continue;
}
```

如果createProcessor()不是返回null，就把这个套接字传给这个HttpProcessor的assign()方法：

```java
processor.assign(socket);
```

剩下的就是HttpProcessor的工作了，它会读取这个套接字的输入流，解析请求。有一点需要注意，assign()方法会直接返回，不会等待HttpProcessor完成解析，因此就可以为下一次收到的请求服务。每个HttpProcessor会在自己的线程中完成解析，在下一节中你就会看到。

