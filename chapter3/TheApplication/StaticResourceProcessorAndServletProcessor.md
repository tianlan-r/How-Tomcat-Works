# 静态资源处理器和servlet处理器

ServletProcessor和第2章相似，它们都只有一个方法：process()，但是本章的ServletProcessor的process()方法参数是HttpRequest和HttpResponse，而不是Request和Response。下面是它的方法签名：

```java
public void process(HttpRequest request, HttpResponse response)
```

另外，process()方法使用了外观类HttpRequestFacade和HttpResponseFacade，在调用了servlet的service()方法之后，它还会调用HttpResonse的finishResponse()。

```java
servlet = (Servlet) myClass.newInstance();
HttpRequestFacade requestFacade = new HttpRequestFacade(request);
HttpResponseFacade responseFacade = new HttpResponseFacade(response);
servlet.service(requestFacade, responseFacade);
((HttpResponse) response).finishResponse();
```

StaticResourceProcessor则和第2章中的完全一致。