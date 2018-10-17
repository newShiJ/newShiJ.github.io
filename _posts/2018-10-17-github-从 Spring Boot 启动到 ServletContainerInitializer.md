---
layout: post
title:  "从 Spring Boot 启动到 ServletContainerInitializer"
date:   2018-10-17
categories: jekyll update
---

## Spring Boot Tomcat 启动配置

> 编写这篇文章的起因是同事问了一个关于Spring Boot项目Tomcat启动配置的问题。  

> 正常平时开发过程中，我们开发spring boot项目一般会使用Spring官方的脚手架搭建Spring Boot项目，启动也是一般使用Spring Boot启动。同样的打包方式为jar包，但是如果想部署到Tomcat中的话就需要对项目进行一个简单改造,首先将pom.xml中的打包方式从 jar 改变到 war。接着在Application同级包下创建一个启动类（代码如下）。

```java
@SpringBootApplication
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class,args);
    }
}
```
```java
public class Bootstrap extends SpringBootServletInitializer {
    @Override
    protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
        return application.sources(Application.class);
    }
}
```
> 这是Spring官方为我们提供的启动改造启动方式。
> 接下来，我们来看看为什么可以这样做就能做到Tomcat启动。

## 凭什么 SpringBootServletInitializer 可以被注入
通过上述代码可知SpringBootServletInitializer主要是将Application的启动参数注入到SpringApplicationBuilder。那么我们将探究为什么可以通过这种方式进行传递参数。

首先阅读源码什么是SpringBootServletInitializer

```java
public abstract class SpringBootServletInitializer implements WebApplicationInitializer {
	...
}
```
可以看出SpringBootServletInitializer是一个抽象类，实现了WebApplicationInitializer接口，接着看看代码上的注释是怎么说的

```java
/**
 * An opinionated {@link WebApplicationInitializer} to run a {@link SpringApplication}
 * from a traditional WAR deployment. Binds {@link Servlet}, {@link Filter} and
 * {@link ServletContextInitializer} beans from the application context to the server.
 * <p>
 * To configure the application either override the
 * {@link #configure(SpringApplicationBuilder)} method (calling
 * {@link SpringApplicationBuilder#sources(Class...)}) or make the initializer itself a
 * {@code @Configuration}. If you are using {@link SpringBootServletInitializer} in
 * combination with other {@link WebApplicationInitializer WebApplicationInitializers} you
 * might also want to add an {@code @Ordered} annotation to configure a specific startup
 * order.
 * <p>
 * Note that a WebApplicationInitializer is only needed if you are building a war file and
 * deploying it. If you prefer to run an embedded web server then you won't need this at
 * all.
 *
 * @author Dave Syer
 * @author Phillip Webb
 * @author Andy Wilkinson
 * @since 2.0.0
 * @see #configure(SpringApplicationBuilder)
 */
 
 / **
 *一个自以为是的{@link WebApplicationInitializer}来运行{@link SpringApplication}
 *来自传统的WAR部署。绑定{@link Servlet}，{@ link Filter}和
 * {@link ServletContextInitializer} bean从应用程序上下文到服务器。
 * <p>
 *要配置应用程序，请覆盖
 * {@link #configure（SpringApplicationBuilder）}方法（调用
 * {@link SpringApplicationBuilder＃sources（Class ...）}）或使初始化器本身成为一个
 * {@code @Configuration}。如果您正在使用{@link SpringBootServletInitializer}
 *与其他{@link WebApplicationInitializer WebApplicationInitializers}组合你
 *可能还想添加{@code @Ordered}注释来配置特定的启动
 *订单。
 * <p>
 *请注意，只有在构建war文件时才需要WebApplicationInitializer
 *部署它。如果您更喜欢运行嵌入式Web服务器，那么您将无需使用此服务器
 *全部。
 *
 * @author Dave Syer
 * @author Phillip Webb
 * @author Andy Wilkinson
 * @since 2.0.0
 * @see #configure（SpringApplicationBuilder）
 * /
```
说白了就是当以war形式部署时，你才需要使用到。如果是以嵌入式启动可以不用关注它。

