# The ValveContext Interface

这个接口有两个方法：invokeNext方法已经讨论过了，getInfo方法返回这个ValveContext实现的信息。接口如下：

```java
package org.apache.catalina;

import java.io.IOException;
import javax.servlet.ServletException;

public interface ValveContext {
    public String getInfo();
    public void invokeNext(Request request, Response response) throws IOException, ServletException;
```

