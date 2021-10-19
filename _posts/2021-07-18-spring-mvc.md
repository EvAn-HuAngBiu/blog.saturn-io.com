---
layout: post
title: Spring MVC
categories: Spring
keywords: Spring MVC, Spring, MVC
---

## Spring MVC

### MVC模式

MVC指的是模型-视图-控制器模式，最上面的一层是用户可以看到的展示层即视图（V）层，最下面的一层是直接和数据交互的模型（M）层，中间的控制器（C）层则是负责用户和数据的交互工作，包括用户对数据的修改以及向用户展示数据的行为：

- **Model（模型）** - 模型代表一个存取数据的对象或 JAVA POJO。它也可以带有逻辑，在数据变化时更新控制器。
- **View（视图）** - 视图代表模型包含的数据的可视化。
- **Controller（控制器）** - 控制器作用于模型和视图上。它控制数据流向模型对象，并在数据变化时更新视图。它使视图与模型分离开。

<div align=center><img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/MVC-Arch.png" alt="MVC-Arch" style="zoom:50%;"/></div>

> ​	来源：https://www.runoob.com/design-pattern/mvc-pattern.html

### Web环境中的Spring MVC

首先我们明确一个概念：Spring IoC容器和Web容器之间的关系。IoC容器是用于存储Bean的，Web环境中常用的是`WebApplicationContext`的子类，默认使用的是`XmlWebApplicationContext`，通过加载XML文件的形式获取需要在Web环境中使用的Bean对象；而Web容器则提供了一系列对Web资源访问的可能性，比较常用的Web容器就是Servlet了。所以我们在理解ServletContext和ApplicationContext的时候就知道，ApplicationContext只是一个容器性的概念，而ServletContext则是容器的使用者。所以我们得到这样一个结构：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/ServletContext-ApplicationContext.png" alt="ServletContext-ApplicationContext" style="zoom:48%;" />

习惯上我们会把全局Servlet容器称为ServletContext或Spring容器，把每个Servlet独享的Servlet容器称为Spring MVC容器。每个Servlet容器都有自己的IoC容器，全局IoC容器是所有Servlet共享的，也称为根容器或根上下文；而每个Servlet也独享有自己的IoC容器。

下面举个例子：

以web.xml文件中的context-param中的contextConfigLocation定义的就是全局容器的配置xml文件地址：

```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:config/chcp4-mvc-context.xml</param-value>
</context-param>
```

而每个servlet也可以定义自己的配置xml文件地址：

```xml
<servlet>
    <servlet-name>dispatcher</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
      <param-name>contextConfigLocation</param-name>
      <param-value>classpath:config/dispatcher-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
```

这里的init-parm中的contextConfigLocation就是每个Servlet独享的配置文件的地址了，在这里指定的xml文件中定义的Bean只能被这里的Servlet获取并使用（这里是dispatcher-servlet.xml文件中定义的只能在DispatcherServlet中可用）

**但是，如果你要自己写一个servlet，它要读取自定的contextConfigLocation属性，那么这个容器初始化过程、根上下文的获取工作都只能自己来写，否则当前新建的Servlet中不会包含IoC容器，只会使用根上下文的容器。**

而对于`DispatcherServlet`，它已经定义好了加载过程，所以可以直接读取distpatcher-servlet.xml（文件名可以通过上面的参数修改）中的Bean定义。要分别获取Spring上下文和Spring MVC上下文的方法如下：

```java
// Spring上下文
WebApplicationContext springContext = WebApplicationContextUtils.getWebApplicationContext(request.getSession().getServletContext());

// SpringMVC上下文 
WebApplicationContext springMVCContext = RequestContextUtils.getWebApplicationContext(request);
```

(如果没有定义Servlet自己的容器，那么二者的结果一样都指向根上下文容器即Spring上下文)

而根上下文的启动则是依赖web.xml文件中配置的监听器的时间触发实现的：

```xml
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>
```

默认采用的监听器是`ContextLoaderListener`，这个监听器是与Web服务器的生命周期相关联的，由`ContextLoaderListener`监听器负责完成IoC容器在Web环境中的启动工作。

综上所述，DispatcherServlet和ContextLoaderListener提供了在Web容器中对Spring的接口，**也就是说这些接口与Web容器的耦合是通过ServletContext来完成的**。而ServletContext为Spring的IoC容器提供了一个宿主环境，在宿主环境中，Spring MVC建立起一个IoC容器的体系。这个IoC容器的体系是通过ContextLoaderListener的初始化来建立的，在建立IoC容器体系后，把DisptchServlet作为Spring MVC处理Web请求的转发器建立起来，从而完成响应HTTP请求的准备。

### IoC容器启动的过程

从上面的分析可以知道，IoC容器的启动是在ContextLoaderListener中进行的，我们先来看这个类的继承关系：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/ContextLoaderListener-Arch.png" alt="ContextLoaderListener-Arch" style="zoom:50%;" />

`EventListener`这条线索主要定义了ServletContext时间加载时的监听动作，而`ContextLoader`这个基类则是负责了上下文加载和初始化的核心工作。将这个继承体系合并起来理解就是在ServletContext初始化时实现上下文和IoC容器的初始化，这样就能理解`ContextLoaderListener`的作用了。事实上也是这样，`ContextLoaderListener`类在实现上完全使用基类`ContextLoader`的方法进行初始化，这些初始化的实际则由`ServletContextListener`接口所定义：

```java
public class ContextLoaderListener extends ContextLoader implements ServletContextListener {
	public ContextLoaderListener() {
	}

	public ContextLoaderListener(WebApplicationContext context) {
		super(context);
	}

	@Override
	public void contextInitialized(ServletContextEvent event) {
		initWebApplicationContext(event.getServletContext());
	}

	@Override
	public void contextDestroyed(ServletContextEvent event) {
		closeWebApplicationContext(event.getServletContext());
		ContextCleanupListener.cleanupAttributes(event.getServletContext());
	}
}
```

