# 读取socket输入流

在第1章和第2章的HttpRequest中，已经对请求进行了初步的解析。通过调用InputStream的read方法，可以得到请求行信息，包括请求方法、URI和HTTP版本。

```java
byte[] buffer = new byte [2048]; 
try { 
    // input is the InputStream from the socket. 
    i = input.read(buffer); 
}
```

在本章中，有一个ex03.pyrmont.connector.http.SocketInputStream类，它是org.apache.catalina.connector.http.SocketInputStream的拷贝。这个类提供了获取请求头和请求行的方法。

创建一个SocketInputStream实例，需要传入一个inputStream和一个integer，这个integer表示这个SocketInputStream实例的buffer大小。在HttpProcessor的process方法里就创建了一个SocketInputStream对象：

```java
SocketInputStream input = null;
OutputStream output = null;
try {
  input = new SocketInputStream(socket.getInputStream(), 2048);
```

就像前面提到的，创建SocketInputStream对象是因为它有两个重要方法：readRequestLine和readHeader。



