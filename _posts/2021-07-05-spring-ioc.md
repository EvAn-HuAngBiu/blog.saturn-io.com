---
layout: post
title: Spring IoC
categories: Spring
keywords: Spring IoC, Spring, IoC
---

## Spring IoC

IoC：对象反转容器，**解决的问题**如果合作对象的引用或依赖关系的管理由具体的对象管理会导致代码的高度耦合性和可测试性的降低。IoC则在对象生成或初始化时直接将数据注入到对象中，也可以通过将对象引用注入到对象数据域中的方式来注入对方法调用的依赖（**简而言之就是构造器注入和设值注入**）。这种注入可以是递归，对象被逐层注入。

## Spring中IoC容器的实现

IoC容器的**最顶层接口**为`BeanFactory`，仅实现了该接口的类为简单容器，这系列容器只包含了容器的最基本功能，`BeanFactory`类的方法如下：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring IoC/BeanFactory-Method.png" alt="BeanFactory-Method" style="zoom:50%;" />



一般常用的容器为ApplicationContext接口的实现类，它是容器的高级形态，封装了一系列的容器操作（如读取、解析、注册等，继承了`BeanFactory`接口），同时通过继承接口的方式实现支持不同信息源(`MessageSource`)、访问资源(`ResourceLoader`)、支持应用事件(`ApplicationEventPublisher`)以及其它附加服务等，ApplicationContext类的继承层次如下：

![ApplicationContext-Hierachy](/images/Spring/Spring IoC/ApplicationContext-Hierachy.png)

对上述流程图的一些理解

- 从`BeanFactory`到`HierachicalBeanFactory`再到`ConfigurableBeanFactory`是一条主要的`BeanFactory`设计路径。在这条路径中`BeanFactory`接口定义了基本IoC容器的规范，`HierachicalBeanFactory`接口提供了访问父类容器的可能性，而`ConfigurableBeanFactory`接口则提供了一些对`BeanFactory`的配置功能。

- 第二条设计主线是从`BeanFactory`到`ListableBeanFactory`再到`ApplicationContext`以及`ApplicationContext`的一系列子接口，如`WebApplicationContext`和`ConfigurableApplicationContext`等等。其中`ListableBeanFactory`提供了对枚举容器内Bean的可能性。

  关于`ListableBeanFactory`接口的作用，源码中的注释是这样说的：

  Extension of the {@link BeanFactory} interface to be implemented by bean factories **that can enumerate all their bean  instances, rather than attempting bean lookup by name one by one as requested by clients**. BeanFactory implementations that preload all their bean definitions (such as XML-based factories) may implement this interface.

- 上述提供的只是接口的继承层次图，而具体的IoC容器都是在这个接口体系下实现的，例如最为常用的IoC容器即`DefaultLisableBeanFactory`就是`ConfigurableBeanFactory`的一个实现类

综上所述，`BeanFactory`接口定义了IoC容器最基本的形式，并且提供了IoC容器所应该遵守的最基本的服务契约。



## BeanFactory和FactoryBean

直观理解：BeanFactory是指一个对象工厂，用来管理Bean的；而FactoryBean它实际上是一个Bean，用来产生Bean的。也就是说BeanFactory是一个IoC容器，用于储存所有加载的Bean，而我们可以自定义一个FactoryBean对象来生产我们需要的Bean对象。下面给出一个示例代码：

1. 定义一个需要装配到IoC容器的对象（这里以Service接口及其实现来测试）

   ```java
   public interface FactoryBeanService {
       void testFactoryBean();
   }
   
   public class FactoryBeanServiceImpl implements FactoryBeanService {
       @Override
       public void testFactoryBean() {
           System.out.println("This is a factory bean test method");
       }
   }
   ```

2. 定义一个FactoryBean对象

   ```java
   public class TestFactoryBean implements FactoryBean<FactoryBeanService> {
       @Override
       public FactoryBeanService getObject() throws Exception {
           return new FactoryBeanServiceImpl();
       }
   
       @Override
       public Class<?> getObjectType() {
           return FactoryBeanService.class;
       }
   
       @Override
       public boolean isSingleton() {
           return false;
       }
   }
   ```

   可以看出，`TestFactoryBean`类在每次`getObject`时都会创建一个新的对象，即多例模式（要注意，这里如果要实现单例模式，仅仅更改`isSingleton`返回`true`是没用的，必须在这个类里手动实现单例模式。）

3. 在applicationContext.xml里配置bean，这里配置的bean为`TestFactoryBean`类

   ```xml
   <?xml version="1.0" encoding="UTF-8"?>
   <beans xmlns="http://www.springframework.org/schema/beans"
          xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
          xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">
       <!-- Factory bean definition -->
       <bean id="testFactoryBean" class="com.learn.spring.chcp2.pojo.TestFactoryBean" />
   </beans>
   ```

   这里可以看出，实际上`FactoryBeanService`接口的实现类并没有**显示的**作为Bean被装配到IoC容器中，但是用于生产该实例的工厂被装配了。

4. 测试

   ```java
   public class FactoryBeanTest {
         @Test
       public void getFactoryBeanObj() {
           ApplicationContext context =
                   new ClassPathXmlApplicationContext("classpath:config/applicationContext.xml");
           FactoryBeanService factoryBean = context.getBean(FactoryBeanService.class);
           // 由于FactoryBean的缘故，上面的写法和下面这种等价
           // context.getBean("testFactoryBean");
           Assert.assertNotNull(factoryBean);
           factoryBean.testFactoryBean();
       }
   }
   ```

   实际上上述测试代码通过并且没有报错，正确输出了`FactoryBeanServiceImpl`类中`testFactoryBean`方法的"This is a factory bean test method"，原理分析见后文。

   而如果要获取`TestFactoryBean`对象，那么可以使用以下代码：

   ```java
   TestFactoryBean testFactoryBean = context.getBean("&testFactoryBean");
   ```

   

## BeanFactory的用法

这里区别于使用`ApplicationContext`，后者内部封装了一系列的容器加载、Bean读取、注册流程，这里直接使用`BeanDefinition`的默认实现类`DefaultListableBeanDefinition`以及`XmlBeanDefinitionReader`来读取Bean并注册到容器中：

```java
ClassPathResource resource = new ClassPathResource("config/test.xml");
DefaultListableBeanFactory factory = new DefaultListableBeanFactory();
XmlBeanDefinitionReader reader = new XmlBeanDefinitionReader(factory);
reader.loadBeanDefinitions(resource);
```

上述代码演示了Bean加载的三个步骤：