很明显上述代码没有任何有意义的初始化，完全委托给了基类进行初始化，下面我们开始分析`ContextLoader`类，`ContextLoader`类完成两件事：1. 在Web容器中建立双亲IoC容器，2. 生成相应的WebApplicationContext并将其初始化。

#### Web容器

所谓Web容器就是包装了IoC容器的上层容器，和BeanFactory与ApplicationContext的关系一样，Web容器最上层的接口也是ApplicationContext，并在这个接口之下定义了WebApplicationContext以及其它实现，我们来看继承层次图：

![XmlWebApplicationContext-Arch](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/XmlWebApplicationContext-Arch.png)

这里是以`XmlWebApplicationContext`为基础进行分析的，同时省略了`ApplicationContext`继承的接口。在分析Spring IoC中我们也知道，和IoC容器的对接是通过`ApplicationContext`接口的，而对于Web容器而言，很多操作都封装在了`AbstractRefreshableWebApplicationContext`。

#### WebApplicationContext

我们先来看提供Web容器服务的顶层接口`WebApplicationContext`的源码：

```java
public interface WebApplicationContext extends ApplicationContext {
  // 根容器的名称
	String ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE = WebApplicationContext.class.getName() + ".ROOT";

  // 定义了作用域名称
	String SCOPE_REQUEST = "request";
	String SCOPE_SESSION = "session";
	String SCOPE_APPLICATION = "application";
  
  // 定义了一些参数Bean的名称
	String SERVLET_CONTEXT_BEAN_NAME = "servletContext";
	String CONTEXT_PARAMETERS_BEAN_NAME = "contextParameters";
	String CONTEXT_ATTRIBUTES_BEAN_NAME = "contextAttributes";
	
  // 获得当前Web容器的Servlet上下文环境，通过这个方法相当于提供了一个Web容器级别的全局环境
	ServletContext getServletContext();
}
```

注意一点，这里所说的Web容器即一个Servlet内IoC容器，这里的全局也是指一个Servlet内的全局。注意区别Web容器和Servlet容器之间的关系。

一般来说Spring会使用一个默认的`WebApplicationContext`的实现作为IoC容器，一般使用的是`XmlWebApplicationContext`。这种关系在IoC容器解析时已经说的很清楚了，这里不再赘述，直接来看`XmlWebApplicationContext`的源码。

#### XmlWebApplicationContext

```java
public class XmlWebApplicationContext extends AbstractRefreshableWebApplicationContext {
	// 默认配置文件存储路径
	public static final String DEFAULT_CONFIG_LOCATION = "/WEB-INF/applicationContext.xml";
    // 默认配置文件文件夹
	public static final String DEFAULT_CONFIG_LOCATION_PREFIX = "/WEB-INF/";
    // 默认配置文件扩展名
	public static final String DEFAULT_CONFIG_LOCATION_SUFFIX = ".xml";

	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
        // 加载BeanDefinition的重写，这里和ClasspathXmlApplicationContext的loadBeanDefinitions方法一致
		// 实际读取委托给了XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		beanDefinitionReader.setEnvironment(getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
        
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

	protected void initBeanDefinitionReader(XmlBeanDefinitionReader beanDefinitionReader) {
	}

	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
        // 这个方法也一样，只是配置文件给出的肯定是Location而不是Resource，所以要转换为Resource进行读取
        // 这个转换也是由XmlBeanDefinitionReader来实现的
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			for (String configLocation : configLocations) {
				reader.loadBeanDefinitions(configLocation);
			}
		}
	}

	@Override
	protected String[] getDefaultConfigLocations() {
        // 唯一的一个不同点就是在默认配置文件路径的获取上，Namespace就是文件名，允许在给定的文件目录和扩展名的基础上自定义默认文件名
		if (getNamespace() != null) {
			return new String[] {DEFAULT_CONFIG_LOCATION_PREFIX + getNamespace() + DEFAULT_CONFIG_LOCATION_SUFFIX};
		}
		else {
            // 如果没有给出自定义的默认文件名，那么就使用全局的默认文件名
			return new String[] {DEFAULT_CONFIG_LOCATION};
		}
	}
}
```

这里的设计也和`ClasspathXmlApplicationContext`一致，都是利用了`XmlBeanDefinition`完成具体的XML文件读取、BeanDefinition的解析工作，这些之前都分析过了，所以这里不说了。实际上这个子类覆写了的这两个方法是在`AbstractApplicationContext`的`refresh`方法中使用的，而`refresh`方法的调用以及整个容器的初始化则是在`ContextLoader`中调用的。

#### ContextLoader

接下来我们来看`ContextLoader`，从`ContextLoaderListener`的源码中可以看到，实际初始化发生在`contextInitialized`方法中，调用的是父类也就是`ContextLoader`中的`initWebApplicationContext`方法：

