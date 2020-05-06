# The Pipeline Interface

Pipeline接口中，我们最先注意到的方法就是invoke()，容器会用这个方法来对管道里的阀和基础阀进行调用。此外，这个接口提供了addValve()方法来添加新阀，removeValve()用来移除阀，setBasic()和getBasic()分别用来设置基础阀和获取基础阀。基础阀会在最后被调用，它负责处理请求和相应的响应。Pipeline接口如下：

```java
package org.apache.catalina;

import java.io.IOException;
import javax.servlet.ServletException;

public interface Pipeline {
    public Valve getBasic();
    public void setBasic(Valve valve);
    public void addValve(Valve valve);
    public Valve[] getValves();
    public void invoke(Request request, Response response) throws IOException, ServletException;
    public void removeValve(Valve valve);
}
```