1. 定义一个输入IO，Spring中以`Resource`接口及其实现类来代表IO操作，常用的有`ClassPathResource`和`FileSystemResource`
2. 定义一个IoC容器，默认使用的是`DefaultListableBeanFactory`（注意即使是`ApplicationContext`接口的实现类也默认使用这个容器，它在继承层次中的`AbstractRefreshableBeanFactory`中被定义，并被后续子类使用
3. 定义一个读取器用于读取并注册Bean，这里用到的是`XmlBeanDefinitionReader`（内部的注册原理见下文）



## IoC容器的初始化过程

我们首先写一个测试方法：

```java
public class BeanDefinitionTest {
    @Test
    public void getEvanBean() {
        ApplicationContext context =
                new ClassPathXmlApplicationContext("classpath:config/applicationContext.xml");
        SimpleObject evanBean = context.getBean("evan", SimpleObject.class);
        Assert.assertNotNull(evanBean);
    }
}
```

我们深入到`ClassPathXmlApplicationContext`的构造函数中(`FileSystemXmlApplicationContext`同理)，可以找到最终调用的构造函数为：

```java
public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {
  
		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
}
```

其中refresh方法就启动了IoC容器的初始化，这个初始化过程包括以下三个方面：

1. BeanDefiniton的Resource的定位。这个Resource的定位指的是BeanDefinition资源的定位（比如说对应xml文件），它是由统一的`ResourceLoader`接口及其实现类通过统一的`Resource`接口实现的（前面说过`Resource`接口代表了Spring中的资源文件IO，比如类路径下的文件资源就会使用`ClassPathResource`接口进行抽象。
2. BeanDefinition的载入。这个载入过程是把用户定义好的Bean表示成IoC容器内部的数据结构，容器内部的数据结构即`BeanDefinition`类。
3. 向IoC容器注册BeanDefinition。这个注册过程是通过调用`BeanDefinitionRegistry`接口的实现来完成的。这个注册过程把载入过程中解析得到的`BeanDefinition`向IoC容器进行注册。

**注意**：以上过程为IoC容器的初始化过程，这个过程中一般不包含Bean依赖注入（DI）的实现，也就是说IoC初始化和DI是分开的两个过程，DI一般发生在应用第一次通过`getBean`方法向容器索取Bean的时候（但是可以通过设置`lazy-init`属性来改变这一点）



### BeanDefinition的Resource定位

手动编程时，我们首先会定义一个`Resource`类的实现（如上文定义的`ClassPathResource`），但是这个定义的Resource并不能被容器直接使用（容器即指`DefaultListableBeanFactory`）。这是因为IoC容器如`DefaultListableBeanFactory`只是一个纯粹的IoC容器，需要为它配置特定的读取器才能完成读取功能。但是对于`ApplicationContext`类来说，从上文的继承层次图可以看出，它继承了`ResourceLoader`接口，所以它们天生具有读取功能，下图为`ClassPathXmlApplicationContext`类的继承层次图：

![ClassPathXmlApplicationContext-Hierachy](/images/Spring/Spring IoC/ClassPathXmlApplicationContext-Hierachy.png)

可以看出`ClassPathXmlApplicationContext`继承链上包含了`DefaultResourceLoader`接口，实际上它也是利用这个接口完成的资源定位和加载

这里可能会产生一个疑问，作为父类的`DefaultResourceLoader`是怎么感知到子类使用了ClassPath而不是FileSystem路径的，其实很简单，只要子类通过覆写父类的某些接口就可以实现。但是仔细看`ClassPathXmlApplicationContext`类发现它并没有覆写`DefaultResourceLoader`类的任何方法，这是由于Spring中默认的加载方法就是按类路径加载，这里可以看`DefaultResourceLoader`源码中的`getResourceByPath`方法，它实际返回的就是一个`ClassPathResource`对象：

```java
public class DefaultResourceLoader implements ResourceLoader {
  @Override
	public Resource getResource(String location) {
		Assert.notNull(location, "Location must not be null");

		for (ProtocolResolver protocolResolver : getProtocolResolvers()) {
			Resource resource = protocolResolver.resolve(location, this);
			if (resource != null) {
				return resource;
			}
		}

		if (location.startsWith("/")) {
			return getResourceByPath(location);
		}
		else if (location.startsWith(CLASSPATH_URL_PREFIX)) {
      // 默认解析行为
			return new ClassPathResource(location.substring(CLASSPATH_URL_PREFIX.length()), getClassLoader());
		}
		else {
			try {
				// 尝试使用URL的方式解析
				URL url = new URL(location);
				return (ResourceUtils.isFileURL(url) ? new FileUrlResource(url) : new UrlResource(url));
			}
			catch (MalformedURLException ex) {
				// 上述解析方式都不起作用就交给最后的解析方法，要么是默认的getResourceByPath要么是子类覆写的getResourceByPath方法
				return getResourceByPath(location);
			}
		}
	}
  
  protected Resource getResourceByPath(String path) {
    // 默认的解析方法还是ClassPath解析，如果要改变这种默认行为，只要子类覆写这个方法即可
		return new ClassPathContextResource(path, getClassLoader());
	}
}
```

实际上由于默认使用了ClassPath，所以`DefaultResourceLoader`默认会按照ClassPath解析，这也就解释了为什么`DefaultResourceLoader`类中的`getResourceByPath`依然按照ClassPath解析的原因，如果遇到其它解析方法只要子类覆盖这个方法就可以改变解析方式。

下面给出了另一个`ApplicationContext`的实现类`FileSystemXmlApplicationContext`的实现，其中就包含了覆写`getResourceByPath`的代码实现自定义解析：

```java
public class FileSystemXmlApplicationContext extends AbstractXmlApplicationContext {
  @Override
	protected Resource getResourceByPath(String path) {
		if (path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}
}
```

上面覆写的`getResourceByPath`方法就会在`DefaultResourceLoader`中被使用。

#### Resource加载时序图

![ParticialSD-BeanDefinition-Resource](/images/Spring/Spring IoC/ParticialSD-BeanDefinition-Resource.png)

上图即为resource加载的时序图（部分），可以看到在构造`ClassPathXmlApplicationContext`时调用了`refresh`方法来启动整个调用，使用的IoC容器是`DefaultListableBeanFactory`,具体的资源在`AbstractBeanDefinitionReader`中被读入解析。

### BeanDefinition的载入和解析

对IoC容器来说，这个载入过程，相当于把定义的BeanDefinition在IoC容器中转化成一个Spring内部表示的数据结构的过程。IoC容器对Bean的管理和依赖注入功能的实现，是通过对其持有的BeanDefinition进行各种相关操作来完成的。这些BeanDefinition数据在IoC容器中通过一个HashMap来保持和维护。

由上文的分析可以知道，对`ApplicationContext`的实现类而言，Bean加载的入口在其构造函数的refresh方法中。而refresh方法是在抽象类`AbstractApplicationContext`中定义和实现的，其源码为：