```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    // 如果根上下文已经被创建，抛出异常
    if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        throw new IllegalStateException(
                "Cannot initialize context because there is already a root application context present - " +
                "check whether you have multiple ContextLoader* definitions in your web.xml!");
    }
    long startTime = System.currentTimeMillis();

    try {
        // 存储创建出来的根上下文，这个context在中止前都保持有效
        if (this.context == null) {
            this.context = createWebApplicationContext(servletContext);
        }
        if (this.context instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
            if (!cwac.isActive()) {
                // 如果this.context是ConfigurableWebApplicationContext的实例，并且没有刷新过WebApplicationContext，
                // 则配置并刷新WebApplicationContext。
                if (cwac.getParent() == null) {
                    // 如果上下文被注入的时候没有明确的父上下文，这里尝试加载并注入，默认是null
                    ApplicationContext parent = loadParentContext(servletContext);
                    cwac.setParent(parent);
                }
                // 配置上下文刷新
                configureAndRefreshWebApplicationContext(cwac, servletContext);
            }
        }
        // 将创建出来的context保存为根上下文
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

        ClassLoader ccl = Thread.currentThread().getContextClassLoader();
        if (ccl == ContextLoader.class.getClassLoader()) {
            // 如果当前线程的类加载器和当前ContextLoader的类加载器一致
            // 说明二者初始化线程一样，保存在全局变量currentContext中（WebApplicationContext）属性
            currentContext = this.context;
        }
        else if (ccl != null) {
            // 建立类加载器到当前WebApplicationContext容器的映射
            currentContextPerThread.put(ccl, this.context);
        }
        // 返回初始化好的WebApplicationContext
        return this.context;
    }
    catch (RuntimeException | Error ex) {
        logger.error("Context initialization failed", ex);
        servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
        throw ex;
    }
}
```

很简单，核心流程就是先创建根上下文保存在context字段中并设置到ServletContext的根上下文属性中，这里还要考虑有可能对于WebApplicationContext的初始化而言，需要考虑是否已经存在一个没有刷新的容器，如果是没有刷新的，那么就刷新它并且设置双亲上下文。

我们先来看其中创建上下文的方法`createWebApplicationContext`：

```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    // 委托给determineContextClass函数决定使用哪个WebApplicationContext的实现类
    Class<?> contextClass = determineContextClass(sc);
    // 这个实现类必须是ConfigurableWebApplicationContext的子类，即必须具有可配置的能力
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
                "] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
    }
    // 实例化
    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}
```

很明显交给了`determineContextClass`来决定到底使用哪个类作为默认的IoC容器：

```java
protected Class<?> determineContextClass(ServletContext servletContext) {
    // 如果在web.xml文件中指定了Context的类型那么就尝试加载这个类
    String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
    if (contextClassName != null) {
        try {
            return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                    "Failed to load custom context class [" + contextClassName + "]", ex);
        }
    }
    else {
        // 否则使用默认策略中的类
        contextClassName = defaultStrategies.getProperty(WebApplicationContext.class.getName());
        try {
            return ClassUtils.forName(contextClassName, ContextLoader.class.getClassLoader());
        }
        catch (ClassNotFoundException ex) {
            throw new ApplicationContextException(
                    "Failed to load default context class [" + contextClassName + "]", ex);
        }
    }
}
```

这个默认策略实际上是已经写在spring-webmvc组件中的ContextLoader.properties中定义好的，这个文件中只定义了这一个属性：

```properties
org.springframework.web.context.WebApplicationContext=org.springframework.web.context.support.XmlWebApplicationContext
```

可以看到，默认的IoC容器使用的是`XmlWebApplicationContext`。到这里，我们就获得了一个可以使用WebApplicationContext的实例对象，我们知道IoC容器的初始化是在`AbstractApplicationContext`的`refresh`方法中实现的，而在`ContextLoader`中实际上调用了`configureAndRefreshWebApplicationContext`来完成这项工作：

```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        // 如果在web.xml中指定了ID，那么就使用这个指定的ID
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        }
        else {
            // 否则则创建一个ID
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                    ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }
    // 将当前IoC容器和Servlet容器关联，这一步实现了根上下文和它的共享容器之间的关联
    wac.setServletContext(sc);
    // 如果指定了需要加载的全局Bean配置，那么就设置这个配置，供getConfigLocations方法使用
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        wac.setConfigLocation(configLocationParam);
    }

    // 先初始化配置文件源供后续使用
    ConfigurableEnvironment env = wac.getEnvironment();
    if (env instanceof ConfigurableWebEnvironment) {
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }
    // 未知
    customizeContext(sc, wac);
    // 调用AbstractApplicationContext实现初始化
    wac.refresh();
}
```

可以看到实际上还是调用了`AbstractApplicationContext`的`refresh`方法完成初始化。`refresh`的具体步骤不再分析。

至此，我们就已经将全局ServletContext的IoC容器和全局ServletContext绑定起来，并且完成了初始化，到这里就说明根上下文已经初始化完成，这个根上下文在未来将会被设置成每一个Servlet的双亲上下文。

### Spring MVC的实现

在完成对`ContextLoaderListener`的初始化以后，Web容器开始初始化`DispatcherServlet`，DispatcherServlet会建立自己的上下文来持有Spring MVC的Bean对象，在建立这个自己持有的IoC容器时，会从ServletContext中得到根上下文作为DispatcherServlet的持有的上下文的双亲上下文。

为了了解DispatcherServlet的启动过程，我们先来看它的继承关系：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/DispatcherServlet-Arch.png" alt="DispatcherServlet-Arch" style="zoom:50%;" />

它的核心功能很大一部分由父类`FrameworkServlet`实现，同时也因为间接继承了`HttpServlet`类使得DispatcherServlet具有处理HTTP请求的能力。DispatcherServlet的工作大概分为两部分：

1. 初始化部分，由`initServletBean`启动，通过`initWebApplicationContext`方法最终调用DispatcherServlet的`initStrategies`方法，对MVC模块的其他部分进行了初始化，如HandlerMapping、ViewResolver等；
2. 2.对HTTP请求进行响应，Web容器会调用Servlet的doGet或doPost方法，在经过FrameworkServlet的`processRequest`方法处理之后，会调用DispatcherServlet的`doService`方法，在这个方法里封装了`doDispatch`来实现整个MVC功能。

#### DispatcherServlet的启动和初始化

对于Servlet而言，它定义的初始化接口为init函数：