我们再看看`WebApplicationInitializer`是个什么东西

```java
public interface WebApplicationInitializer {
	void onStartup(ServletContext servletContext) throws ServletException;
}
```

我们再看看`WebApplicationInitializer`在哪里被使用

```java
@HandlesTypes(WebApplicationInitializer.class)
public class SpringServletContainerInitializer implements ServletContainerInitializer {
	@Override
	public void onStartup(@Nullable Set<Class<?>> webAppInitializerClasses, ServletContext servletContext)
			throws ServletException {
	...
}
```
这里实现了一个`ServletContainerInitializer`接口。这个接口是j2ee servlet 3.0以后的规范。关于`ServletContainerInitializer`这里有一些相关的介绍，[Apache](https://tomcat.apache.org/tomcat-8.0-doc/servletapi/javax/servlet/ServletContainerInitializer.html)，[开源中国](https://my.oschina.net/u/3574106/blog/1819394)都有一些相关的介绍。当然最好可以去[JCP官网](https://www.jcp.org/en/home/index)寻找官方文档说明

大致介绍如下
> 公共接口ServletContainerInitializer
ServletContainerInitializers（SCI）通过文件META-INF / services / javax.servlet.ServletContainerInitializer中的条目注册，该文件必须包含在包含SCI实现的JAR文件中。 (Apache介绍翻译)
>
> 说的简单一些就是利用java [SPI](https://juejin.im/post/5af952fdf265da0b9e652de3) 技术将 `ServletContainerInitializer`的实现类写在 META-INF / services / javax.servlet.ServletContainerInitializer 中反射获取 字节码对象（Class）并且将 @HandlesTypes 感兴趣的类型进行实例化注入


## Spring Boot Tomcat 部署启动过程

根据多次debug发现 Tomcat启动过程中会将所有的类文件加载到容器中进行解析,该过程主要代码在`ContextConfig` 即`org.apache.catalina.startup.ContextConfig `中,主体代码如下，实现了`LifecycleListener `接口

```java
public class ContextConfig implements LifecycleListener {
	...
	@Override
    public void lifecycleEvent(LifecycleEvent event) {

        // Identify the context we are associated with
        try {
            context = (Context) event.getLifecycle();
        } catch (ClassCastException e) {
            log.error(sm.getString("contextConfig.cce", event.getLifecycle()), e);
            return;
        }

        // Process the event that has occurred
        if (event.getType().equals(Lifecycle.CONFIGURE_START_EVENT)) {
            configureStart();
        } else if (event.getType().equals(Lifecycle.BEFORE_START_EVENT)) {
            beforeStart();
        } else if (event.getType().equals(Lifecycle.AFTER_START_EVENT)) {
            // Restore docBase for management tools
            if (originalDocBase != null) {
                context.setDocBase(originalDocBase);
            }
        } else if (event.getType().equals(Lifecycle.CONFIGURE_STOP_EVENT)) {
            configureStop();
        } else if (event.getType().equals(Lifecycle.AFTER_INIT_EVENT)) {
            init();
        } else if (event.getType().equals(Lifecycle.AFTER_DESTROY_EVENT)) {
            destroy();
        }

    }
    ...
}
```
这里有一张截图是`LifecycleListener `的注释介绍了Tomcat启动的状态变化过程

![](https://image.ibb.co/nRj7RL/1539762719678.jpg) [Apache API 介绍](https://tomcat.apache.org/tomcat-8.0-doc/api/org/apache/catalina/Lifecycle.html)

Spring Boot Tomcat 启动

![](https://image.ibb.co/cWdZ6L/1539763041243.jpg)

可以看出第一次调用触发的事件是 `before_init`，很遗憾并不能进行处理。

经过多次debug后发现Tomcat事件依次如下 


1. `before_init` 
2. `after_init` 
3. `before_start` -> 调用 beforeStart();
4. `before_start` -> 调用 beforeStart();
5. `configure_start` -> 调用 configureStart();
6. `start` 
7. `after_start` 
8. `periodic`  不处理，但是开始进行 ServletContainerInitializer 的初始化过程
9.  ...

> 通过多次测试，应该是正在configureStart()进行java SPI 获取数据操作 获取 `ServletContainerInitializer` 中 `@HandlesTypes`感兴趣的类，即这些操作是在 Tomcat `configure_start` 事件触发时执行的代码。

> 在`configureStart()`方法中有一个`webConfig()`的调用

```java
/**
 * Scan the web.xml files that apply to the web application and merge them
 * using the rules defined in the spec. For the global web.xml files,
 * where there is duplicate configuration, the most specific level wins. ie
 * an application's web.xml takes precedence over the host level or global
 * web.xml file.
 */
/**
 *扫描适用于Web应用程序的web.xml文件并合并它们
 *使用规范中定义的规则。 对于全局web.xml文件，
 *如果存在重复配置，则最具体的级别获胜。即
 *应用程序的web.xml优先于主机级别或全局级别
 * web.xml文件。
 */
protected void webConfig() {
	...
}
```

* 在`webConfig()`有`processResourceJARs`处理 servlet3.0中模块化支持的解析，[Oracle 介绍](https://blogs.oracle.com/swchan/servlet-30-web-fragmentxml)，[国内博客介绍](http://elim.iteye.com/blog/2017099)，但是这个一般不使用。

* 在当前这个场景下主要还是关注`processAnnotationsWebResource`这个方法，顾名思义，是处理java注解标注的web资源。
![](https://image.ibb.co/jfosRL/1539765282724.jpg)

> 可以看出会把全部的资源都加载进来进行解析处理，所以我们的定义的 `Bootstrap` 项目辅助启动类也会被加载处理，因为他是继承 `SpringBootServletInitializer ` 所以也是 `WebApplicationInitializer ` 接口的实现类，所以在`SpringServletContainerInitializer` 处理是也可以被感兴趣处理作为参数加载进来。


> 小结： 在Tomcat启动时触发`Lifecycle.CONFIGURE_START_EVENT`事件时，调用`configure_start`会将项目中所有的类加载进来进行处理，作为`@HandlesTypes`感兴趣类的备选

## 编写自己的 SCI SpringServletContainerInitializer

* 首先需要让Tomcat识别到我们自定义的SCI，我们先看看Spring是怎么操作

 Spring 也是利用SPI技术的,有图为证

![](https://image.ibb.co/jAwqD0/1539766314104.jpg)

所以我们也可以这样做

定义一个我们感兴趣的接口

```java
/**
 * SCI 启动测试类
 * @author chenmingming
 * @date 2018/10/16
 */
public interface MyContainerInitalizer {
    void onStartup(ServletContext context);
}
```
编写两个我们这个接口的实现类

```java
public class MyListContainerInitalizer implements MyContainerInitalizer {
    @Override
    public void onStartup(ServletContext context) {
        context.setAttribute("MyListContainerInitalizer",this);
        System.out.println("MyListContainerInitalizer Init ...");
    }
}

public class MyMapContainerInitalizer implements MyContainerInitalizer {
    @Override
    public void onStartup(ServletContext context) {
        context.setAttribute("MyMapContainerInitalizer",this);
        System.out.println("MyMapContainerInitalizer Init ...");
    }
}
```
编写我们自定义的 SPI

```java
@HandlesTypes(MyContainerInitalizer.class)
public class MySCI implements ServletContainerInitializer {
    @Override
    public void onStartup( Set<Class<?>> c, ServletContext ctx) throws ServletException {
        for (Class<?> clazz : c) {
            if(!clazz.isInterface()){
                try {
                    System.out.println(clazz);
                    Constructor<?> constructor = clazz.getConstructor();
                    Object instance = constructor.newInstance();
                    MyContainerInitalizer containerInitalizer = (MyContainerInitalizer)instance;
                    containerInitalizer.onStartup(ctx);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        System.out.println(">>>>>>");
    }
}
```

创建 SPI 必须的 META-INF/services/javax.servlet.ServletContainerInitializer 文件 

```java
cn.hyperchain.sci.MySCI
```
如上即可

启动Tomcat，结果如下，在 Spring 容器启动之前先执行了我们的servlet容器初始化器（SCI），即我们定义的SCI在Spring定义SCI之前先触发。
![](https://image.ibb.co/nODZO0/1539767006088.jpg)

### 那么我们是不是可以抛弃Spring boot方式，自己去启动一个Spring容器？

对自定义SCI代码进行改造，因为所有的类均会被加载，所有先将`Bootstrap `启动类注释掉，其次使用方法内匿名内部类进行处理。具体改造如下

```java
@HandlesTypes(MyContainerInitalizer.class)
public class MySCI implements ServletContainerInitializer {
     @Override
    public void onStartup( Set<Class<?>> c, ServletContext ctx) throws ServletException {
        for (Class<?> clazz : c) {
            if(!clazz.isInterface()){
                try {
                    System.out.println(clazz);
                    Constructor<?> constructor = clazz.getConstructor();
                    Object instance = constructor.newInstance();
                    MyContainerInitalizer containerInitalizer = (MyContainerInitalizer)instance;
                    containerInitalizer.onStartup(ctx);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        SpringBootServletInitializer initializer = new SpringBootServletInitializer(){
            @Override
            protected SpringApplicationBuilder configure(SpringApplicationBuilder application) {
                return application.sources(Application.class);
            }
        };
        initializer.onStartup(ctx);
//        new MyBootstrap().onStartup(ctx);
        System.out.println(">>>>>>");
    }
}
```
* 然而很遗憾在自定义的SCI内部初始化完成了一个Spring容器，但是在`SpringServletContainerInitializer`会报错，认为有一个内部类的存在，还无法被实例化。。。
* 不知道，这个是不是一个bug？

```java
17-Oct-2018 17:10:48.610 严重 [RMI TCP Connection(2)-127.0.0.1] org.apache.catalina.core.StandardContext.startInternal Error during ServletContainerInitializer processing
 javax.servlet.ServletException: Failed to instantiate WebApplicationInitializer class
	at org.springframework.web.SpringServletContainerInitializer.onStartup(SpringServletContainerInitializer.java:155)
```

![](https://image.ibb.co/mfBJGL/WX20181017-172822.png)


所以迫不得已，只能将`SpringBootServletInitializer`重写一遍，覆盖那个`configure `方法，是一个`MyBootstrap`代码太长就不罗列了，注意不要实现`WebApplicationInitializer`接口

所以就代码如下

```java/**
 * @author chenmingming
 * @date 2018/10/16
 */
@HandlesTypes(MyContainerInitalizer.class)
public class MySCI implements ServletContainerInitializer {
    @Override
    public void onStartup( Set<Class<?>> c, ServletContext ctx) throws ServletException {
        for (Class<?> clazz : c) {
            if(!clazz.isInterface()){
                try {
                    System.out.println(clazz);
                    Constructor<?> constructor = clazz.getConstructor();
                    Object instance = constructor.newInstance();
                    MyContainerInitalizer containerInitalizer = (MyContainerInitalizer)instance;
                    containerInitalizer.onStartup(ctx);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
        new MyBootstrap().onStartup(ctx);
        System.out.println(">>>>>>");
    }
}

```
![](https://image.ibb.co/iU5RmL/1539772564312.jpg)

> 所以最好还是使用官方推荐的方法，比较稳一些，这里只是做一个展示而已。