```java
public abstract class AbastractApplicationContext extends DefaultResourceLoader
		implements ConfigurableApplicationContext {
  @Override
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			prepareRefresh();
			// 告诉子类刷新其内部的beanFactory，实际起作用的是其中的refreshBeanFactory()方法
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
			// Prepare the bean factory for use in this context.
			prepareBeanFactory(beanFactory);
			try {
				// 设置BeanFactory的后置处理
				postProcessBeanFactory(beanFactory);
				// 调用BeanFactory的后置处理器，这些后置处理器是在Bean定义中向容器注册的
				invokeBeanFactoryPostProcessors(beanFactory);
				// 注册Bean的后置处理器，在Bean的创建过程调用
				registerBeanPostProcessors(beanFactory);
				// 对上下文中的信息源进行初始化
				initMessageSource();
				// 初始化上下文的事件机制
				initApplicationEventMulticaster();
				// 初始化其他的特殊Bean
				onRefresh();
				// 检查监听Bean并且将这些Bean向容器注册
				registerListeners();
				// 实例化所有非懒加载的实例
				finishBeanFactoryInitialization(beanFactory);
				// 发布容器事件，结束refresh过程
				finishRefresh();
			}
			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}
				// 销毁已经创建的Bean，防止资源占用
				destroyBeans();
				// 重置'active'标志.
				cancelRefresh(ex);
				// 异常传播
				throw ex;
			}
			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
}
```

由以上代码可知，实际上这里使用了模板设计模式，如果想要更改哪一部分的行为只要覆写这个方法即可。上述源代码中最核心的代码就是：

```java
ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();
```

这段代码实际上包含IoC容器的创建，Bean的读取和注册等所有步骤。而obtainFreshBeanFactory中最核心的方法就是refreshBeanFactory了，这个方法在`AbstractApplicationContext`中没有实现，而是在它的子类`AbstractRefreshableApplicationContext`中实现了这个方法，源代码如下：

```java
public abstract class AbstractRefreshableApplicationContext extends AbstractApplicationContext {
	@Override
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
      // 如果已经有一个BeanFactory了，那么刷新时会销毁所有Bean和原有的Factory，重新读取Bean并创建Factory
			destroyBeans();
			closeBeanFactory();
		}
		try {
      // 创建一个新的IoC容器
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			beanFactory.setSerializationId(getId());
			customizeBeanFactory(beanFactory);
      // 读取Bean并载入、注册（核心方法）
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
}
```

上述代码的核心就在于loadBeanDefinitions函数，它负责读取定义Bean的文件，并且解析定义然后载入容器并注册。但是我们同样知道，Bean的定义形式五花八门，读取方法也会根据定义形式的不同而发生变化，就比如我们之前用的`ClassPathXmlApplicationContext`，Bean显然是定义在XML文件中的，所以这里读取Bean的时候就是从XML文件中读取，因此这里的loadBeanDefinitions函数为抽象函数，如果是以XML形式给出的Bean定义，那么它的实现由子类`AbstractXmlApplicationContext`给出。源代码如下：

```java
public abstract class AbstractXmlApplicationContext extends AbstractRefreshableConfigApplicationContext {
  @Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 创建一个XML读取器并和BeanFactory绑定（这里和我们之前手动读取的步骤一致）
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
    // 设置读取器的上下文信息
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    // 模板模式：允许子类覆写初始化和实际读取的过程
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}
  
  protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
}
```

第一个方法实际上初始化了一个XML读取器，利用读取器来读取XML文件。第二个方法根据提供的资源不同调用读取器的不同方法，如果提供的是Resourse，那么说明Spring IO已经明确，直接利用Resource就能获取到实际IO读取文件了；反之如果给到的是Location，那么说明具体路径和IO还没有被解析，所以会先进行IO解析得到Resource对象，再通过Resource对象读取文件。

在初始化`ClassPathXmlApplicationContext`的过程中是通过调用IoC容器的refresh方法来启动整个BeanDefinition的载入过程的，这个初始化是通过定义的`XmlBeanDefinitionReader`来完成的。同时我们也知道实际使用的IoC容器是`DefaultListableBeanFactory`，具体的Resource载入在`XmlBeanDefinitionReader`读入BeanDefinition时实现。因为Spring可以对应不同形式的BeanDefinition。由于这里使用的是XML方式的定义，所以需要使用`XmlBeanDefinitionReader`。如果使用了其他的BeanDefinition方式，就需要使用其它种类的BeanDefinitionReader来完成数据载入工作。在`AbstractXmlApplicationContext`的实现中可以看到，是在`reader.loadBeanDefinitions`中开始进行BeanDefinition的载入的。

在这里我们直接分析Location形式的loadBeanDefinitions方法，因为这个方法内部最终也会转化为使用Resource形式的loadBeanDefinitions。上述loadBeanDefinitions方法都是在`AbstractBeanDefinitionReader`中实现的，源代码如下：

```java
public abstract class AbstractBeanDefinitionReader implements BeanDefinitionReader, EnvironmentCapable {
  // 以location数组作为参数输入
  @Override
	public int loadBeanDefinitions(String... locations) throws BeanDefinitionStoreException {
		Assert.notNull(locations, "Location array must not be null");
		int count = 0;
		for (String location : locations) {
			count += loadBeanDefinitions(location);
		}
		return count;
	}
  
  // 解析单个location
  @Override
	public int loadBeanDefinitions(String location) throws BeanDefinitionStoreException {
		return loadBeanDefinitions(location, null);
	}
  
  // 实际解析location，第二个参数Resource保存了已经加载的所有Resource的集合
  public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		if (resourceLoader == null) {
			throw new BeanDefinitionStoreException(
					"Cannot load bean definitions from location [" + location + "]: no ResourceLoader available");
		}

		if (resourceLoader instanceof ResourcePatternResolver) {
			// 通配符Resource解析
			try {
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
        // 实际通过这个方法转换为Resource的形式读取
				int count = loadBeanDefinitions(resources);
				if (actualResources != null) {
					Collections.addAll(actualResources, resources);
				}
				if (logger.isTraceEnabled()) {
					logger.trace("Loaded " + count + " bean definitions from location pattern [" + location + "]");
				}
				return count;
			}
			catch (IOException ex) {
				throw new BeanDefinitionStoreException(
						"Could not resolve bean definition resource pattern [" + location + "]", ex);
			}
		}
		else {
			// 单个Resource解析，以绝对路径URL的形式
			Resource resource = resourceLoader.getResource(location);
      // 实际通过这个方法转换为Resource的形式读取
			int count = loadBeanDefinitions(resource);
			if (actualResources != null) {
				actualResources.add(resource);
			}
			if (logger.isTraceEnabled()) {
				logger.trace("Loaded " + count + " bean definitions from location [" + location + "]");
			}
			return count;
		}
	}
  
  // 以Resource数组的形式作为输入
  @Override
	public int loadBeanDefinitions(Resource... resources) throws BeanDefinitionStoreException {
		Assert.notNull(resources, "Resource array must not be null");
		int count = 0;
		for (Resource resource : resources) {
      // 实际上是通过调用单个Resource的形式读取，这个函数是一个抽象函数，会由具体的子类实现
			count += loadBeanDefinitions(resource);
		}
		return count;
	}
  
  
}
```