```java
public void init(ServletConfig config) throws ServletException;
```

对于DispatcherServlet而言，它直接使用了父类`HttpServletBean`实现好的`init`方法：

```java
public final void init() throws ServletException {
    // 编程式读取web.xml中配置的关于DispatcherServlet的init-parm
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
            bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
            initBeanWrapper(bw);
            bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
            if (logger.isErrorEnabled()) {
                logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
            }
            throw ex;
        }
    }
    // 由子类FrameworkServlet实现具体的初始化
    initServletBean();
}
```

可以看到，在这里，先读取了在web.xml配置的当前Servlet的init-param，然后交由子类进行具体的初始化，具体的初始化过程由`FrameworkServlet`来实现：

```java
protected final void initServletBean() throws ServletException {
    long startTime = System.currentTimeMillis();
    try {
        // 初始化当前Servlet自己持有的IoC容器
        this.webApplicationContext = initWebApplicationContext();
        // 允许子类重写进行自定义初始化
        initFrameworkServlet();
    }
    catch (ServletException | RuntimeException ex) {
        logger.error("Context initialization failed", ex);
        throw ex;
    }
}
```

在初始化过程中，会创建一个属于当前Servlet的上下文。**可以认为，根上下文是和Web应用相对应的一个上下文，而DispatcherServlet持有的上下文是和Servlet对应的一个上下文。在一个Web应用中允许存在多个Servlet，而每个Servlet在寻找Bean时会首先尝试在双亲上下文中进行寻找。**接下来我们来看创建IoC容器的过程：

```java
protected WebApplicationContext initWebApplicationContext() {
    // 获取根上下文
    WebApplicationContext rootContext =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    // 当前应用上下文
    WebApplicationContext wac = null;

    // 如果当前Servlet持有的上下文不为空，说明可能已经创建过了，或者是通过构造函数传入的
    if (this.webApplicationContext != null) {
        wac = this.webApplicationContext;
        // 同样尝试设置当前上下文的双亲上下文为根上下文并初始化这个上下文
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                if (cwac.getParent() == null) {
                    cwac.setParent(rootContext);
                }
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    // 否则说明当前上下文还没有被创建，这里首先尝试在当前Servlet的Attribute中寻找，
    // 如果找到就利用这个已经创建好的上下文
    if (wac == null) {
        wac = findWebApplicationContext();
    }
    // 否则就说明上下文还没有被创建，那就创建一个上下文（和根上下文的创建过程几乎一致）
    if (wac == null) {
        wac = createWebApplicationContext(rootContext);
    }

    if (!this.refreshEventReceived) {
        // 触发刷新事件
        synchronized (this.onRefreshMonitor) {
            onRefresh(wac);
        }
    }

    if (this.publishContext) {
        // 将当前上下文保存到Servlet的属性域中
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
    }

    return wac;
}

protected WebApplicationContext findWebApplicationContext() {
    // 获取保存在属性的上下文
    String attrName = getContextAttribute();
    if (attrName == null) {
        return null;
    }
    WebApplicationContext wac =
            WebApplicationContextUtils.getWebApplicationContext(getServletContext(), attrName);
    if (wac == null) {
        throw new IllegalStateException("No WebApplicationContext found: initializer not registered?");
    }
    return wac;
}
```

这个初始化过程还是简单的，首先判断是否存在一个没有被激活的上下文，如果有则设置双亲并激活；如果没有这样的上下文，尝试在属性中查找是否有已经创建的上下文，如果有则使用它；否则创建一个新的上下文。这里创建新上下文的方法`createWebApplicationContext`就忽略了，它和根上下文的创建几乎一致，只是多了刷新和设置双亲上下文的步骤，其余的都这不多。

好了到目前为止，整个DispatcherServlet的IoC容器就创建和初始化了，可以看到，这个流程和根上下文的初始化大同小异，所以我们不做过多的分析。接下来主要分析一下其它MVC模块的初始化流程。

最核心的初始化入口为DispatcherServlet中的`initStrategies`方法，我们先来看整个调用栈：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/onRefresh-CallStack.png" alt="onRefresh-CallStack" style="zoom:50%;" />

可以看到，实际上调用初始化入口的是在`AbstractApplicationContext`中的`refresh`方法发布了上下文刷新事件时，调用了当前这个`initStrategies`方法，其源码如下：

```java
protected void initStrategies(ApplicationContext context) {
    initMultipartResolver(context);
    initLocaleResolver(context);
    initThemeResolver(context);
    initHandlerMappings(context);
    initHandlerAdapters(context);
    initHandlerExceptionResolvers(context);
    initRequestToViewNameTranslator(context);
    initViewResolvers(context);
    initFlashMapManager(context);
}
```

我们就说最重要的`initHandlerMappings`方法，这里的Mapping关系的作用是，**为HTTP请求找到相应的Controller控制器**，从而利用这些控制器去完成设计好的数据处理工作。**要注意，我们如果自定义了Servlet，那么自定义的Servlet的处理关系是由Tomcat来负责分发的，并不会经过DispatcherServlet，而DispatcherServlet只负责分发需要转交给Controller执行的任务。**HandlerMappings完成对MVC中Controller的定义和配置，这些Controller和具体的HTTP请求相对应，我们来看初始化的代码：

