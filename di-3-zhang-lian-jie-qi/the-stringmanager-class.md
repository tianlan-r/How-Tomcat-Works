# The StringManager Class

像Tomcat这样的大型程序需要小心仔细地处理错误信息。这些错误信息对于系统管理员和servlet程序员都是非常有用的。例如，系统管理员可以很容易地根据Tomcat的错误日志定位到异常发生的地方，对于servlet程序员来说，Tomcat抛出的每个ServletException里包含了详细的错误信息，程序员们就可以知道自己的servlet出了什么问题。

Tomcat将错误信息存储在properties文件中，方便编辑。然而，Tomcat里有几百个类，如果将这些类用到的错误信息全都保存在一个properties文件里，那将会是维护的噩梦。为了避免这种情况，Tomcat为每个包分配一个properties文件。例如，org.apache.catalina.connector包里的properties文件包含了这个包里所有类能够抛出的错误信息（<font color="#dd0000">不同包有同样的错误信息==>properties内容重复</font>）。一个properties文件用一个org.apache.catalina.util.StringManager实例来处理。Tomcat运行的时候会有很多个StringManager实例，一个实例会读取一个特定的包的properties文件。由于Tomcat的流行，提供多语言的错误信息是很有意义的。目前，支持三种语言，LocalStrings.properties是英语，LocalStrings_es.properties是西班牙语，LocalStrings_ja.properties是日语。

当一个类需要在properties文件里寻找错误信息的时候，首先应该获得一个StringManager实例。然而，为同一个包中、每一个需要错误信息的类都创建一个StringManager实例是很浪费资源的，因此将StringManager设计为：一个包中的所有类共享一个StringManager实例。这就是单例模式，将构造器私有，使得你不能在这个类之外的地方用new创建实例。你只能调用公共的静态方法getManager来获取实例，方法的参数为包名。将实例放入一个Hashtable，key就是对应的包名。

```java
private static Hashtable managers = new Hashtable(); 
public synchronized static StringManager getManager(String packageName) { 		 
	StringManager mgr = (StringManager)managers.get(packageName);
    if (mgr == null) {
        mgr = new StringManager(packageName);         
        managers.put(packageName, mgr); 
    } 
    return mgr;
}
```

例如，获取ex03.pyrmont.connector.http包的StringManager实例：

```java
StringManager sm = StringManager.getManager("ex03.pyrmont.connector.http");
```

ex03.pyrmont.connector.http中你可以看到三个properties文件：LocalStrings.properties，LocalStrings_es.properties和LocalStrings_ja.properties。StringManager实例会根据服务器语言环境来选择使用哪个文件。打开LocalStrings.properties文件，第一行是：

```java
httpConnector.alreadyInitialized=HTTP connector has already been initialized 
```

调用StringManager的getString方法，传入错误码，返回错误信息。getString的方法签名：

```java
public String getString(String key)
```

传入"httpConnector.alreadyInitialized"，返回"HTTP connector has already been initialized"。