上面的代码按顺序就是loadBeanDefinitions从输入String类型的Pattern到最终能够被解析的Resource之间的调用过程，最后一个`loadBeanDefinitions(Resource)`方法会交给具体子类来实现，这里由于Resource代表的是一个XML文件的IO流，所以实际实现的子类是`XmlBeanDefinitionReader`。具体读取过程的源代码如下：

```java
public class XmlBeanDefinitionReader extends AbstractBeanDefinitionReader {
  @Override
	public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    // 读取调用入口
		return loadBeanDefinitions(new EncodedResource(resource));
	}
  
  public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Loading XML bean definitions from " + encodedResource);
		}

		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
      // 得到XML文件及其IO流准备进行读取
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
        // 具体读取的入口函数
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}
  
  protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
    // 这个函数是具体的读取函数，从特定的XML文件中载入BeanDefinition
		try {
      // 这里Document对象只是通用解析XML文件得到的一个文档树对象，并不是已经解析之后的XML文件对象
			Document doc = doLoadDocument(inputSource, resource);
      // 这个函数是实际开始解析BeanDefinition的入口
			int count = registerBeanDefinitions(doc, resource);
			if (logger.isDebugEnabled()) {
				logger.debug("Loaded " + count + " bean definitions from " + resource);
			}
			return count;
		}
		catch (BeanDefinitionStoreException ex) {
			throw ex;
		}
		catch (SAXParseException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"Line " + ex.getLineNumber() + " in XML document from " + resource + " is invalid", ex);
		}
		catch (SAXException ex) {
			throw new XmlBeanDefinitionStoreException(resource.getDescription(),
					"XML document from " + resource + " is invalid", ex);
		}
		catch (ParserConfigurationException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Parser configuration exception parsing XML from " + resource, ex);
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"IOException parsing XML document from " + resource, ex);
		}
		catch (Throwable ex) {
			throw new BeanDefinitionStoreException(resource.getDescription(),
					"Unexpected exception parsing XML document from " + resource, ex);
		}
	}
  
  public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // 实际类型是DefaultBeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		int countBefore = getRegistry().getBeanDefinitionCount();
    // 具体的解析过程交给了BeanDefinitionDocumentReader
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
}
```

以上代码的整个执行逻辑还是比较清晰的，首先交由这个类处理的Resource代表了定义Bean的XML IO流，首先就从这个抽象的IO流中获取实际的IO即`InputStream`；接下来对XML文档进行通用解析，目的是获取XML文档的文档树，这个解析过程和Bean的解析无关，只是单纯的构造XML文档树；最后将读取的XML文档树即Document对象传递给实际的读取类`BeanDefinitionDocumentReader`进行Bean的解析，所有解析过程都在`DefaultBeanDefinitionDocumentReader`类中实现，这个类的处理结果由`BeanDefinitionHolder`对象持有（包括`BeanDefintion`对象以及其他与`BeanDefinition`的使用相关的信息，如Bean的名字、别名的集合等等），`BeanDefinitionHolder`对象的生成实际上是由`DefaultBeanDefinitionDocumentReader`委托给`BeanDefinitionParserDelegate`来实现的，具体代码如下：

```java
public class DefaultBeanDefinitionDocumentReader implements BeanDefinitionDocumentReader {
  @Override
	public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		doRegisterBeanDefinitions(doc.getDocumentElement());
	}
  
  protected void doRegisterBeanDefinitions(Element root) {
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);

		if (this.delegate.isDefaultNamespace(root)) {
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isDebugEnabled()) {
						logger.debug("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}

		preProcessXml(root);
    // 具体是由这个方法开始解析过程的
		parseBeanDefinitions(root, this.delegate);
		postProcessXml(root);
    
		this.delegate = parent;
	}
  
  protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						parseDefaultElement(ele, delegate);
					}
					else {
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
  
  private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
      // 这里触发Bean的解析
			processBeanDefinition(ele, delegate);
		}
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
  
  protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // 委托给BeanDefinitionParserDelegate来实际解析
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// 注册Bean
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
}
```

上述代码展现的就是Bean的实际解析过程了，具体的解析代码细节不太需要了解，无非就是读取bean标签的各种属性以及子标签，最后得到表示Bean的数据结构`BeanDefinitionHolder`，这个过程还是比较清晰的，而具体的解析过程可以参看`BeanDefinitionReaderDelegate`类中的`parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean)`方法。





## Bean依赖注入过程

**重点：所有Bean的依赖注入都是由`getBean`方法开始的**

### Non-Lazy-Init的Bean的注入

非懒加载的Bean，依赖注入实际发生在`AbstractApplicationContext`类中的`refresh`方法中的`finishBeanFactoryInitialization(beanFactory);`中。这个函数实际上通过调用已经读取为BeanDefinition形式的Bean的BeanName实现加载，具体代码如下：

```java
protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
    ...
		// Instantiate all remaining (non-lazy-init) singletons.
		beanFactory.preInstantiateSingletons();
}
```

实际上就通过调用传入的BeanFactory中的`preInstantiateSingletons`方法实现了非懒加载单例的预加载。这里的`ConfigurableListableBeanFactory`类参数beanFactory在Spring中实际上是使用了其默认实现`DefaultListableBeanFactory`，所以可以查看`DefaultListableBeanFactory`类中的`preInstantiateSingletons`方法（代码经过删减）：

```java
public void preInstantiateSingletons() throws BeansException {
		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
      // 获取BeanDefinition
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
        // 处理FactoryBean
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
        // 非FactoryBean则直接加载
				else {
					getBean(beanName);
				}
			}
		}

		// 调用Bean的后置处理器
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```

**而Lazy-Init Bean的注入则发生在首次调用getBean方法时**

### getBean方法

getBean方法是一切依赖注入的入口方法，通过调用getBean方法实现了所有的依赖调用过程。getBean方法是由`BeanFactory`接口定义的。所以包括`ApplicationContext`接口在内的所有子类都有`getBean`方法，但是这些类实际调用的是`AbstractBeanFactory`中的`getBean`方法实现的注入，例如比较常用的一个重载方法如下：