```java
private void initHandlerMappings(ApplicationContext context) {
    this.handlerMappings = null;

    if (this.detectAllHandlerMappings) {
        // 获取所有注册的HandlerMapping，包括双亲上下文中注册过的
        Map<String, HandlerMapping> matchingBeans =
                BeanFactoryUtils.beansOfTypeIncludingAncestors(context, HandlerMapping.class, true, false);
        if (!matchingBeans.isEmpty()) {
            // 保存到当前ServletContext中
            this.handlerMappings = new ArrayList<>(matchingBeans.values());
            AnnotationAwareOrderComparator.sort(this.handlerMappings);
        }
    }
    else {
        // 如果不允许获取双亲上下文中的HandlerMapping，那么就尝试在当前上下文中找到一个可用的HandlerMapping
        try {
            HandlerMapping hm = context.getBean(HANDLER_MAPPING_BEAN_NAME, HandlerMapping.class);
            this.handlerMappings = Collections.singletonList(hm);
        }
        catch (NoSuchBeanDefinitionException ex) {
        }
    }

    // 如果还是没有一个可用的HandlerMapping，那么使用默认的HandlerMapping
    if (this.handlerMappings == null) {
        this.handlerMappings = getDefaultStrategies(context, HandlerMapping.class);
    }
}
```

可以看到整个初始化流程还是比较简单的，就是从判断是应该从双亲、当前上下文中获取`HandlerMapping`还是使用默认配置。默认配置保存在spring-webmvc包中的DispatcherServlet.properties中，定义如下：

```properties
org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping,\
	org.springframework.web.servlet.function.support.RouterFunctionMapping
```

经过以上处理，HandlerMapping就已经完成了读取并保存到了当前ServletContext之中了，在出现HTTP请求时，就会利用上述HandlerMapping来匹配相应的Controller。

#### MVC处理HTTP分发请求

我们上面分析之后知道，Spring MVC对HTTP请求的处理是通过HandlerMapping实现的，一个HandlerMapping可以持有一系列的从URL请求到Controller的映射，Spring提供了一系列的HandlerMapping，其继承层次图如下：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/HandlerMapping-Arch.png" alt="HandlerMapping-Arch" style="zoom:50%;" />



我们首先来看最核心的接口即`HandlerMapping`接口的实现：

```java
public interface HandlerMapping {    // 常量定义	String BEST_MATCHING_HANDLER_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingHandler";	String LOOKUP_PATH = HandlerMapping.class.getName() + ".lookupPath";	String PATH_WITHIN_HANDLER_MAPPING_ATTRIBUTE = HandlerMapping.class.getName() + ".pathWithinHandlerMapping";	String BEST_MATCHING_PATTERN_ATTRIBUTE = HandlerMapping.class.getName() + ".bestMatchingPattern";	String INTROSPECT_TYPE_LEVEL_MAPPING = HandlerMapping.class.getName() + ".introspectTypeLevelMapping";	String URI_TEMPLATE_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".uriTemplateVariables";	String MATRIX_VARIABLES_ATTRIBUTE = HandlerMapping.class.getName() + ".matrixVariables";	String PRODUCIBLE_MEDIA_TYPES_ATTRIBUTE = HandlerMapping.class.getName() + ".producibleMediaTypes";    // 返回一个执行链，包括对应的Handler，以及处理这个HTTP请求相关的拦截器	@Nullable	HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception;}
```

它只定义了一个方法，即`getHandler`方法，它返回一个HandlerExecutionChain对象，在这个对象中封装了具体的Controller对象以及一个Interceptor链。HandlerExecutionChain对象实际上就是对Handler的一个包装，它的结构如下：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/HandlerExecutionChain-Arch.png" alt="HandlerExecutionChain-Arch" style="zoom:50%;" />

很明显，内部持有handler对象（代表Controller）以及interceptors链，并且同时支持添加拦截器。对于一个具体的HandlerMapping实现来说，它就需要根据URL映射的方式，注册Handler和interceptor，从而维护一个反映这种映射关系的handlerMap，当有一个请求到达时，就需要查找handlerMap来找到对应的HandlerExecutionChain。这个配置过程是通过一个Bean的postProcessor来完成的。

我们接下来以SimpleUrlHandlerMapping为例子来分析整个初始化过程：

实际上对于HandlerMapping而言，它的初始化发生在Application初始化之后，会调用ApplicationContextAware接口的`setApplicationContext`方法，而`ApplicationObjectSupport`类实现了这个接口，而`ApplicationObjectSupport`类在获取到ApplicationContext后，在`setApplicationContext`中调用了`initApplicationContext`方法完成初始化，而`SimpleHandlerMapping`又重写了`initApplicationContext`方法，所以我们来看最终调用的这个无参的`initApplicationContext`方法：

```java
	public void initApplicationContext() throws BeansException {
		super.initApplicationContext();
		registerHandlers(this.urlMap);
	}
```

可以看到核心就是`registerHandlers`方法：

```java
protected void registerHandlers(Map<String, Object> urlMap) throws BeansException {
    if (urlMap.isEmpty()) {
        logger.trace("No patterns in " + formatMappingName());
    }
    else {
        urlMap.forEach((url, handler) -> {
            // Prepend with slash if not already present.
            if (!url.startsWith("/")) {
                url = "/" + url;
            }
            // Remove whitespace from handler bean name.
            if (handler instanceof String) {
                handler = ((String) handler).trim();
            }
            // 实际注册
            registerHandler(url, handler);
        });
    }
}
```

接着跟踪`registerHandler`方法，这个方法是在父类`AbstractUrlHandlerMapping`中实现的：

