# 整体框架



- Consumer：RPC调用方
- Provider：RPC提供方。对外暴露的是接口，实现类Consumer无法直接调用。
- rpc中间件：Consumer通过rpc中间件调用Provider的接口
- Provider-Common



# Consumer

调用Provider的接口，通过RPC调用Provider。



# Provider

提供调用接口，并实现该方法。



# Provider-Common

Provider对外暴露的接口在统一在此包内创建。Provider是各种接口实现，不对外暴露。Consumer调用Provider的包时，引用Provider-Common 而不是Provider包





# RPC

## 参数

定义了RPC参数信息，请求时携带的是该对象。包括：接口名、方法名、参数类型、参数值信息。

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class Invocation implements Serializable {
    /**
     * 接口名
     */
    private String interfaceName;
    /**
     * 方法名
     */
    private String methodName;
    /**
     * 参数类型
     */
    private Class[] parameterTypes;

    /**
     * 参数值信息
     */
    private Object[] parameters;
    
    /**
     * 版本 别名alias
     */
    private String version;
}
```

## 协议

定义服务器，使用Tomcat，通过Netty处理HTTP请求。创建了Servlet，并进行映射。处理请求时，实现特定的Servlet。HttpServerHandler处理真正的http请求。

```java
        // 读取用户的配置 server.name=netty
        Tomcat tomcat = new Tomcat();

        Server server = tomcat.getServer();
        Service service = server.findService("Tomcat");

        Connector connector = new Connector();
        connector.setPort(port);

        Engine engine = new StandardEngine();
        engine.setDefaultHost(hostname);

        Host host = new StandardHost();
        host.setName(hostname);

        String contextPath = "";
        Context context = new StandardContext();
        context.setPath(contextPath);
        context.addLifecycleListener(new Tomcat.FixContextListener());

        host.addChild(context);
        engine.addChild(host);

        service.setContainer(engine);
        service.addConnector(connector);

        tomcat.addServlet(contextPath, "dispatcher", new DispatcherServlet());
        context.addServletMappingDecoded("/*", "dispatcher");

        try {
            tomcat.start();
            tomcat.getServer().await();
        } catch (LifecycleException e) {
            e.printStackTrace();
        }

```

## 处理器

Httpservlet 是处理器的一种。







## 注册

Consumer通过RPC调用Provider。而RPC并不知道Provider提供了哪些接口，以及具体的实现类。因此，需要Provider向RPC中间件进行注册接口，指定实现类。此时可指定别名或者版本信息。

### 本地注册

注入时，指定接口名称，version版本别名，接口实现类。Servlet实现处理调用时，通过Invocation参数信息确定实现类











### 远程注册