```java
@Override
public Object getBean(String name) throws BeansException {
	return doGetBean(name, null, null, false);
}
```

#### doGetBean方法

可以看到实际调用的是`doGetBean`方法，下面给出`doGetBean`方法源码：

```java
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		final String beanName = transformedBeanName(name);
		Object bean;

		// 检查当前bean是否已经被创建过了，如果被创建过了则跳过
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isTraceEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.trace("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.trace("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
      // 后面会解释getObjectForBeanInstance方法的作用
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// 先检查父Factory，如果父Factory
			BeanFactory parentBeanFactory = getParentBeanFactory();
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else if (requiredType != null) {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
				else {
					return (T) parentBeanFactory.getBean(nameToLookup);
				}
			}

			if (!typeCheckOnly) {
        // 标记Bean为创建状态
				markBeanAsCreated(beanName);
			}

			try {
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// 创建所有依赖Bean
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// 单例模式创建Bean
				if (mbd.isSingleton()) {
          // 实际发生创建的入口，getSingleton方法是一个很重要的方法，第二个参数为ObjectFactory类型，这里用于创建Bean
          // 即createBean方法实际创建了Bean
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// 多例模式则不经过单例检查直接创建Bean
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
          // 处理其它作用域的情况
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// 最后进行类型检查和类型转换
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isTraceEnabled()) {
					logger.trace("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
```

#### 三级缓存

上述代码特别长，其中重点也很多，我们按顺序分析。首先先分析单例模式下的核心函数`getSingleton`，`getSingleton`函数定义在`SingletonBeanRegistry`类中，`DefaultSingletonBeanRegistry`是其默认实现类，它作用就是用来存储所有单例Bean，和负责单例Bean的生产相关工作。

这个类中有三个非常重要的属性，即三级缓存，用来解决循环依赖问题：

```java
/** Cache of singleton objects: bean name to bean instance. */
private final Map<String, Object> singletonObjects = new ConcurrentHashMap<>(256);

/** Cache of singleton factories: bean name to ObjectFactory. */
private final Map<String, ObjectFactory<?>> singletonFactories = new HashMap<>(16);

/** Cache of early singleton objects: bean name to bean instance. */
private final Map<String, Object> earlySingletonObjects = new HashMap<>(16);
```

其中`singletonObjects`储存了所有创建完毕的Bean对象；`singletonFactories`则保存了Bean名到其创建方法之间的映射，其目的是用于保存中途创建一半的对象，比如A对象依赖B对象，B对象依赖A对象，那在A对象创建时发现需要B对象就会转而创建B对象，此时这个未完成创建的A对象就会被放入`singletonFactories`中进行保存，之后会B对象发现依赖A对象之后会从`singletonFactories`中取出对应的ObjectFactory，并获取A对象，然后将A对象放入`earlySingletonObjects`中表示已经前期引用。

#### getSingleton方法

然后我们再来关注核心方法`getSingleton`:

```java
public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		synchronized (this.singletonObjects) {
      // 尝试获取单例对象，如果获取成功则直接返回这个对象，否则会开启创建流程
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
        // 创建前回调函数
				beforeSingletonCreation(beanName);
        // 标记这个单例是否是新创建的
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
          // 实际调用创建函数在这里, singletonFactory是一个ObjectFactory类型参数，用于生产对应的Bean
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
          // 单例创建后置回调函数
					afterSingletonCreation(beanName);
				}
				if (newSingleton) {
          // 如过是一个新的单例，那么将其加入到singletonObjects内储存，至此单例创建完毕
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

上面就是整个单例Bean创建的全过程，总结一下就是：1. 首先尝试从已创建对象集合中获取，如果获取成功则直接返回，否则开始创建；2. 调用对象工厂函数创建Bean；3. 将创建好的对象保存。实际创建Bean的调用点即为`singletonObject = singletonFactory.getObject();`而singletonFactory是一个ObjectFactory类型的对象，它以Lambda表达式的形式在`doGetBean`方法中给出了:

```java
sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							destroySingleton(beanName);
							throw ex;
						}
					});