```java
protected void registerHandler(String urlPath, Object handler) throws BeansException, IllegalStateException {
    Assert.notNull(urlPath, "URL path must not be null");
    Assert.notNull(handler, "Handler object must not be null");
    Object resolvedHandler = handler;

    // 如果不启用懒加载，并且给出的handler是一个Controller Bean的名称，那么直接尝试获取Bean
    if (!this.lazyInitHandlers && handler instanceof String) {
        String handlerName = (String) handler;
        ApplicationContext applicationContext = obtainApplicationContext();
        if (applicationContext.isSingleton(handlerName)) {
            resolvedHandler = applicationContext.getBean(handlerName);
        }
    }

    Object mappedHandler = this.handlerMap.get(urlPath);
    if (mappedHandler != null) {
        // 如果Map中已经存在一个urlPath到Handler的映射，那么要判断Handler是否一致
        // 不允许出现有一个urlPath同时使用多个Handler
        if (mappedHandler != resolvedHandler) {
            throw new IllegalStateException(
                    "Cannot map " + getHandlerDescription(handler) + " to URL path [" + urlPath +
                    "]: There is already " + getHandlerDescription(mappedHandler) + " mapped.");
        }
    }
    else {
        // 否则说明Map中不存在这样的映射
        if (urlPath.equals("/")) {
            // 处理根目录常见
            setRootHandler(resolvedHandler);
        }
        else if (urlPath.equals("/*")) {
            // 处理默认场景（通配符）
            setDefaultHandler(resolvedHandler);
        }
        else {
            // 处理正常URL映射
            this.handlerMap.put(urlPath, resolvedHandler);
        }
    }
}
```

