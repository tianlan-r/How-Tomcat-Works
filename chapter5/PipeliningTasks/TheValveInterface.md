# The Valve Interface

Valve接口表示一个阀，负责处理请求。这个接口有两个方法：invoke和getInfo。invoke方法已经讨论过了，getInfo方法返回这个阀实现的信息。接口如下：

```java
package org.apache.catalina;

import java.io.IOException;
import javax.servlet.ServletException;

public interface Valve {
    public String getInfo();
    public void invoke(Request request, Response response, ValveContext context) throws IOException, ServletException;
```