```

#### createBean方法

所以我们直接分析这里给出的`createBean`函数，`createBean`函数是`AbstractBeanFactory`类定义的一个抽象函数，它的实现在`AbstractAutowireCapableBeanFactory`中：

```java
protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {
		RootBeanDefinition mbdToUse = mbd;
    // 确保当前要加载的Bean的Class是已知的，如果是动态加载的Class则需要复制定义以防止覆盖
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// 创建前置操作
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// 由BeanPostProcessors代理生成Bean实例
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
      // 实际创建入口
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isTraceEnabled()) {
				logger.trace("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

#### doCreateBean方法

再来看`doCreateBean`函数： 

```java
protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// 所有Bean都由BeanWrapper持有
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
      // 如果是单例那么移出缓存内的Bean实例
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
      // 创建一个Bean实例
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
      // 设置BeanDefinition的实际类型
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
      // 如果允许早期引用EarlyReference，那么将其加入到缓存中，注意此时加入的Bean因为没有设置任何属性，所以
      // 不存在任何循环引用问题，Lambda表达式中的bean就是当前已经创建的但是所有属性都为空的Bean实例
      // 就是通过引用这种早期对象来破解循环引用问题，后续每个对象属性被装配后，所有引用就都正确了
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// 初始化Bean实例
		Object exposedObject = bean;
		try {
      // 装配Bean的属性
			populateBean(beanName, mbd, instanceWrapper);
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
      // 对早期暴露对象的处理
			Object earlySingletonReference = getSingleton(beanName, false);
			if (earlySingletonReference != null) {
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// 注册Bean
		try {
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```

#### createBeanInstance方法

还是按顺序，先看对象实例的创建过程即`createBeanInstance`函数：

```java
protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		Class<?> beanClass = resolveBeanClass(mbd, beanName);
		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}
    // 如果BeanDefinition中给定了Supplier用于创建对象那就使用Supplier
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		// 如果BeanDefinition中给定了工厂方法那么就调用工厂方法创建对象
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// 如果这个对象已经被创建过了，那么相应的构造函数信息会被保存在BeanDefinition中，这里优先考虑复用
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
        // 根据参数个数调用对应构造函数
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
        // 使用默认无参构造函数
				return instantiateBean(beanName, mbd);
			}
		}

		// 寻找对应的构造函数
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
    // 解析的构造器不为空 || 注入类型为构造函数自动注入 || bean定义中有构造器参数 || 传入参数不为空
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// Preferred constructors for default construction?
		ctors = mbd.getPreferredConstructors();
		if (ctors != null) {
			return autowireConstructor(beanName, mbd, ctors, null);
		}

		// 没有适配的构造函数则使用默认构造函数
		return instantiateBean(beanName, mbd);
	}
```

重点关注`instantiateBean`函数即可，无需关注构造器自动注入的过程。这里不列出`instantiateBean`的源码，只要知道它默认是使用CGLIB来创建对象的就可以了，当然也支持反射创建对象。创建好的对象以BeanWrapper的方式返回。这样就完成了Bean的实例化。

#### Bean早期暴露及再谈三级缓存

接下来我们关注`doCreateBean`函数中有关early exposure的内容，源码片段为：

```java
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}
```

这里还是需要讨论Spring的三层缓存，因为早期引用的目的就是为了解决循环依赖，我们之前分析过，已经完成创建的单例对象（无论是否使用了早期引用对象）都会被放置在`singletonObjects`中，而允许早期引用的对象在被实例化但未装配属性之前，会以ObjectFactory的形式保存在`singletonFactories`中，而早期引用对象被首次使用后则将对应的ObjectFactory移除，并将实例加入到`earlySingletonObjects` 中，最终就是通过判断`earlySingletonObjects` 中是否存在对应的对象来判断是否发生早期引用。

可以看到上面一段代码中，addSingletonFactory函数参数的第二个参数就是对应的ObjectFactory，而其中的getEarlyBeanReference函数的作用只是为了同时持有BeanDefiniton和当前创建的Bean实例，并允许后置处理器对Bean实例进行处理而设计的，具体代码不进行分析了；而addSingletonFactory方法则就是实现了关于三层缓存的保存问题，代码比较简单，具体不分析了：

```java
protected void addSingletonFactory(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(singletonFactory, "Singleton factory must not be null");
		synchronized (this.singletonObjects) {
			if (!this.singletonObjects.containsKey(beanName)) {
				this.singletonFactories.put(beanName, singletonFactory);
				this.earlySingletonObjects.remove(beanName);
				this.registeredSingletons.add(beanName);
			}
		}
	}
```

#### populateBean方法

接下来`doCreateBean`方法要做的最重要的一件事就是装配属性，即以下这行代码：

```java
populateBean(beanName, mbd, instanceWrapper);
```

直接看populateBean方法：

```java
protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
    // 空实例的处理
		if (bw == null) {
			if (mbd.hasPropertyValues()) {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
			}
			else {
				return;
			}
		}

		// Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
		// state of the bean before properties are set. This can be used, for example,
		// to support styles of field injection.
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
						return;
					}
				}
			}
		}
		
    // 实际上指的是xml文件定义的property标签的所有属性，是string->string形式的
		PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

    // 自动装配
		int resolvedAutowireMode = mbd.getResolvedAutowireMode();
		if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
			MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
			// Add property values based on autowire by name if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
				autowireByName(beanName, mbd, bw, newPvs);
			}
			// Add property values based on autowire by type if applicable.
			if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
				autowireByType(beanName, mbd, bw, newPvs);
			}
			pvs = newPvs;
		}

		boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
		boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

		PropertyDescriptor[] filteredPds = null;
    // 初始化处理器处理过程
		if (hasInstAwareBpps) {
			if (pvs == null) {
				pvs = mbd.getPropertyValues();
			}
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof InstantiationAwareBeanPostProcessor) {
					InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
					PropertyValues pvsToUse = ibp.postProcessProperties(pvs, bw.getWrappedInstance(), beanName);
					if (pvsToUse == null) {
						if (filteredPds == null) {
							filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
						}
						pvsToUse = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
						if (pvsToUse == null) {
							return;
						}
					}
					pvs = pvsToUse;
				}
			}
		}
    // 需要依赖检查
		if (needsDepCheck) {
			if (filteredPds == null) {
				filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
			}
			checkDependencies(beanName, mbd, filteredPds, pvs);
		}

		if (pvs != null) {
      // 核心：加载属性值
			applyPropertyValues(beanName, mbd, bw, pvs);
		}
	}
```

#### applyPropertyValues方法

接着看applyPropertyValues方法：

```java
protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
		if (pvs.isEmpty()) {
			return;
		}

		if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
			((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
		}

		MutablePropertyValues mpvs = null;
		List<PropertyValue> original;

		if (pvs instanceof MutablePropertyValues) {
			mpvs = (MutablePropertyValues) pvs;
			if (mpvs.isConverted()) {
				// 对于已经解析过的属性则直接使用，不需要再次解析
				try {
					bw.setPropertyValues(mpvs);
					return;
				}
				catch (BeansException ex) {
					throw new BeanCreationException(
							mbd.getResourceDescription(), beanName, "Error setting property values", ex);
				}
			}
			original = mpvs.getPropertyValueList();
		}
		else {
			original = Arrays.asList(pvs.getPropertyValues());
		}

		TypeConverter converter = getCustomTypeConverter();
		if (converter == null) {
			converter = bw;
		}
    // 核心解析器
		BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

		// 创建关于待解析属性的深拷贝
		List<PropertyValue> deepCopy = new ArrayList<>(original.size());
		boolean resolveNecessary = false;
		for (PropertyValue pv : original) {
			if (pv.isConverted()) {
        // 已解析：直接保存
				deepCopy.add(pv);
			}
			else {
				String propertyName = pv.getName();
				Object originalValue = pv.getValue();
				if (originalValue == AutowiredPropertyMarker.INSTANCE) {
					Method writeMethod = bw.getPropertyDescriptor(propertyName).getWriteMethod();
					if (writeMethod == null) {
						throw new IllegalArgumentException("Autowire marker for property without write method: " + pv);
					}
					originalValue = new DependencyDescriptor(new MethodParameter(writeMethod, 0), true);
				}
        // 具体值类型交由resolver解析，这里略过，只要知道它能返回一个对应的包装类型即可
        // 包括对于引用其它Bean也是在这里实现的，由resolver去获取其它Bean，从而触发其它Bean的创建
				Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
				Object convertedValue = resolvedValue;
        // 检查值合法且属性可写
				boolean convertible = bw.isWritableProperty(propertyName) &&
						!PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
				if (convertible) {
					convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
				}
				// Possibly store converted value in merged bean definition,
				// in order to avoid re-conversion for every created bean instance.
				if (resolvedValue == originalValue) {
					if (convertible) {
            // 缓存已解析的属性
						pv.setConvertedValue(convertedValue);
					}
					deepCopy.add(pv);
				}
				else if (convertible && originalValue instanceof TypedStringValue &&
						!((TypedStringValue) originalValue).isDynamic() &&
						!(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
					pv.setConvertedValue(convertedValue);
					deepCopy.add(pv);
				}
				else {
					resolveNecessary = true;
					deepCopy.add(new PropertyValue(pv, convertedValue));
				}
			}
		}
		if (mpvs != null && !resolveNecessary) {
			mpvs.setConverted();
		}

		// Set our (possibly massaged) deep copy.
		try {
      // 利用反射机制具体设值，将属性值保存
			bw.setPropertyValues(new MutablePropertyValues(deepCopy));
		}
		catch (BeansException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Error setting property values", ex);
		}
	}
```

关于具体resolver解析过程和具体设值过程不做具体分析。



## Bean的生命周期

以下是Bean的生命周期完整示意图：

![Bean Lifecycle](/images/Spring/Spring IoC/Bean Lifecycle.jpeg)

Bean的生命周期可以通俗的概括为：

- Bean实例的创建
- 为Bean实例设置属性
- 调用Bean的初始化方法
- 应用通过IoC容器使用Bean
- 容器关闭时，调用Bean的销毁方法

上面分析过了，Spring容器加载的入口方法就是定义在`AbstractApplicationContext`中的`refresh`方法，源码可见**BeanDefinition的载入和解析**节的实现。refresh方法中调用的`finishBeanFactoryInitialization`方法则完成了Non-lazy-init Bean的初始化，其本质也是调用`getBean`方法实现的Bean实例创建和属性装配，这在上面已经谈过了。我们从上面的图中可以看到，Bean具体生命周期从属性注入结束后开始，在谈论`populateBean`时我们知道，`populateBean`方法完成了Bean的属性注入，而后续的生命周期则从`AbstractAutowireCapableBeanFactory`中的`initializeBean`方法开始。

#### initializeBean方法

`initializeBean`方法的源码如下：

```java
protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

我们可以看到关键方法包括`invokeAwareMethods`、`applyBeanPostProcessorsBeforeInitialization`、`invokeInitMethods`和`applyBeanPostProcessorsAfterInitialization`四个方法，我们逐一分析。

#### invokeAwareMethods方法

首先分析`invokeAwareMethods`方法，源码如下：

```java
private void invokeAwareMethods(final String beanName, final Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof BeanNameAware) {
				((BeanNameAware) bean).setBeanName(beanName);
			}
			if (bean instanceof BeanClassLoaderAware) {
				ClassLoader bcl = getBeanClassLoader();
				if (bcl != null) {
					((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
				}
			}
			if (bean instanceof BeanFactoryAware) {
				((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
			}
		}
	}
```

各种Aware方法实际上就干一件事，就是如果Bean对象需要获取容器加载Bean后的BeanName、ClassLoader以及BeanFactory对象的话，只要实现对应的`BeanNameAware`、`BeanClassLoaderAware`和`BeanFactoryAware`接口即可，这样在这个函数里就会调用对应的set方法将BeanName等信息传递给Bean对象，供Bean对象使用。我们定义一个Bean对象：

```java
public class AllLifeCycleBean implements BeanNameAware, BeanFactoryAware, BeanClassLoaderAware,
        ApplicationContextAware, InitializingBean, DisposableBean {
    private String name;

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        System.out.printf("Set bean factory method called, Bean factory is %s\n", beanFactory.getClass().getName());
    }

    @Override
    public void setBeanName(String name) {
        System.out.printf("Set bean name method called, Bean name is %s\n", name);
    }

    @Override
    public void setBeanClassLoader(ClassLoader classLoader) {
        System.out.printf("Set classloader method called, Classloader is %s\n", classLoader.getClass().getName());
    }

    @Override
    public void destroy() throws Exception {
        System.out.println("Destroy bean method called");
    }

    @Override
    public void afterPropertiesSet() throws Exception {
        System.out.println("After properties set method called");
    }

    @Override
    public void setApplicationContext(ApplicationContext applicationContext) throws BeansException {
        System.out.printf("Set application context method called, Application context is %s\n", applicationContext.getClass().getName());
    }

    public void myInitMethod() {
        System.out.println("My init method is called");
    }

    public void myDestroyMethod() {
        System.out.println("My destroy method is called");
    }

    @PostConstruct
    public void postConstructMethod() {
        System.out.println("Post construct method is called");
    }

    @PreDestroy
    public void preDestroyMethod() {
        System.out.println("Pre destroy method is called");
    }
}
```

这里我们先关注`setBeanName`、`setBeanClassLoader`和`setBeanFactory`三个方法，这三个方法定义在上述三个Aware接口中，Bean通过继承接口即可使用这些属性。而这些属性就在Bean属性注入结束后由invokeAwareMethods调用，并且单例的话只会调用一次。

#### BeanPostProcessor

接下来我们关注`applyBeanPostProcessorsBeforeInitialization`和`applyBeanPostProcessorsAfterInitialization`这两个方法。这两个方法只有调用顺序的先后区别，它们调用的目标对象是注册到容器的、实现了`BeanPostProcessors`的对象，其源码如下：

```java
public interface BeanPostProcessor {
	@Nullable
	default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
	@Nullable
	default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
		return bean;
	}
}
```

```java
@Override
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}
```

`BeanPostProcessor`类提供了两个方法，一个作用在Bean init-method调用前，一个作用在Bean init-method调用后。所有BeanPostProcessor对象如果要起作用都必须以Bean的形式注册到容器，这样才能被使用。

**注意：这里要注意一点，如果我们想指定BeanPostProcessor作用的Bean对象，那么手动判断Bean的类型，默认情况下BeanPostProcessor会作用在所有Bean对象上**

例如有一个BeanPostProcessor的实现类`ApplicationContextAwareProcessor`，它的源码中只会作用在实现了某些接口（如`ApplicationContextAware`）的Bean对象上，这个过程是需要手动实现的，其源码如下：

```java
@Override
	@Nullable
	public Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
		if (!(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
				bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
				bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)){
			return bean;
		}

		AccessControlContext acc = null;

		if (System.getSecurityManager() != null) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}

		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