在这个处理流程中，如果使用Bean的名称作为映射，那么直接从容器中获取这个HTTP映射对应的Bean，然后还要对不同的URL配置进行解析处理，比如根目录/，以及通配符/*的URL和正常URL请求，完成这个解析过程后就把URL和handler作为键值对放到一个handlerMap中去，完成了对Handler的解析和准备。

以上就是整个HandlerMapping的处理过程，但是上述过程有一点抽象，我们从头开始梳理一下这个初始化流程：

首先来看整个请求的调用栈：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/HandlerReg-Process.png" alt="HandlerReg-Process" style="zoom:50%;" />

首先明确，所有HandlerMapping实际上都是一个Bean，都会在BeanDefinition中保存，以Tomcat为例，会将`RequestMappingHandlerMapping`、`BeanNameUrlHandlerMapping`以及`SimpleUrlHandlerMapping`加入BeanDefinition中，这些HandlerMapping是默认的HandlerMapping。

#### Handler及HandlerMapping的加载时机

流程：

1. 首先调用发生在ContextLoaderListener中，由Tomcat触发了ContextIntialized事件；而ContextLoaderListener又委托给ContextLoader完成初始化
2. ContextLoader创建WebApplicationContext，并且完成对XmlWebApplicationContext的初始化即调用refresh函数
3. 在AbstractApplicationContext中的refresh函数中调用finishBeanFactoryInitialization方法提前加载不需要懒加载的Bean
4. 对于上述提到的HandlerMapping均设置了不需要懒加载，所以会调用getBean方法创建实例
5. 在实例创建结束以及属性填充完毕后，对加载的HandlerMapping Bean应用BeanPostProcessor
6. 由于ApplicationObjectSupport类实现了ApplicationContextAware接口，所以所有HandlerMapping子类都实现类该接口，在BeanPostProcessor中，会被ApplicationContextAwareProcessor处理
7. 调用ApplicationObjectSupport的setApplicationContext方法，传递ApplicationContext
8. 由ApplicationObjectSupport方法逐级调用initApplicationContext方法，这里会调用到具体HandlerMapping的initApplicationContext方法
9. 由具体HandlerMapping负责注册和加载所有的Handler，并维护URL到Handler以及拦截器的映射

上述就是整个Handler的加载流程的，Handler是由HandlerMapping触发加载的，触发时机是在HandlerMapping实现ApplicationContextAware接口的setApplicationContext方法中。

好了，以上就是整个MVC对Handler和HandlerMapping的初始化流程，至此，MVC已经完成了所有的准备工作，等待接受HTTP请求即可为其提供服务。

#### HandlerMapping匹配

我们这里还是以`SimpleUrlHandlerMapping`来分析其最为核心的`getHandler`方法，这里的`getHandler`方法是在父类`AbstractHandlerMapping`中实现的：

```java
public final HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    // 获取handler
    Object handler = getHandlerInternal(request);
    // 如果handler为空则使用默认handler
    if (handler == null) {
        handler = getDefaultHandler();
    }
    // 如果还空说明没有能够处理这个请求的Controller
    if (handler == null) {
        return null;
    }
    // 如果handler是一个字符串，那么就获取对应的Bean
    if (handler instanceof String) {
        String handlerName = (String) handler;
        handler = obtainApplicationContext().getBean(handlerName);
    }
    // 包装为HandlerExecutionChain对象，主要是包装拦截器
    HandlerExecutionChain executionChain = getHandlerExecutionChain(handler, request);
    // 处理CORS的相关配置
    if (hasCorsConfigurationSource(handler) || CorsUtils.isPreFlightRequest(request)) {
        CorsConfiguration config = (this.corsConfigurationSource != null ? this.corsConfigurationSource.getCorsConfiguration(request) : null);
        CorsConfiguration handlerConfig = getCorsConfiguration(handler, request);
        config = (config != null ? config.combine(handlerConfig) : handlerConfig);
        executionChain = getCorsHandlerExecutionChain(request, executionChain, config);
    }
    return executionChain;
}
```

可以看到，最终调用的是`getHandlerExecutionChain`来将handler和interceptor结合起来：

```java
protected HandlerExecutionChain getHandlerExecutionChain(Object handler, HttpServletRequest request) {    // 如果当前handler已经是一个包装好的对象（缓存过），那就直接使用，否则新建一个    HandlerExecutionChain chain = (handler instanceof HandlerExecutionChain ?            (HandlerExecutionChain) handler : new HandlerExecutionChain(handler));    // 获取当前请求的地址    String lookupPath = this.urlPathHelper.getLookupPathForRequest(request, LOOKUP_PATH);    for (HandlerInterceptor interceptor : this.adaptedInterceptors) {        if (interceptor instanceof MappedInterceptor) {            // 如果是需要判断URL的拦截器，那么需要判断一下是否可以被添加到当前handler中            MappedInterceptor mappedInterceptor = (MappedInterceptor) interceptor;            if (mappedInterceptor.matches(lookupPath, this.pathMatcher)) {                chain.addInterceptor(mappedInterceptor.getInterceptor());            }        }        else {            // 否则说明是全局拦截器，直接添加即可            chain.addInterceptor(interceptor);        }    }    return chain;}
```

而从`getHandler`方法中可以看到，`getHandlerInternal`是获取Handler的核心方法，这个方法在`AbstractHandlerMapping`中是一个抽象方法，其实现在`AbstractUrlHandlerMapping`中：

```java
protected Object getHandlerInternal(HttpServletRequest request) throws Exception {
    // 获取请求URL
    String lookupPath = getUrlPathHelper().getLookupPathForRequest(request);
    request.setAttribute(LOOKUP_PATH, lookupPath);
    // 查找对应的handler
    Object handler = lookupHandler(lookupPath, request);
    if (handler == null) {
        Object rawHandler = null;
        // 判断根handler，和默认handler
        if ("/".equals(lookupPath)) {
            rawHandler = getRootHandler();
        }
        if (rawHandler == null) {
            rawHandler = getDefaultHandler();
        }
        // 处理普通handler
        if (rawHandler != null) {
            if (rawHandler instanceof String) {
                String handlerName = (String) rawHandler;
                rawHandler = obtainApplicationContext().getBean(handlerName);
            }
            validateHandler(rawHandler, request);
            handler = buildPathExposingHandler(rawHandler, lookupPath, lookupPath, null);
        }
    }
    return handler;
}
```

这个流程很明显，最关键的就是`lookupHandler`方法，但是这里给出代码，不分析这个方法，它实现的就是寻找一个最符合当前URL的Handler：

```java
protected Object lookupHandler(String urlPath, HttpServletRequest request) throws Exception {
    // 直接匹配
    Object handler = this.handlerMap.get(urlPath);
    if (handler != null) {
        if (handler instanceof String) {
            String handlerName = (String) handler;
            handler = obtainApplicationContext().getBean(handlerName);
        }
        validateHandler(handler, request);
        return buildPathExposingHandler(handler, urlPath, urlPath, null);
    }

    // 模式匹配
    List<String> matchingPatterns = new ArrayList<>();
    for (String registeredPattern : this.handlerMap.keySet()) {
        if (getPathMatcher().match(registeredPattern, urlPath)) {
            matchingPatterns.add(registeredPattern);
        }
        else if (useTrailingSlashMatch()) {
            if (!registeredPattern.endsWith("/") && getPathMatcher().match(registeredPattern + "/", urlPath)) {
                matchingPatterns.add(registeredPattern + "/");
            }
        }
    }
    // 构建最佳比较器
    String bestMatch = null;
    Comparator<String> patternComparator = getPathMatcher().getPatternComparator(urlPath);
    if (!matchingPatterns.isEmpty()) {
        matchingPatterns.sort(patternComparator);
        bestMatch = matchingPatterns.get(0);
    }
    if (bestMatch != null) {
        handler = this.handlerMap.get(bestMatch);
        if (handler == null) {
            if (bestMatch.endsWith("/")) {
                handler = this.handlerMap.get(bestMatch.substring(0, bestMatch.length() - 1));
            }
            if (handler == null) {
                throw new IllegalStateException(
                        "Could not find handler for best pattern match [" + bestMatch + "]");
            }
        }
        if (handler instanceof String) {
            String handlerName = (String) handler;
            handler = obtainApplicationContext().getBean(handlerName);
        }
        validateHandler(handler, request);
        String pathWithinMapping = getPathMatcher().extractPathWithinPattern(bestMatch, urlPath);

        // 这里处理如果有多个最佳匹配的情况
        Map<String, String> uriTemplateVariables = new LinkedHashMap<>();
        for (String matchingPattern : matchingPatterns) {
            if (patternComparator.compare(bestMatch, matchingPattern) == 0) {
                Map<String, String> vars = getPathMatcher().extractUriTemplateVariables(matchingPattern, urlPath);
                Map<String, String> decodedVars = getUrlPathHelper().decodePathVariables(request, vars);
                uriTemplateVariables.putAll(decodedVars);
            }
        }
        return buildPathExposingHandler(handler, bestMatch, pathWithinMapping, uriTemplateVariables);
    }

    // 没有合适的handler
    return null;
}
```

经过上述一系列方法，就可以得到包装有handler和interceptor的HandlerExecutionChain对象，依靠这个对象，DispatcherServlet就能够很好的处理HTTP请求了，下面我们来看DispatcherServlet处理HTTP请求的流程：

#### DispatcherServlet分发请求

我们知道，HttpServlet类已经定义了处理各类请求的方法如下：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring MVC/HttpServlet-Arch.png" alt="HttpServlet-Arch" style="zoom:50%;" />

可以看到，对于各种的HTTP请求都有对应的方法进行处理，但是这些方法在`DispatcherServlet`的父类`FrameworkServlet`中都被重写了：

```java
protected final void doGet(HttpServletRequest request, HttpServletResponse response)			throws ServletException, IOException {		processRequest(request, response);}protected final void doPost(HttpServletRequest request, HttpServletResponse response)        throws ServletException, IOException {    processRequest(request, response);}protected final void doPut(HttpServletRequest request, HttpServletResponse response)        throws ServletException, IOException {    processRequest(request, response);}protected final void doDelete(HttpServletRequest request, HttpServletResponse response)        throws ServletException, IOException {    processRequest(request, response);}
```

除了OPTION和TRACE方法，基本上都交给`processRequest`处理了：

```java
protected final void processRequest(HttpServletRequest request, HttpServletResponse response)
			throws ServletException, IOException {

		long startTime = System.currentTimeMillis();
		Throwable failureCause = null;

		LocaleContext previousLocaleContext = LocaleContextHolder.getLocaleContext();
		LocaleContext localeContext = buildLocaleContext(request);

		RequestAttributes previousAttributes = RequestContextHolder.getRequestAttributes();
		ServletRequestAttributes requestAttributes = buildRequestAttributes(request, response, previousAttributes);

		WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);
		asyncManager.registerCallableInterceptor(FrameworkServlet.class.getName(), new RequestBindingInterceptor());

		initContextHolders(request, localeContext, requestAttributes);

		try {
      // 核心点：交给了doService处理请求
			doService(request, response);
		}
		catch (ServletException | IOException ex) {
			failureCause = ex;
			throw ex;
		}
		catch (Throwable ex) {
			failureCause = ex;
			throw new NestedServletException("Request processing failed", ex);
		}

		finally {
			resetContextHolders(request, previousLocaleContext, previousAttributes);
			if (requestAttributes != null) {
				requestAttributes.requestCompleted();
			}
			logResult(request, response, failureCause, asyncManager);
			publishRequestHandledEvent(request, response, startTime, failureCause);
		}
	}
```

这里就一个重点，实际上`processRequest`将请求交给了`doService`处理，而`FrameworkServlet`中的`doService`方法是抽象方法，实际上交给了`DispatcherServlet`中的`doService`方法处理，我们分析`doService`方法可以知道，其实际上是交给了`doDispatch`方法完成核心功能，所以这里直接看`doDispatch`方法：

```java
protected void doDispatch(HttpServletRequest request, HttpServletResponse response) throws Exception {
    HttpServletRequest processedRequest = request;
    HandlerExecutionChain mappedHandler = null;
    boolean multipartRequestParsed = false;
    // 处理异步请求
    WebAsyncManager asyncManager = WebAsyncUtils.getAsyncManager(request);

    try {
        ModelAndView mv = null;
        Exception dispatchException = null;

        try {
            // 判断是否是多部分请求，例如文件上传等
            processedRequest = checkMultipart(request);
            multipartRequestParsed = (processedRequest != request);

            // 获取处理的Handler
            mappedHandler = getHandler(processedRequest);
            if (mappedHandler == null) {
                // 如果没有有效的handler，实际上这里会返回404错误
                noHandlerFound(processedRequest, response);
                return;
            }

            // 获得请求适配器，主要是为了检查Handler是否合法，是否是按要求编写的handler
            HandlerAdapter ha = getHandlerAdapter(mappedHandler.getHandler());

            // 对于GET方法和HEAD方法，首先判断数据是否发生改变，如果没有发生改变，那么直接返回缓存即可
            String method = request.getMethod();
            boolean isGet = "GET".equals(method);
            if (isGet || "HEAD".equals(method)) {
                long lastModified = ha.getLastModified(request, mappedHandler.getHandler());
                if (new ServletWebRequest(request, response).checkNotModified(lastModified) && isGet) {
                    return;
                }
            }
            // 调用handler中的方法执行器前拦截器
            if (!mappedHandler.applyPreHandle(processedRequest, response)) {
                return;
            }

            // 实际调用Controller处理请求
            mv = ha.handle(processedRequest, response, mappedHandler.getHandler());

            if (asyncManager.isConcurrentHandlingStarted()) {
                return;
            }
            // 设置视图
            applyDefaultViewName(processedRequest, mv);
            mappedHandler.applyPostHandle(processedRequest, response, mv);
        }
        catch (Exception ex) {
            dispatchException = ex;
        }
        catch (Throwable err) {
            dispatchException = new NestedServletException("Handler dispatch failed", err);
        }
        // 处理返回值
        processDispatchResult(processedRequest, response, mappedHandler, mv, dispatchException);
    }
    catch (Exception ex) {
        triggerAfterCompletion(processedRequest, response, mappedHandler, ex);
    }
    catch (Throwable err) {
        triggerAfterCompletion(processedRequest, response, mappedHandler,
                new NestedServletException("Handler processing failed", err));
    }
    finally {
        if (asyncManager.isConcurrentHandlingStarted()) {
            // Instead of postHandle and afterCompletion
            if (mappedHandler != null) {
                mappedHandler.applyAfterConcurrentHandlingStarted(processedRequest, response);
            }
        }
        else {
            // Clean up any resources used by a multipart request.
            if (multipartRequestParsed) {
                cleanupMultipart(processedRequest);
            }
        }
    }
}
```

这个方法虽然很长，但是逻辑很清楚：首先获取handler，将其用适配器包装；判断是否需要重新执行Controller，是否可以用缓存数据直接返回；如果不行，那么先调用执行器拦截器，然后调用Controller并设置返回值，最后调用执行后拦截器；最后处理返回值返回即可。

而取得handler则是调用了`getHandler`方法：

```java
protected HandlerExecutionChain getHandler(HttpServletRequest request) throws Exception {
    if (this.handlerMappings != null) {
        for (HandlerMapping mapping : this.handlerMappings) {
            HandlerExecutionChain handler = mapping.getHandler(request);
            if (handler != null) {
                return handler;
            }
        }
    }
    return null;
}
```

本质上就是调用了`DispatcherServlet`中保存的handlerMapping中每个mapping的getHandler方法实现的，如果不能匹配当前URL，则返回null，只要找到一个能匹配的，就直接返回。

经过以上过程，实际上DispatcherServlet就可以能够正确的处理HTTP请求了，总结一下就是：DispatcherServlet寻找已经缓存的HandlerMapping，再调用对应Handler执行Controller即可。