```

可以看到，源码一开始就判断了Bean是否继承规定的接口，如果没有继承则直接返回，否则才会继续执行，根据继承的接口不同来执行不同的方法。

**第二个注意点：注意@PostConstruct注解和@PreDestroy注解的实现就是依靠BeanPostProcessors，它们和构造函数与析构函数、init-method和destroy-method不同，是可以并存的，这里要注意区分，后面会讨论它们的执行顺序**

因为@PostConstruct注解和@PreDestroy注解是由JSR规范提供的而不是Spring，所以使用时需要添加依赖（SpringBoot不用）：

```xml
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>
```

同时在Spring MVC中要启用这两个注解（以及@Autowired）注解，那么还需要在beans的xml文件中配置：

首先，xml文件头要启用`xmlns:context`即`xmlns:context="http://www.springframework.org/schema/context"`；

同时在`xsi:schemaLocation`中添加：`http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context-4.1.xsd`；

最后启用注解：`<context:annotation-config/>`

这样上述三个注解（包括@Autowired）才会起作用。

可以调试`applyBeanPostProcessorsBeforeInitialization`方法，在其中的`getBeanPostProcessors()`方法中看到加载的所有BeanPostProcessors（这里忽略了将@PostConstruct和@PreDestroy注解解析为BeanPostProcessor的过程，实际上它是通过`InitDestroyAnnotationBeanPostProcessor`类完成解析的，会将两个注解对应的方法解析为`LifecycleMetadata`对象，其中保存了要调用的方法名，调用时即通过反射来调用）：

<img src="/images/Spring/Spring IoC/BeanPostProcessors.png" alt="BeanPostProcessors" style="zoom:50%;" />



**最后一个注意点：BeanPostProcessor是无序的，调用顺序是不可知的，例如上图中我们自定义的AllLifeCycleBeanPostProcessor的位置就是不可知的**

#### Bean的初始化

Bean的初始化和调用构造函数是不一样的，构造函数是由Spring负责调用的，而初始化又可以分为以下三种：继承`InitializingBean`接口，指定init-method以及使用@PostConstruct注解。

由于PostConstruct方法之前上一节已经说过，所以这里略过。我们首先先来看initializeBean方法中的`invokeInitMethods`方法和`InitializingBean`接口：

```java
public interface InitializingBean {
	void afterPropertiesSet() throws Exception;
}
```

```java
protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```

可以清楚的看到，`invokeInitMethods`函数中先调用了`InitializingBean`接口中的`afterPropertiesSet`方法，然后调用了init-method，所以这两个方法是不矛盾的，两者可以同时存在，但是在`InitializingBean`接口注释中也说到了，这两种方法实现的功能是一致的，都是在属性注入后执行一些自定义操作，二者选其一即可，当然两者都用也可以，唯一区别在于使用`InitializingBean`接口时，Bean类需要继承接口实现功能，而inti-method只要在xml中指定函数即可。

从上面的分析可以知道由于@PostConstruct注解是在BeanPostProcessor中处理的，所以它是最先触发的初始化方法，其次是继承`InitializingBean`接口的`afterPropertiesSet`方法，最后是init-method。

#### Bean的销毁

Bean的销毁过程和Bean的初始化过程基本类似，先给出结论：**@PreDestroy注解的方法 -> 实现DisposableBean接口继承的destroy方法 -> xml中定义的destroy-method**，至于为什么是这个顺序也不解释了，和初始化一致：@PreDestroy注解是由`postProcessBeforeDestruction`处理的，被最先执行；其次是如果Bean实现了`DisposableBean`接口，那么会调用其中的`destroy`方法；最后就是destroy-method了。

下面来分析Bean的销毁流程，Bean的销毁首先是由BeanFactory的容器关闭而导致的，如果在测试中没有关闭容器那么销毁生命周期不一定会被执行。BeanFactory的销毁方法被定义在`AbstractApplicationContext`里的`close`方法里了：

由于中间过程较多，所以只给出中间步骤的调用顺序和所在类:

 	1. `AbstractApplicationContext` -> `close`调用`doClose`方法
 	2. `AbstractApplicationContext` -> `doClose`调用`destroyBeans`方法
 	3. `AbstractApplicationContext` -> `destroyBeans`调用`destroySingletons`方法
 	4. `DefaultListableBeanFactory` -> `destroySingletons`调用`super.destroySingletons`方法
 	5. `DefaultSingletonBeanRegistry` -> `destroySingletons`调用`destroySingleton`方法
 	6. `DefaultSingletonBeanRegistry` -> `destroySingleton`调用`destroyBean`方法
 	7. `DefaultSingletonBeanRegistry` -> `destroyBean`调用`bean.destroy`方法

最终对Bean的实际销毁是在`DisposableBeanAdapter`类中的`destroy`，源码如下：

```java
public void destroy() {
		if (!CollectionUtils.isEmpty(this.beanPostProcessors)) {
			for (DestructionAwareBeanPostProcessor processor : this.beanPostProcessors) {
        // 后置处理器
				processor.postProcessBeforeDestruction(this.bean, this.beanName);
			}
		}

		if (this.invokeDisposableBean) {
			if (logger.isTraceEnabled()) {
				logger.trace("Invoking destroy() on bean with name '" + this.beanName + "'");
			}
			try {
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((DisposableBean) this.bean).destroy();
						return null;
					}, this.acc);
				}
				else {
          // 调用DisposableBean的接口
					((DisposableBean) this.bean).destroy();
				}
			}
			catch (Throwable ex) {
				String msg = "Invocation of destroy method failed on bean with name '" + this.beanName + "'";
				if (logger.isDebugEnabled()) {
					logger.warn(msg, ex);
				}
				else {
					logger.warn(msg + ": " + ex);
				}
			}
		}

		if (this.destroyMethod != null) {
      // 调用destroy-method
			invokeCustomDestroyMethod(this.destroyMethod);
		}
		else if (this.destroyMethodName != null) {
			Method methodToInvoke = determineDestroyMethod(this.destroyMethodName);
			if (methodToInvoke != null) {
				invokeCustomDestroyMethod(ClassUtils.getInterfaceMethodIfPossible(methodToInvoke));
			}
		}
	}
```

上述代码的结构和初始化过程几乎一致，这里不再过多分析。



## 其它方法

### getObjectForBeanInstance

这个方法用于从FactoryBean中获取实际要生产的Bean。源码定义在了`AbstractBeanFactory`类中：

```java
protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// 先判断是不是FactoryBean的间接引用或空引用，如果是要取工厂对象本身，那在这里就直接返回
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
			if (mbd != null) {
				mbd.isFactoryBean = true;
			}
			return beanInstance;
		}

		// 如果以&开头并且不是工厂，那么直接返回实例即可，不需要工厂生产
		if (!(beanInstance instanceof FactoryBean)) {
			return beanInstance;
		}

		Object object = null;
		if (mbd != null) {
			mbd.isFactoryBean = true;
		}
		else {
      // 如果对象曾经被创建过，那么从缓存中直接返回即可
			object = getCachedObjectForFactoryBean(beanName);
		}
		if (object == null) {
			// Return bean instance from factory.
			FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
			// Caches object obtained from FactoryBean if it is a singleton.
			if (mbd == null && containsBeanDefinition(beanName)) {
				mbd = getMergedLocalBeanDefinition(beanName);
			}
			boolean synthetic = (mbd != null && mbd.isSynthetic());
      // 实际使用工厂创建Bean的入口
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

从上述源码中可以看到实际上调用工厂创建Bean的方法入口为`getObjectFromFactoryBean`，这里对这个方法不做过多的分析，因为这个方法完成的事就是实例化Bean、属性注入以及实现Bean生命周期的一系列操作，上面都说过，所以这里不多做叙述。