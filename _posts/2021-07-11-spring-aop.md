---
layout: post
title: Spring AOP
categories: Spring
keywords: Spring AOP, Spring, AOP
---

## Spring AOP

AOP的作用很简单，就是让某些功能独立出来（专业点可以是解耦合），这些种独立和封装成方法或类的最大区别就是它对于原来的方法没有任何的侵入性，我们不需要知道原来的方法是什么样的就可对它进行一定的增强。设想一下，我们要对一个类进行增强时，除了直接修改代码这种方法也就只能使用适配器来实现了，适配器实现不能说不优雅，但是至少缺乏一定的灵活性，这种灵活性包括适配对象的灵活性、拦截粒度的灵活性。利用AOP可以非常简单的实现对某些方法、某些类的正则适配或其它适配规则，同时也预置了对方法发生前、发生后以及抛出异常三个角度的切入点。这也使得使用AOP要更加灵活高效。

### AOP三大基本概念

AOP的三个基本概念包括：切点（Pointcut）、切面（Advice，也可以叫通知）和通知器（Advisor）。其分工非常明确，我们要尝试对一个方法进行增强，那么一定要明确三件事：哪一个方法、做哪些增强、什么时候做。其中切点定义了哪一个方法、切面定义了做哪些增强和什么时候做、通知器将二者结合起来作为一个整体供Spring IoC整体调用。

我们先来看切面：

#### Advice

上面说过了，Aspect定义了做哪些增强和什么时候做这两件事，Aspect结构为了同时实现这两种功能采用了继承的层次结构来实现。先来给出完成的Aspect继承层级：

![Advice-Arch](
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring AOP/Advice-Arch.png)

从这个继承层次可以看出，`Advice`接口是最上层的接口，它有三个直接实现子类`BeforeAdvice`、`AfterAdvice`和`Interceptor`。其实这个逻辑不太好理解而且可能有一些怪异，在这里，`BeforeAdvice`和`AfterAdvice`是两种已经预定义好的运行时机（即解决了"什么时候做"这件事），如果需要使用这两种预设的时机只要实现`MethodBeforeAdvice`或`AfterReturningAdvice`或`ThrowsAdvice`三种接口即可。但是如果想要自定义运行时机，比如在开始和结束同时增强，那么就需要继承`MethodInterceptor`接口自己来实现这种运行逻辑了。

综上所述，`Aspect`类对于增强时机的控制是通过类结构层次来实现的（其实本质上就是使用了预定义的一些`MethodInterceptor`接口的实现类来实现的，比如`MethodBeforeAdviceInterceptor`、`ThrowsAdviceinterceptor`以及`AfterReturningAdviceInterceptor`），而要实现具体的增强功能实际上是通过实现接口来实现的，后面会分析这个流程。

我们先来看`Advice`、`BeforeAdvice`和`AfterAdvice`三个接口的源码，它们都是标记接口，没有任何代码：

```java
public interface Advice {
}

public interface BeforeAdvice extends Advice {
}

public interface AfterAdvice extends Advice {
}
```

而实际定义了功能的`MethodBeforeAdvice`和`AfterReturningAdvice`接口如下：

```java
public interface MethodBeforeAdvice extends BeforeAdvice {
	void before(Method method, Object[] args, @Nullable Object target) throws Throwable;
}

public interface AfterReturningAdvice extends AfterAdvice {
	void afterReturning(@Nullable Object returnValue, Method method, Object[] args, @Nullable Object target) throws Throwable;
}
```

上述两个接口的实现很简单，就是定义了两个接口，每个接口的方法也很清楚了，不多讲，这个两个接口主要就是为了后续的`MethodBeforeAdviceInterceptor`和`AfterReturningAdviceInterceptor`调用的，这里只负责解决"做哪些增强"这件事，至于在什么时候做其实委托给了`MethodInterceptor`的实现类去做。

而`ThrowsAdvice`比较特殊，它也是一个空接口，没有定义任何方法：

```java
/**
 * Tag interface for throws advice.
 *
 * <p>There are not any methods on this interface, as methods are invoked by
 * reflection. Implementing classes must implement methods of the form:
 *
 * <pre class="code">void afterThrowing([Method, args, target], ThrowableSubclass);</pre>
 *
 * <p>Some examples of valid methods would be:
 *
 * <pre class="code">public void afterThrowing(Exception ex)</pre>
 * <pre class="code">public void afterThrowing(RemoteException)</pre>
 * <pre class="code">public void afterThrowing(Method method, Object[] args, Object target, Exception ex)</pre>
 * <pre class="code">public void afterThrowing(Method method, Object[] args, Object target, ServletException ex)</pre>
 *
 * The first three arguments are optional, and only useful if we want further
 * information about the joinpoint, as in AspectJ <b>after-throwing</b> advice.
 *
 * <p><b>Note:</b> If a throws-advice method throws an exception itself, it will
 * override the original exception (i.e. change the exception thrown to the user).
 * The overriding exception will typically be a RuntimeException; this is compatible
 * with any method signature. However, if a throws-advice method throws a checked
 * exception, it will have to match the declared exceptions of the target method
 * and is hence to some degree coupled to specific target method signatures.
 * <b>Do not throw an undeclared checked exception that is incompatible with
 * the target method's signature!</b>
 *
 * @author Rod Johnson
 * @author Juergen Hoeller
 * @see AfterReturningAdvice
 * @see MethodBeforeAdvice
 */
public interface ThrowsAdvice extends AfterAdvice {
}
```

但是它的注释写的很清楚，定义的方法名必须为`afterThrowing`，可以有两种参数形式：

`void afterThrowing(Throwable t)`或者`void afterThrowing(Method method, Object[] args, Object target, Throwable t)`二选一即可，但是要注意必须按照这种格式，因为后续调用的时候会对参数和方法名进行校验。

`Interceptor`接口也是一个空接口：

```java
public interface Interceptor {
}
```

但是继承自它的`MethodInterceptor`接口就定义了执行入口：

```java
public interface MethodInterceptor extends Interceptor {
	// 调用一切Advice的入口，invocation就是方法的入口点
	Object invoke(MethodInvocation invocation) throws Throwable;
}
```

而`MethodBeforeAdviceInterceptor`的源码如下：

```java
public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {
	// 持有一个Advice对象
	private final MethodBeforeAdvice advice;

  // 构造函数要求传入一个MethodBeforeAdvice的实现类
	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}
	
  // 调用逻辑很简单：先调用Advice的before方法，然后再调用被增强的方法即可
	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		return mi.proceed();
	}

}
```

这个逻辑非常简单，不多解释，而`AfterReturningAdviceInterceptor`的逻辑和这个一样，无非就是invoke函数中执行顺序的差异。

但是`ThrowsAdviceInterceptor`的逻辑就稍微复杂一点了：

```java
public class ThrowsAdviceInterceptor implements MethodInterceptor, AfterAdvice {
    // 固定调用函数名
	private static final String AFTER_THROWING = "afterThrowing";
	private static final Log logger = LogFactory.getLog(ThrowsAdviceInterceptor.class);
    // 持有的切点
	private final Object throwsAdvice;
	// 存储异常类型到方法的映射
	private final Map<Class<?>, Method> exceptionHandlerMap = new HashMap<>();

    // 构造函数
	public ThrowsAdviceInterceptor(Object throwsAdvice) {
		Assert.notNull(throwsAdvice, "Advice must not be null");
		this.throwsAdvice = throwsAdvice;
        // 这里从所有的方法中去寻找afterThrowing方法并且校验参数个数
		Method[] methods = throwsAdvice.getClass().getMethods();
		for (Method method : methods) {
			if (method.getName().equals(AFTER_THROWING) &&
					(method.getParameterCount() == 1 || method.getParameterCount() == 4)) {
                // 能适配的异常类型就是最后一个参数，这里将这个参数取出放入Map保存
				Class<?> throwableParam = method.getParameterTypes()[method.getParameterCount() - 1];
				if (Throwable.class.isAssignableFrom(throwableParam)) {
					// 保存异常类型，防止后续重复解析方法
					this.exceptionHandlerMap.put(throwableParam, method);
					if (logger.isDebugEnabled()) {
						logger.debug("Found exception handler method on throws advice: " + method);
					}
				}
			}
		}
        // 如果没有可以处理的异常则抛出异常
		if (this.exceptionHandlerMap.isEmpty()) {
			throw new IllegalArgumentException(
					"At least one handler method must be found in class [" + throwsAdvice.getClass() + "]");
		}
	}


	/**
	 * Return the number of handler methods in this advice.
	 */
	public int getHandlerMethodCount() {
		return this.exceptionHandlerMap.size();
	}


	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		try {
			return mi.proceed();
		}
		catch (Throwable ex) {
            // 获取异常对应的处理方法
			Method handlerMethod = getExceptionHandler(ex);
			if (handlerMethod != null) {
				invokeHandlerMethod(mi, ex, handlerMethod);
			}
			throw ex;
		}
	}

	/**
	 * Determine the exception handle method for the given exception.
	 * @param exception the exception thrown
	 * @return a handler for the given exception type, or {@code null} if none found
	 */
	@Nullable
	private Method getExceptionHandler(Throwable exception) {
		Class<?> exceptionClass = exception.getClass();
		if (logger.isTraceEnabled()) {
			logger.trace("Trying to find handler for exception of type [" + exceptionClass.getName() + "]");
		}
		Method handler = this.exceptionHandlerMap.get(exceptionClass);
        // 如果当前异常对应的方法为空，那么就不断寻找异常的超类是否有能够解析的方法
		while (handler == null && exceptionClass != Throwable.class) {
			exceptionClass = exceptionClass.getSuperclass();
			handler = this.exceptionHandlerMap.get(exceptionClass);
		}
        // 如果异常包括其超类都没有能使用的方法则退返回null
		if (handler != null && logger.isTraceEnabled()) {
			logger.trace("Found handler for exception of type [" + exceptionClass.getName() + "]: " + handler);
		}
		return handler;
	}

	private void invokeHandlerMethod(MethodInvocation mi, Throwable ex, Method method) throws Throwable {
        // 主要对afterThrowing 1个参数和4个参数的适配
		Object[] handlerArgs;
		if (method.getParameterCount() == 1) {
			handlerArgs = new Object[] {ex};
		}
		else {
			handlerArgs = new Object[] {mi.getMethod(), mi.getArguments(), mi.getThis(), ex};
		}
		try {
			method.invoke(this.throwsAdvice, handlerArgs);
		}
		catch (InvocationTargetException targetEx) {
			throw targetEx.getTargetException();
		}
	}

}
```

`ThrowsAdviceInterceptor`内部内置了一个Map对象用于存储异常Class到Method的映射，这样做的原因是方便一个`ThrowsAdvice`定义多种异常处理函数，只要它们的名字都是afterThrowing，使用了规定的一个或四个参数，那么就可以任意重载。在构造函数中会被预先缓存避免调用时重复解析。同时由于异常具有继承机制，所以这里会找到离当前异常层次最近的一个层次的方法进行调用。

好了说完了切面，接下来就是切点PointCut了：

#### Pointcut

Pointcut要做的事很简单，那就是确定要增强哪个方法，它会从类层次上和方法层次上进行判断，其源码如下：

```java
public interface Pointcut {
	// 类过滤器
	ClassFilter getClassFilter();
	// 方法过滤器
	MethodMatcher getMethodMatcher();
	// 预置的过滤器，对一切类和一切方法都返回True，即都会被增强
	Pointcut TRUE = TruePointcut.INSTANCE;
}
```

`ClassFilter`的源码如下：

```java
public interface ClassFilter {
	// 方法过滤器，根据Class对象判断是否匹配
	boolean matches(Class<?> clazz);
	// 预置过滤器，对一切类都返回True
	ClassFilter TRUE = TrueClassFilter.INSTANCE;
}
```

`MethodMatcher`的源码如下：

```java
public interface MethodMatcher {
  // 静态匹配方法，不需要运行时参数，性能较好
	boolean matches(Method method, Class<?> targetClass);
	// 标志是运行时匹配还是静态匹配
	boolean isRuntime();
	// 运行时匹配方法，会传入调用函数的所有参数，性能较差
	boolean matches(Method method, Class<?> targetClass, Object... args);
	// 预置过滤器，对一切方法都返回True
	MethodMatcher TRUE = TrueMethodMatcher.INSTANCE;
}
```

Pointcut可以自己实现也可以使用一些其它的匹配规则或方法，这里不讨论其它方法的实现，只讨论原生Pointcut的实现原理

#### Advisor

Advisor是Advice和Pointcut的结合，它非常重要，因为我们定义了Advice即定义了做什么和什么时候做，而Pointcut则定义了在哪做，那么我们必须把这三个要素都结合起来才能发挥作用，所以这就是Advisor的作用。Advisor上承IoC容器，是IoC容器调用AOP的入口，下接AOP基础设施，是一个非常重要的概念。其源码如下：

```java
public interface Advisor {
  // 默认空Advice
	Advice EMPTY_ADVICE = new Advice() {};

	Advice getAdvice();
	// 本方法已经不再使用！
	boolean isPerInstance();
}
```

可以看到Advisor只定义了Advice对象没有Pointcut对象，这是因为对于确定被增强的对象来说，Pointcut只是一种方式，还可以使用诸如远程调用、序列化、字节码等方式确定切入点，而Pointcut指定义了类层次和方法层次上的切入，这并不全面，所以Advisor类并不指定切入点的具体形式。但是对于要执行操作的时机和执行什么操作来说，这是固定存在的，这也解释了为什么Advisor类中只有Advice对象这个原因。如果要使用带有Pointcut的Advisor可以使用它的子接口`PointcutAdvisor`，源码如下：

```java
public interface PointcutAdvisor extends Advisor {
	Pointcut getPointcut();
}
```

很简单，在`Advisor`的基础上添加了`Pointcut`对象即可，Spring中会比较多的使用到`DefaultPointcutAdvisor`这个类：

```java
public class DefaultPointcutAdvisor extends AbstractGenericPointcutAdvisor implements Serializable {

	private Pointcut pointcut = Pointcut.TRUE;

	public DefaultPointcutAdvisor() {
	}

	public DefaultPointcutAdvisor(Advice advice) {
		this(Pointcut.TRUE, advice);
	}

	public DefaultPointcutAdvisor(Pointcut pointcut, Advice advice) {
		this.pointcut = pointcut;
		setAdvice(advice);
	}

	public void setPointcut(@Nullable Pointcut pointcut) {
		this.pointcut = (pointcut != null ? pointcut : Pointcut.TRUE);
	}

	@Override
	public Pointcut getPointcut() {
		return this.pointcut;
	}

	@Override
	public String toString() {
		return getClass().getName() + ": pointcut [" + getPointcut() + "]; advice [" + getAdvice() + "]";
	}
}
```

代码很简单不加注释了，默认的Pointcut就是`Pointcut`接口中定义的`TruePointcut`的实例，会作用于一切对象上。如果要自己实现支持`Pointcut`的`Advisor`可以考虑直接继承这个类或者`PointcutAdvisor`接口，最后会有示例。

### 代理模式

我们都知道AOP是依赖代理的方式实现功能的，Spring AOP使用的代理方式有两种：JDK代理和CGLIB代理。前者基于反射，后者基于字节码；前者创建慢运行快，后者创建快运行慢。下面回顾以下这两种代理模式的特点：

#### JDK代理

JDK代理的最主要特点就是被代理对象必须实现一个接口，因为被代理之后是通过`Proxy`类的`newProxyInstance`方法来创建对象的，我们看一下这个方法定义：

```java
public static Object newProxyInstance(ClassLoader loader,
                                          Class<?>[] interfaces,
                                          InvocationHandler h) {
        Objects.requireNonNull(h);
        final Class<?> caller = System.getSecurityManager() == null
                                    ? null
                                    : Reflection.getCallerClass();
        Constructor<?> cons = getProxyConstructor(caller, loader, interfaces);
        return newProxyInstance(caller, cons, h);
}
```

明确说明了需要一个interface数组来寻找接口实现类的构造函数，所以如果使用JDK动态代理那么必须要继承某个接口。同时对代理对象进行增强的类需要实现`InvocationHandler`接口。

下面给出一个使用JDK动态代理的例子：

```java
class SmsJDKProxyHandler implements InvocationHandler {
    private Object target;

    public SmsJDKProxyHandler(Object target) {
        this.target = target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        System.out.println("Before proxy invoke");
        Object result = method.invoke(target, args);
        System.out.println("After proxy invock");
        return result;
    }
}

public class JDKProxyTest {
    public static Object getSmsHandlerProxy(Object target) {
        return Proxy.newProxyInstance(
                target.getClass().getClassLoader(),
                // 如果没有接口的话，代理类是找不到它对应的类型的
                target.getClass().getInterfaces(),
                new SmsJDKProxyHandler(target)
        );
    }

    public static void main(String[] args) {
        SmsService service = ((SmsService) getSmsHandlerProxy(new SmsServiceImpl()));
        service.send("Hi");
    }
}
```

#### CGLIB代理

CGLIB是基于字节码的代理，所以它不需要被增强的类实现任何接口，一种使用如下：

```java
class NoInterfaceSmsService {
    public void send(String message) {
        System.out.println("Send: " + message);
    }
}

class CglibMethodInterceptor implements MethodInterceptor {
    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        System.out.println("Before proxy invoke");
        Object result = method.invoke(o, args);
        System.out.println("After proxy invoke");
        return result;
    }
}

public class CglibProxy {
    public static void main(String[] args) {
        Enhancer enhancer = new Enhancer();
      	// 设置类加载器
        enhancer.setClassLoader(NoInterfaceSmsService.class.getClassLoader());
      	// 设置需要被增强的类
        enhancer.setSuperclass(NoInterfaceSmsService.class);
      	// 回调函数就是拦截函数
        enhancer.setCallback(new CglibMethodInterceptor());
        NoInterfaceSmsService service = (NoInterfaceSmsService) enhancer.create();
        service.send("Java");
    }
}
```

### Spring AOP代理的实现

我们知道，要实现AOP主要依赖的是代理机制，那么AOP代理的实现最核心的部分就是代理对象的生成操作了。对于Spring对象而言，是通过调用`ProxyFactoryBean`来完成这个任务的，在`ProxyFactoryBean`中封装了主要代理对象的生成过程。在这个生成过程中可以使用JDK的Prxoy和CGLIB两种方式，以`ProxyFactory`的设计为中心，可以看到相关的类继承关系：

<img src="
https://evanblog.oss-cn-shanghai.aliyuncs.com/image/Spring/Spring AOP/ProxyFactory-Arch.png" alt="ProxyFactory-Arch" style="zoom:50%;" />

在上面的类继承关系中，`ProxyConfig`是数据基类，为子类提供了配置属性；而在`AdvisedSupport`中，封装了AOP对通知和通知器的相关操作，但对于具体的AOP代理对象的创建，`AdvisedSupport`把它交给它的子类们去完成，对于`ProxyCreatorSupport`，可以将它看成是其子类创建AOP代理对象的一个辅助类。

我们分析Spring AOP的实现原理的时候从`ProxyFactoryBean`类入手分析，它是使用Spring AOP创建AOP应用的底层方法。我们下面先给出一个AOP的Bean配置：

```xml
<bean id="testAdvisor" class="com.learn.spring.chcp3.TestAdvisor" />
<bean id="proxyFactoryBean" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="proxyInterfaces" value="com.learn.spring.chcp3.pojo.ProxyInterfaces"/>
    <property name="target">
        <bean class="com.learn.spring.chcp3.pojo.ProxyInterfacesImpl" lazy-init="true"/>
    </property>
    <property name="interceptorNames">
        <list>
            <value>testAdvisor</value>
        </list>
    </property>
<!--        <property name="proxyTargetClass" value="true" />-->
</bean>
```

其中首先定义了一个Advisor的Bean也就是拦截器，然后定义了一个属性proxyInterfaces接口用于指定代理的接口类（可以省略），target属性定义了目标类，interceptorNames定义了所有需要应用的Advisor，proxyTargetClass指定了是否是以类的形式进行代理（即使用CGLIB代理，可以省略）。然后我们来看`ProxyFactoryBean`的源码，我们知道作为一个FactoryBean，它的入口就是getObject函数：

#### ProxyFactoryBean

```java
public Object getObject() throws BeansException {
    initializeAdvisorChain();
    if (isSingleton()) {
        return getSingletonInstance();
    }
    else {
        if (this.targetName == null) {
            logger.info("Using non-singleton proxies with singleton targets is often undesirable. " +
                    "Enable prototype proxies by setting the 'targetName' property.");
        }
        return newPrototypeInstance();
    }
}
```

getObject方法先对通知器链进行初始化，通知器链封装了一系列的拦截器，这些拦截器都要从配置中读取，然后为代理对象的生成做好准备。在生成代理对象时，因为Spring中有singleton类型和prototype类型这两种不同的Bean，所以要对代理对象的生成做一个区分。

接着我们来看通知器链的初始化函数即`initializeAdvisorChain`：

```java
private synchronized void initializeAdvisorChain() throws AopConfigException, BeansException {
    // 如果已经被初始化过了，那么就不再重复初始化
    if (this.advisorChainInitialized) {
        return;
    }

    if (!ObjectUtils.isEmpty(this.interceptorNames)) {
        // 判断当前是否存在BeanFactory, 由于ProxyFactoryBean类实现了BeanFactoryAware接口，所以它能获得到具体的BeanFactory
        if (this.beanFactory == null) {
            throw new IllegalStateException("No BeanFactory available anymore (probably due to serialization) " +
                    "- cannot resolve interceptor names " + Arrays.asList(this.interceptorNames));
        }

        // 最后一个AOP不能是GLOBAL拦截器，除非我们指定了target
        if (this.interceptorNames[this.interceptorNames.length - 1].endsWith(GLOBAL_SUFFIX) &&
                this.targetName == null && this.targetSource == EMPTY_TARGET_SOURCE) {
            throw new AopConfigException("Target required after globals");
        }

        for (String name : this.interceptorNames) {
            // 如果是GLOBAL AOP，那么需要单独处理一下
            if (name.endsWith(GLOBAL_SUFFIX)) {
                if (!(this.beanFactory instanceof ListableBeanFactory)) {
                    throw new AopConfigException(
                            "Can only use global advisors or interceptors with a ListableBeanFactory");
                }
                addGlobalAdvisor((ListableBeanFactory) this.beanFactory,
                        name.substring(0, name.length() - GLOBAL_SUFFIX.length()));
            }
            else {
                Object advice;
                // 需要判断一下Advisor是单例还是多例
                if (this.singleton || this.beanFactory.isSingleton(name)) {
                    advice = this.beanFactory.getBean(name);
                }
                else {
                    advice = new PrototypePlaceholderAdvisor(name);
                }
                // 添加到Advisor链中
                addAdvisorOnChainCreation(advice, name);
            }
        }
    }
    this.advisorChainInitialized = true;
}
```

我们深入到`addAdvisorOnChainCreation`方法中看一下一个Bean是如何被包装成Advisor的：

```java
private void addAdvisorOnChainCreation(Object next, String name) {
    // 转换为Advisor类型，因为装配的Bean有可能只是Advice类型的实现类
    Advisor advisor = namedBeanToAdvisor(next);
    addAdvisor(advisor);
}

private Advisor namedBeanToAdvisor(Object next) {
    try {
        // 调用包装器将对象包装为Advisor类型
        return this.advisorAdapterRegistry.wrap(next);
    }
    catch (UnknownAdviceTypeException ex) {
        throw new AopConfigException("Unknown advisor type " + next.getClass() +
                "; Can only include Advisor or Advice type beans in interceptorNames chain except for last entry, " +
                "which may also be target or TargetSource", ex);
    }
}

public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {
    // 如果对象已经是一个Advisor了，那么直接类型转换后返回即可
    if (adviceObject instanceof Advisor) {
        return (Advisor) adviceObject;
    }
    // 如果既不是Advisor也不是Advice，说明这个对象不是一个有效的AOP对象，抛出异常
    if (!(adviceObject instanceof Advice)) {
        throw new UnknownAdviceTypeException(adviceObject);
    }
    Advice advice = (Advice) adviceObject;
    // 如果这个advice是一个MethodInterceptor类型的实现类, 说明指定了拦截方法但是没有指定拦截时机
    // 所以这里用一个默认的Pointcut给它包装
    if (advice instanceof MethodInterceptor) {
        return new DefaultPointcutAdvisor(advice);
    }
    // 否则说明，它是BeforeAdvice、AfterAdvice或ThrowsAdvice的子类，那么调用对应的适配器生成即可
    for (AdvisorAdapter adapter : this.adapters) {
        // Check that it is supported.
        if (adapter.supportsAdvice(advice)) {
            return new DefaultPointcutAdvisor(advice);
        }
    }
    // 如果是自定义的Advice那么需要自己提供Adapter，否则抛出异常
    throw new UnknownAdviceTypeException(advice);
}
```

在wrap方法中我们可以看到，有效的AOP类型只有Advisor接口和Advice接口的实现类。其中如果我们要自定义切入的时机的话，我们一般选择继承`AbstractGenericPointcutAdvisor`类；而如果我们要使用预定的时机的话一般继承`MethodBeforeAdvice`、`AfterReturningAdvice`和`ThrowsAdvice`，当然也可以直接重写`MethodInterceptor`接口来实现。而如果自己定义了一个`Advice`，那么一定要提供一个`AdvisorAdapter`，默认提供了三个分别是`MethodBeforeAdviceAdapter`、`AfterReturningAdviceAdapter`以及`ThrowsAdviceAdaapter`用来将默认提供的三个预定切入实现类转换为对应的`Advisor`。

##### AdvisorAdapter

我们来看`AdvisorAdapter`接口的定义以及`MethodBeforeAdviceAdapter`类的实现：

```java
public interface AdvisorAdapter {

	boolean supportsAdvice(Advice advice);

	MethodInterceptor getInterceptor(Advisor advisor);
}

class MethodBeforeAdviceAdapter implements AdvisorAdapter, Serializable {

	@Override
	public boolean supportsAdvice(Advice advice) {
		return (advice instanceof MethodBeforeAdvice);
	}

	@Override
	public MethodInterceptor getInterceptor(Advisor advisor) {
		MethodBeforeAdvice advice = (MethodBeforeAdvice) advisor.getAdvice();
		return new MethodBeforeAdviceInterceptor(advice);
	}
}

public class MethodBeforeAdviceInterceptor implements MethodInterceptor, BeforeAdvice, Serializable {

	private final MethodBeforeAdvice advice;

	public MethodBeforeAdviceInterceptor(MethodBeforeAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		this.advice.before(mi.getMethod(), mi.getArguments(), mi.getThis());
		return mi.proceed();
	}
}

public class AfterReturningAdviceInterceptor implements MethodInterceptor, AfterAdvice, Serializable {

	private final AfterReturningAdvice advice;

	public AfterReturningAdviceInterceptor(AfterReturningAdvice advice) {
		Assert.notNull(advice, "Advice must not be null");
		this.advice = advice;
	}

	@Override
	public Object invoke(MethodInvocation mi) throws Throwable {
		Object retVal = mi.proceed();
		this.advice.afterReturning(retVal, mi.getMethod(), mi.getArguments(), mi.getThis());
		return retVal;
	}
}
```

可以看到`MethodBeforeAdviceAdapter`类通过构造一个`MethodInterceptor`类来实现适配，这里生成的是`MethodBeforeAdviceInterceptor`类，在其中的invoke方法中可以看到，就是在调用实际方法之前调用了Advice的方法实现了在方法前注入的功能。而`AfterReturningAdviceInterceptor`则相反，先执行方法再调用Advice。这里比较复杂的是`ThrowsAdviceAdapter`：

##### ThrowsAdviceAdapter

```java
public Object invoke(MethodInvocation mi) throws Throwable {
    try {
        return mi.proceed();
    }
    catch (Throwable ex) {
        // 抛出异常时，先去寻找异常对应的处理器
        Method handlerMethod = getExceptionHandler(ex);
        // 如果存在处理器，那么调用处理器方法
        if (handlerMethod != null) {
            invokeHandlerMethod(mi, ex, handlerMethod);
        }
        throw ex;
    }
}
```

它的异常处理类中可能包含若干个异常处理器，可以处理不同的异常，所以`ThrowsAdviceAdapter`中使用了一个Map来保存异常对应的处理器，通过`getExceptionHandler`方法从Map中获取异常处理器：

```java
private Method getExceptionHandler(Throwable exception) {
    Class<?> exceptionClass = exception.getClass();
    // 从缓存的Map中构建异常到处理器的映射
    Method handler = this.exceptionHandlerMap.get(exceptionClass);
    // 如果当前异常找不到处理器，那么就尝试寻找当前异常的父类，如果到Throwable类也没有可用处理器那么就放弃
  	while (handler == null && exceptionClass != Throwable.class) {
			exceptionClass = exceptionClass.getSuperclass();
			handler = this.exceptionHandlerMap.get(exceptionClass);
		}
    return handler;
}
```

而获取的异常处理器通过`invokeHandlerMethod`方法调用：

```java
private void invokeHandlerMethod(MethodInvocation mi, Throwable ex, Method method) throws Throwable {
    Object[] handlerArgs;
    // 拼接参数，我们之前分析过，ThrowsAdvice接口没有定义方法，但是规定要么是1个参数，要么是4个参数
    // 这里分别处理这两种情况，拼接参数到数组中
    if (method.getParameterCount() == 1) {
        handlerArgs = new Object[] {ex};
    }
    else {
        handlerArgs = new Object[] {mi.getMethod(), mi.getArguments(), mi.getThis(), ex};
    }
    try {
        // 调用处理方法
        method.invoke(this.throwsAdvice, handlerArgs);
    }
    catch (InvocationTargetException targetEx) {
        throw targetEx.getTargetException();
    }
}
```

因为`ThrowsAdvice`中定义了1个参数和4个参数两种参数列表，所以这里进行一个参数适配，最后利用反射调用对应的方法。而异常到处理器的映射是在构造函数中建立的：

```java
public ThrowsAdviceInterceptor(Object throwsAdvice) {
    // 获得方法的入口
    this.throwsAdvice = throwsAdvice;

    Method[] methods = throwsAdvice.getClass().getMethods();
    for (Method method : methods) {
        // 寻找异常处理方法，有两个要素：1. 方法名必须是afterThrowing，2. 有1个参数或4个参数
        // 这个要素在ThrowsAdvice处定义了
        if (method.getName().equals(AFTER_THROWING) &&
                (method.getParameterCount() == 1 || method.getParameterCount() == 4)) {
            // 无论1个参数还是4个参数，最后一个参数一定是该处理器可以处理的异常类型
            // 所以这里判断，如果是Throwable类型的子类，那么就注册到Map中保存
            Class<?> throwableParam = method.getParameterTypes()[method.getParameterCount() - 1];
            if (Throwable.class.isAssignableFrom(throwableParam)) {
                this.exceptionHandlerMap.put(throwableParam, method);
            }
        }
    }
    // 如果一个异常处理器都没有，抛出异常
    if (this.exceptionHandlerMap.isEmpty()) {
        throw new IllegalArgumentException(
                "At least one handler method must be found in class [" + throwsAdvice.getClass() + "]");
    }
}
```

以上就是`ThrowsAdviceAdapter`构造的过程了。我们接下来回到`getObject`方法中，来看单例对象是怎么样被创建出来的，来看`getSingletonInstance`方法：

##### 单例对象的创建

```java
private synchronized Object getSingletonInstance() {
    // 因为一个ProxyFactoryBean对象只会负责生产一个类型的Bean对象，所以在ProxyFactoryBean中会对生成的对象进行一个缓存
    // 这里先判断是否缓存过代理对象，如果缓存过那么就直接返回，否则才会触发创建过程
    if (this.singletonInstance == null) {
        // TargetSource是用来记录代理目标对象，可以在配置文件中提供target、targetName或targetClass
        // 最终都会被解析成TargetSource类型的对象
        this.targetSource = freshTargetSource();
        // 如果允许自动检查接口（默认允许）并且Bean属性中没有指定接口并且Bean属性中没有强制代理类
        // 那么由AOP基础设施来获取当前类实现的所有接口
        if (this.autodetectInterfaces && getProxiedInterfaces().length == 0 && !isProxyTargetClass()) {
            Class<?> targetClass = getTargetClass();
            if (targetClass == null) {
                throw new FactoryBeanNotInitializedException("Cannot determine target class for proxy");
            }
            setInterfaces(ClassUtils.getAllInterfacesForClass(targetClass, this.proxyClassLoader));
        }
        super.setFrozen(this.freezeProxy);
        // 这里为最终生成代理对象的地方
        this.singletonInstance = getProxy(createAopProxy());
    }
    return this.singletonInstance;
}
```

这里首先要先看targetSource，`TargetSource`指明了当前这个AOP要代理的目标对象，它在Bean的配置文件中给出，有三种配置形式target、targetName和targetClass，三者选一个即可，选用target那么传递的是一个被代理的Bean对象，targetName传递的是一个Bean的名字，targetClass传递的是一个Bean的类型，而这里的freshTargetSource做的事就是从IoC容器中解析Target并封装到`TargetSource`接口的实现类中。`TargetSource`的源码如下：

```java
public interface TargetSource extends TargetClassAware {
	@Override
	@Nullable
	Class<?> getTargetClass();

  // static表明每一次当前TargetSource对应的对象是恒定不变的
  // 否则表明可能是会变化的，例如从对象池中取出对象，那么有可能每一次取出的是不一样的
	boolean isStatic();

	@Nullable
	Object getTarget() throws Exception;

	void releaseTarget(Object target) throws Exception;
}
```

在配置完代理的目标类后要考虑当前类是否有接口，Spring AOP对于有实现接口的类一般采用JDK动态代理，反之采用CGLIB代理。这里如果没有显式指定接口，那么需要一下三个条件满足才会自动检测接口：1. 允许自动检测即autodetectInterfaces为true（默认运行）；2. 没有显示指定任何接口；3. 没有强制要求代理类即proxyTargetClass为false。如果满足了以上三个条件，那么就会由AOP的基础设施自动寻找当前类实现的接口。

我们回到`getSingletonInstance`方法的最后来看，AOP代理对象的生成是通过ProxyFactory来生成的，首先调用`ProxyBeanFactory`的基类`ProxyCreatorSupport`类的`createAopProxy`，接着获取AopProxyFactory对象的实现类`DefaultAopProxyFactory`，通过后者的`createAopProxy`方法实现具体对象的创建：

##### 具体代理对象的生产

```java
public AopProxy createAopProxy(AdvisedSupport config) throws AopConfigException {
    // 如果满足：启用最优策略，强制代理类，当前接口不被支持这三个条件之一才有可能使用CGLIB代理
    if (config.isOptimize() || config.isProxyTargetClass() || hasNoUserSuppliedProxyInterfaces(config)) {
        Class<?> targetClass = config.getTargetClass();
        if (targetClass == null) {
            throw new AopConfigException("TargetSource cannot determine target class: " +
                    "Either an interface or a target is required for proxy creation.");
        }
        // 如果被代理的是一个接口，或者被代理的类已经是一个代理对象，那么还是优先使用JDK动态代理
        if (targetClass.isInterface() || Proxy.isProxyClass(targetClass)) {
            return new JdkDynamicAopProxy(config);
        }
        return new ObjenesisCglibAopProxy(config);
    }
    else {
        return new JdkDynamicAopProxy(config);
    }
}
```

这里会根据不同的情况创建不同的代理，最终就会创建AopProxy的实现类返回，生成最终的代理对象。到此一个被代理的AOP对象就已经生成了。

#### AopProxy

接下来我们看一下JDK代理和CGLIB代理两种代理方式各自的实现细节，我们先来看最上层的`AopProxy`接口：

```java
public interface AopProxy {
    // 创建一个代理对象，使用当前类默认的类加载器
	Object getProxy();

    // 创建一个代理对象，使用指定的类加载器
	Object getProxy(@Nullable ClassLoader classLoader);
}
```

`AopProxy`实际就定义了一个代理对象应该做的事，即生成实际对象。我们接着看两种不同的代理的实现细节：

##### JDKDynamicProxy

JDK动态代理创建对象的过程非常简单：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    Class<?>[] proxiedInterfaces = AopProxyUtils.completeProxiedInterfaces(this.advised, true);
    findDefinedEqualsAndHashCodeMethods(proxiedInterfaces);
    return Proxy.newProxyInstance(classLoader, proxiedInterfaces, this);
}
```

无非就是先找到代理对象，然后调用Proxy类即可创建一个代理对象了。因为`JDKDynamicProxy`类作为JDK动态代理的控制类，还实现了`InvocationHandler`接口，所以JDK动态代理对方法的拦截就在`invoke`方法中了，可以看到在创建新的Proxy对象时，newProxyInstance的第三个参数就是指定控制对象的即`InvocationHandler`实现类对象的，这里将this作为参数传进去了，说明当前JDKDynamicProxy类就是控制类：

```java
public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    // 获取控制目标
    TargetSource targetSource = this.advised.targetSource;
    Object target = null;

    try {
        if (!this.equalsDefined && AopUtils.isEqualsMethod(method)) {
            // 如果没有实现equals方法需要单独处理
            return equals(args[0]);
        }
        else if (!this.hashCodeDefined && AopUtils.isHashCodeMethod(method)) {
            // 调用hashCode方法也同理
            return hashCode();
        }
        else if (method.getDeclaringClass() == DecoratingProxy.class) {
            // 如果被代理的类是被装饰过的类，那么需要单独处理
            return AopProxyUtils.ultimateTargetClass(this.advised);
        }
        else if (!this.advised.opaque && method.getDeclaringClass().isInterface() &&
                method.getDeclaringClass().isAssignableFrom(Advised.class)) {
            // 如果不允许使用Advice，或者当前修饰的类是一个接口，或者要运行的方法是Advised子类的方法
            // 那么这里直接尝试调用反射运行，而不经过拦截器处理
            return AopUtils.invokeJoinpointUsingReflection(this.advised, method, args);
        }

        Object retVal;

        if (this.advised.exposeProxy) {
            // 如果需要暴露代理对象，那么将其设置到AopContext中
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }

        // 获取目标对象和目标类
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);

        // 获取缓存的拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);

        // 如果没有缓存拦截器链，那么我们就采用直接利用反射的方式调用方法，避免创建MethodInvocation导致性能下降
        if (chain.isEmpty()) {
            // 这里只要知道最后调用方法的是InvokeerInterceptor就可以判断出来最后没有对这个对象应用任何拦截器，
            // 所以也就不需要考虑热交换以及其它的代理工作
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = AopUtils.invokeJoinpointUsingReflection(target, method, argsToUse);
        }
        else {
            // 如果存在拦截器链，那么我们就需要创建一个MethodInvocation来处理
            MethodInvocation invocation =
                    new ReflectiveMethodInvocation(proxy, target, method, args, targetClass, chain);
            // 沿着拦截器链继续前进
            retVal = invocation.proceed();
        }

        // 处理方法的返回值
        Class<?> returnType = method.getReturnType();
        if (retVal != null && retVal == target &&
                returnType != Object.class && returnType.isInstance(proxy) &&
                !RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
            // 这里要处理的是返回this指针的问题，如果返回的类型和当前类型兼容，或者在当前类中返回了this，
            // 都是合法的，在这里返回方法真正的代理对象，否则则认为是非法的，无法获取真正的代理对象
            retVal = proxy;
        }
        else if (retVal == null && returnType != Void.TYPE && returnType.isPrimitive()) {
            // 这里判断是不是对基本类型返回了null，这是非法的
            throw new AopInvocationException(
                    "Null return value from advice does not match primitive return type for: " + method);
        }
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            // 如果targetSource不是静态的，说明每次产生的Target都不一样，这里要考虑释放对象的问题
            // 因为对象有可能是从对象池中获取的
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            // 恢复旧的AOP代理上下文
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}
```

这里的调用逻辑有一点复杂，首先需要判断调用的是当前对象中的哪一个方法，对于没有重写equals和hashCode的方法这里要进行捕获，并且用当前类定义的equals和hashCode来代替。如果被代理的类已经是一个代理类，那么需要特殊处理，并且如果被代理的类不允许使用Advise或者是一个接口或者是Advised类的子类，那么就调用反射直接运行，不经过代理。接着获取已经加载的拦截器链，如果拦截器链为空说明不需要任何拦截，那么直接利用反射调用即可；反之委托给`ReflectiveMethodInvocation`类来调用具体的方法；在调用结束后还需要处理返回值的this和基本类型为null的情况，最终返回。

以上就是整个代理的执行流程了，我们接着来分析一下里面使用的一些方法：

首先是直接反射调用的`AopUtils.invokeJoinpointUsingReflection`方法：

```java
public static Object invokeJoinpointUsingReflection(@Nullable Object target, Method method, Object[] args) throws Throwable {
    try {
        // 直接利用反射调用
        ReflectionUtils.makeAccessible(method);
        return method.invoke(target, args);
    }   
    catch (InvocationTargetException ex) {
        // 调用方法抛出的受查异常，这里重新抛出
        throw ex.getTargetException();
    }
    catch (IllegalArgumentException ex) {
        throw new AopInvocationException("AOP configuration seems to be invalid: tried calling method [" +
            method + "] on target [" + target + "]", ex);
    }
    catch (IllegalAccessException ex) {
        throw new AopInvocationException("Could not access method [" + method + "]", ex);
    }
}
```

逻辑很简单，直接利用反射调用即可。

接下来看this.advised的`getInterceptorsAndDynamicInterceptionAdvice`方法，这里的this.advised是`AdvisedSupport`类型的，这个对象是在调用`getSingletonInstance`方法时，通过`createAopProxy`传入的，传入的是一个`ProxyCreatorSupport`类型的对象。这里调用的`getInterceptorsAndDynamicInterceptionAdvice`方法是在`AdvisedSupport`类中定义的，实现如下：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(Method method, @Nullable Class<?> targetClass) {
    // 将method包装一下，适配Map的key
    MethodCacheKey cacheKey = new MethodCacheKey(method);
    // 尝试从缓存中读取
    List<Object> cached = this.methodCache.get(cacheKey);
    // 如果读不到，那么尝试重新加载
    if (cached == null) {
        cached = this.advisorChainFactory.getInterceptorsAndDynamicInterceptionAdvice(
                this, method, targetClass);
        this.methodCache.put(cacheKey, cached);
    }
    return cached;
}
```

这里很显然利用了缓存的方法，先从缓存中去取拦截器链，如果没有则调用工厂类`AdvisorChainFactory`来生成拦截器链，再放入缓存中，我们接着来看工厂生产拦截器链的方法`getInterceptorsAndDynamicInterceptionAdvice`，它在`AdvisorChainFactory`的子类`DefaultAdvisorChainFactory`中实现：

```java
public List<Object> getInterceptorsAndDynamicInterceptionAdvice(
			Advised config, Method method, @Nullable Class<?> targetClass) {
        // 这里要先处理Introduction Advice，还要保证所有Advice都按原来的顺序
		AdvisorAdapterRegistry registry = GlobalAdvisorAdapterRegistry.getInstance();
		Advisor[] advisors = config.getAdvisors();
		List<Object> interceptorList = new ArrayList<>(advisors.length);
		Class<?> actualClass = (targetClass != null ? targetClass : method.getDeclaringClass());
		Boolean hasIntroductions = null;

		for (Advisor advisor : advisors) {
			if (advisor instanceof PointcutAdvisor) {
				// 如果是PointcutAdvisor，那么就需要调用其中类匹配器和方法匹配器来判断当前拦截器是否匹配当前类和方法
				PointcutAdvisor pointcutAdvisor = (PointcutAdvisor) advisor;
        // 先判断类是否满足
				if (config.isPreFiltered() || pointcutAdvisor.getPointcut().getClassFilter().matches(actualClass)) {
					MethodMatcher mm = pointcutAdvisor.getPointcut().getMethodMatcher();
					boolean match;
          // 如果是引介类型，那么需要特殊判断
					if (mm instanceof IntroductionAwareMethodMatcher) {
						if (hasIntroductions == null) {
							hasIntroductions = hasMatchingIntroductions(advisors, actualClass);
						}
						match = ((IntroductionAwareMethodMatcher) mm).matches(method, actualClass, hasIntroductions);
					}
					else {
            // 再判断方法是否满足
						match = mm.matches(method, actualClass);
					}
          // 如果满足
					if (match) {
            // 获取已经注册支持当前切面的拦截器
						MethodInterceptor[] interceptors = registry.getInterceptors(advisor);
             // 动态方法需要对拦截器进行包装，静态的则不用，直接添加即可
						if (mm.isRuntime()) {
							for (MethodInterceptor interceptor : interceptors) {
								interceptorList.add(new InterceptorAndDynamicMethodMatcher(interceptor, mm));
							}
						}
						else {
							interceptorList.addAll(Arrays.asList(interceptors));
						}
					}
				}
			}
			else if (advisor instanceof IntroductionAdvisor) {
        // 引入增强处理
				IntroductionAdvisor ia = (IntroductionAdvisor) advisor;
				if (config.isPreFiltered() || ia.getClassFilter().matches(actualClass)) {
					Interceptor[] interceptors = registry.getInterceptors(advisor);
					interceptorList.addAll(Arrays.asList(interceptors));
				}
			}
			else {
        // 其它增强说明没有切入点的限制，直接切入即可
				Interceptor[] interceptors = registry.getInterceptors(advisor);
				interceptorList.addAll(Arrays.asList(interceptors));
			}
		}

		return interceptorList;
	}
```

这里要首先解释一下这个`IntroductionAdvisor`，它的作用是让原来没有实现某个接口的类具备某个接口的功能，并且可以视作当前接口的一个实现类。所以这个切入要在所有切入之前完成，因为它是对类进行类似类型变更的操作，所以必须先完成。这里我们可以看到，实际上是调用registry.getInterceptors来获取当前Advisor注册的所有拦截器。这个registry就是`DefaultAdvisorAdapterRegistry`类的对象，我们之前已经分析过其中的`wrap`方法，我们现在来看一下其它的方法：

```java
public class DefaultAdvisorAdapterRegistry implements AdvisorAdapterRegistry, Serializable {

	private final List<AdvisorAdapter> adapters = new ArrayList<>(3);

    // 默认初始化三个Adapter，对应三种定义好的Advice
	public DefaultAdvisorAdapterRegistry() {
		registerAdvisorAdapter(new MethodBeforeAdviceAdapter());
		registerAdvisorAdapter(new AfterReturningAdviceAdapter());
		registerAdvisorAdapter(new ThrowsAdviceAdapter());
	}

	@Override
	public Advisor wrap(Object adviceObject) throws UnknownAdviceTypeException {...}

	@Override
	public MethodInterceptor[] getInterceptors(Advisor advisor) throws UnknownAdviceTypeException {
		List<MethodInterceptor> interceptors = new ArrayList<>(3);
		Advice advice = advisor.getAdvice();
        // 如果是自定义的Methodinterceptor拦截器，那么直接加入即可，不需要适配
		if (advice instanceof MethodInterceptor) {
			interceptors.add((MethodInterceptor) advice);
		}
        // 否则调用适配器获取拦截器
		for (AdvisorAdapter adapter : this.adapters) {
			if (adapter.supportsAdvice(advice)) {
				interceptors.add(adapter.getInterceptor(advisor));
			}
		}
        // 如果没有有效拦截器，那么抛出异常
		if (interceptors.isEmpty()) {
			throw new UnknownAdviceTypeException(advisor.getAdvice());
		}
        // 否则返回即可
		return interceptors.toArray(new MethodInterceptor[0]);
	}

	@Override
	public void registerAdvisorAdapter(AdvisorAdapter adapter) {
		this.adapters.add(adapter);
	}

}
```

逻辑很简单，要自定义的话，那就要自己提供Adapter或者自己实现`MethodInterceptor`接口；如果是使用自带的Advice，则实现对应接口即可，无需过多处理。至于适配器的工作流程和创建出来的拦截器，上面都已经说过，这里不再过多的赘述。我们下面来看实际调用invoke方法时，拦截器是怎么起作用的：

回到invoke方法中，我们可以看到实际上是通过调用`ReflectiveMethodInvocation`类的`proceed`实现的，我们来看一下这个方法的实现：

```java
public Object proceed() throws Throwable {
    // 如果已经运行到了最后一个拦截器，那么直接调用对应的方法即可
    if (this.currentInterceptorIndex == this.interceptorsAndDynamicMethodMatchers.size() - 1) {
        return invokeJoinpoint();
    }

    // 获取当前拦截器
    Object interceptorOrInterceptionAdvice =
            this.interceptorsAndDynamicMethodMatchers.get(++this.currentInterceptorIndex);
    // 如果当前拦截器是一个动态拦截器，那么需要每次都判断是否起作用
    if (interceptorOrInterceptionAdvice instanceof InterceptorAndDynamicMethodMatcher) {
        // Evaluate dynamic method matcher here: static part will already have
        // been evaluated and found to match.
        InterceptorAndDynamicMethodMatcher dm =
                (InterceptorAndDynamicMethodMatcher) interceptorOrInterceptionAdvice;
        Class<?> targetClass = (this.targetClass != null ? this.targetClass : this.method.getDeclaringClass());
        // 判断当前拦截器是否对当前方法起作用，如果起作用就调用这个拦截器
        if (dm.methodMatcher.matches(this.method, targetClass, this.arguments)) {
            return dm.interceptor.invoke(this);
        }
        // 否则说明当前拦截器对当前方法无效，递归的调用下一个拦截器
        else {
            return proceed();
        }
    }
    // 如果当前拦截器是静态的，那么直接调用即可，无需重复判断
    else {
        return ((MethodInterceptor) interceptorOrInterceptionAdvice).invoke(this);
    }
}
```

##### CGLIB代理

实际上我们可以看到，在`DefaultAopProxyFactory`类中的`createAopProxy`方法，在需要创建CGLIB代理时创建了一个`ObjenesisCglibAopProxy`对象，它实际上是CGLIB的一个包装类，继承了`CglibAopProxy`类，其大多数功能是由`CglibAopProxy`类完成的。同样我们先来看它实现的`getProxy`方法：

```java
public Object getProxy(@Nullable ClassLoader classLoader) {
    try {
        // 获得目标类
        Class<?> rootClass = this.advised.getTargetClass();
        Assert.state(rootClass != null, "Target class must be available for creating a CGLIB proxy");

        Class<?> proxySuperClass = rootClass;
        // 如果当前类已经是一个被CGLIB代理的类，那么需要获得它实际的代理类
        if (rootClass.getName().contains(ClassUtils.CGLIB_CLASS_SEPARATOR)) {
            proxySuperClass = rootClass.getSuperclass();
            // 获得当前类实现的接口，所有获取这些接口实例时，都需要经过AOP代理
            Class<?>[] additionalInterfaces = rootClass.getInterfaces();
            for (Class<?> additionalInterface : additionalInterfaces) {
                this.advised.addInterface(additionalInterface);
            }
        }
        // 验证类
        validateClassIfNecessary(proxySuperClass, classLoader);

        // 创建CGLIB代理
        Enhancer enhancer = createEnhancer();
        if (classLoader != null) {
            enhancer.setClassLoader(classLoader);
            if (classLoader instanceof SmartClassLoader &&
                    ((SmartClassLoader) classLoader).isClassReloadable(proxySuperClass)) {
                enhancer.setUseCache(false);
            }
        }
        // proxySuperClass就是实际上要被代理的类
        enhancer.setSuperclass(proxySuperClass);
        enhancer.setInterfaces(AopProxyUtils.completeProxiedInterfaces(this.advised));
        enhancer.setNamingPolicy(SpringNamingPolicy.INSTANCE);
        enhancer.setStrategy(new ClassLoaderAwareGeneratorStrategy(classLoader));

        // 获取回调函数，这里的回调函数就是拦截器的拦截方法
        Callback[] callbacks = getCallbacks(rootClass);
        Class<?>[] types = new Class<?>[callbacks.length];
        for (int x = 0; x < types.length; x++) {
            types[x] = callbacks[x].getClass();
        }
        enhancer.setCallbackFilter(new ProxyCallbackFilter(
                this.advised.getConfigurationOnlyCopy(), this.fixedInterceptorMap, this.fixedInterceptorOffset));
        enhancer.setCallbackTypes(types);

        // 返回创建的代理实例
        return createProxyClassAndInstance(enhancer, callbacks);
    }
    catch (CodeGenerationException | IllegalArgumentException ex) {
        throw new AopConfigException("Could not generate CGLIB subclass of " + this.advised.getTargetClass() +
                ": Common causes of this problem include using a final class or a non-visible class",
                ex);
    }
    catch (Throwable ex) {
        // TargetSource.getTarget() failed
        throw new AopConfigException("Unexpected AOP exception", ex);
    }
}
```

很简单，主要的逻辑就是利用CGLIB来创建代理对象，而拦截器就是作为CGLIB的回调方法被织入的。这里关于回调方法获取的`getCallbacks`方法比较复杂，这里就不分析这个方法了，只要知道CGLIB是利用这个回调方法来实现AOP注入的即可，至于具体实现的细节就不分析了。

CGLIB在`getCallbacks`方法中构造了`DynamicAdvisedInterceptor`实现拦截回调功能，我们看`getCallbacks`中的这行代码：

```java
Callback aopInterceptor = new DynamicAdvisedInterceptor(this.advised);
```

实际上aop拦截器会作为第一个拦截方法被加入回调函数数组中，而我们接下来来看`DynamicAdvisedInterceptor`类是如何完成拦截的，所有的拦截功能都在`intercept`方法中：

```java
public Object intercept(Object proxy, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
    Object oldProxy = null;
    boolean setProxyContext = false;
    Object target = null;
    TargetSource targetSource = this.advised.getTargetSource();
    try {
        if (this.advised.exposeProxy) {
            // 暴露对象
            oldProxy = AopContext.setCurrentProxy(proxy);
            setProxyContext = true;
        }
        // 尽可能短时间的持有这个目标对象，因为如果是对象池中的对象，那么持有对象会占用资源
        target = targetSource.getTarget();
        Class<?> targetClass = (target != null ? target.getClass() : null);
        // 获得拦截器链
        List<Object> chain = this.advised.getInterceptorsAndDynamicInterceptionAdvice(method, targetClass);
        Object retVal;
        // 如果没有拦截器，并且调用的是一个公有方法，那么直接使用反射调用即可
        if (chain.isEmpty() && Modifier.isPublic(method.getModifiers())) {
            Object[] argsToUse = AopProxyUtils.adaptArgumentsIfNecessary(method, args);
            retVal = methodProxy.invoke(target, argsToUse);
        }
        else {
            // 否则则交给CglibMethodInvocation代理执行，执行的流程由CGLIB负责完成
            retVal = new CglibMethodInvocation(proxy, target, method, args, targetClass, chain, methodProxy).proceed();
        }
        // 返回值的判定和JDK动态代理一样，要判断this的指向问题以及基本类型的空值问题
        retVal = processReturnType(proxy, target, method, retVal);
        return retVal;
    }
    finally {
        if (target != null && !targetSource.isStatic()) {
            targetSource.releaseTarget(target);
        }
        if (setProxyContext) {
            // Restore old proxy.
            AopContext.setCurrentProxy(oldProxy);
        }
    }
}

// 和JDK动态代理套路一样，只不过这里封装了一下而已
private static Object processReturnType(Object proxy, @Nullable Object target, Method method, @Nullable Object returnValue) {
	if (returnValue != null && returnValue == target &&
			!RawTargetAccess.class.isAssignableFrom(method.getDeclaringClass())) {
		returnValue = proxy;
	}
	Class<?> returnType = method.getReturnType();
	if (returnValue == null && returnType != Void.TYPE && returnType.isPrimitive()) {
		throw new AopInvocationException(
				"Null return value from advice does not match primitive return type for: " + method);
	}
	return returnValue;
}
```

可以看到整个拦截器执行的逻辑和JDK动态代理大同小异，不同点就在于JDK动态代理交给`ReflectiveMethodInvocation`完成，而CGLIB则交给了`CglibMethodInvocation`来完成，而`CglibMethodInvocation`则由CGLIB来负责完成具体的代理流程，而拦截过程则大同小异，这里就不再分析了。

##### 流程总结

好了以上就是利用`ProxyFactoryBean`创建AOP代理的流程，我们总结一下就是：

1. 首先我们需要定义一个Advice，通用的做法是使用默认的MethodBeforeAdvice、AfterReturningAdvice以及ThrowsAdvice三个因为这三个Advice，Spring已经提供了默认的适配器Adapter，可以将Advice适配到MethodInterceptor接口上。当然我们也可以自己定义Advice供后续复用，但是如果自己定义Advice，那么就需要同时将Adapter作为Bean装配到IoC容器中；或者我们也可以直接实现MethodInterceptor接口，通过继承AbstractGenericPointcutAdvisor类可以直接将Pointcut和Advice以及MethodInterceptor的功能整合。

我们下面来看这两个例子：

第一个例子，使用预定义的接口MethodBeforeAdvice：

```java
public class TestAdvisor implements MethodBeforeAdvice {
    @Override
    public void before(Method method, Object[] args, Object target) throws Throwable {
        System.out.printf("Before method [%s], TestAdvisor is running\n", method.getName());
    }
}
```

这样子我们只需要配置bean即可：

```xml
<bean id="testAdvisor" class="com.learn.spring.chcp3.TestAdvisor" />
<bean id="proxyFactoryBean" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="proxyInterfaces" value="com.learn.spring.chcp3.pojo.ProxyInterfaces"/>
    <property name="target">
        <bean class="com.learn.spring.chcp3.pojo.ProxyInterfacesImpl" lazy-init="true"/>
    </property>
    <property name="interceptorNames">
        <list>
            <value>testAdvisor</value>
        </list>
    </property>
</bean>
```

就可以直接实现对方法运行的拦截功能。同样我们也可以直接定义一个Advisor：

```java
public class FullAdvisor extends AbstractGenericPointcutAdvisor implements Serializable {
    private Pointcut pointcut;

    public FullAdvisor() {
        // 配置切入点，核心就在于重写ClassFilter来匹配类，以及重写MethodMatcher来匹配方法
        this.pointcut = new Pointcut() {
            @Override
            public ClassFilter getClassFilter() {
                return clazz -> clazz.equals(MyObject.class);
            }

            @Override
            public MethodMatcher getMethodMatcher() {
                return new MethodMatcher() {
                    @Override
                    public boolean matches(Method method, Class<?> targetClass) {
                        return targetClass.equals(MyObject.class) && "run".equalsIgnoreCase(method.getName());
                    }

                    @Override
                    public boolean isRuntime() {
                        return false;
                    }

                    @Override
                    public boolean matches(Method method, Class<?> targetClass, Object... args) {
                        throw new UnsupportedOperationException();
                    }
                };
            }
        };
        // 配置Advice，来确定方法执行的流程，这里实现的是一个AroundAdvice
        setAdvice((MethodInterceptor) invocation -> {
            System.out.println("Before custom advice");
            Object ret = invocation.proceed();
            System.out.println("After custom advice");
            return ret;
        });
    }

    public FullAdvisor(Advice advice) {
        this();
        setAdvice(advice);
    }

    public void setPointcut(Pointcut pointcut) {
        this.pointcut = pointcut;
    }

    @Override
    public Pointcut getPointcut() {
        return pointcut;
    }
}

```

同样也需要配置Bean：

```xml
<bean id="fullAdvisor" class="com.learn.spring.chcp3.FullAdvisor" />
<bean id="fullFactoryBean" class="org.springframework.aop.framework.ProxyFactoryBean">
    <property name="target">
        <bean class="com.learn.spring.chcp3.pojo.MyObject" />
    </property>
    <property name="interceptorNames">
        <list>
            <value>fullAdvisor</value>
        </list>
    </property>
</bean>
```

即可。

2. 我们从上面的xml中可以看到，AOP代替原始对象发挥作用实际上是通过一个FactoryBean实现的，当我们要获取比如上面的第一个ProxyInterfaces和ProxyInterfacesImpl对象时，实际上就由对应的ProxyFactoryBean来生成代理对象。而生成对象最核心的方法就是FactoryBean接口中的getObject函数。

3. getObject函数会负责初始化拦截器链并生成单例对象，单例对象实际上是委托给DefaultAopProxyFactory的createAopProxy方法来生成的，在createAopProxy方法中会判断到底应该使用JDK动态代理还是CGLIB代理。

   JDK动态代理的实现类JdkDynamicAopProxy实际上实现了AopProxy接口，本身会负责代理对象的封装工作。封装是通过getProxy方法实现的，getProxy方法会负责将代理接口、拦截方法等组合起来生成最终的代理对象，由于JdkDynamicAopProxy还实现了InvocationHandler接口，所以JdkDynamicAopProxy类本身就是拦截处理器，拦截时调用invoke方法即可。

   CGLIB代理的实现类是ObjenesisCglibAopProxy，但是大多数工作都在它的父类CglibAopProxy中实现了，CglibAopProxy同样实现了AopProxy接口，代理生成方式也是getProxy方法，基本流程也一致，只不过是使用了CGLIB来生成代理对象罢了。在方法调用的时候实际上是通过回调方法的形式来调用AOP方法，其实现是通过创建一个DynamicAdvisedInterceptor类型的回调来实现的，拦截时调用intercept方法即可。

4. 当单例对象创建之后就会通过getBean方法返回给调用者。此时调用者收到的Bean对象就是代理对象了。调用代理对象是就会调用对应的拦截方法来实现AOP功能。

#### ProxyFactory

ProxyFactory就是一种编程式的方式而不是使用ProxyFactoryBean自动去管理AOP对象，ProxyFactory由于采用了编程式的实现，所以整个流程就是把ProxyFactoryBean那种自动配置改成了编程配置，下面看一个具体代码：

```java
@Test
public void testProxyFactory() {
    ProxyFactory pf = new ProxyFactory();
    pf.setTarget(new MyObject2());
    FullAdvisor ad = new FullAdvisor();
    ad.setPointcut(Pointcut.TRUE);
    pf.addAdvisor(ad);
    MyObject2 obj = (MyObject2) pf.getProxy(this.getClass().getClassLoader());
    obj.hello();
}
```

可以看到完全采用编程的方式配置了AOP。整个AOP代理的创建流程和前面的ProxyFactoryBean一致。

#### ExposeProxy问题

可以看这一篇：

> https://blog.csdn.net/z69183787/article/details/83186774

ExposeProxy要解决的问题就是如果一个被代理的类中的一个方法调用了另一个方法，那么AOP只会发生一次。例如：

```java
public class ProxyInterfacesImpl implements ProxyInterfaces{
    @Override
    public void say() {
        System.out.println("This is say method in ProxyInterfacesImpl");
    }

    @Override
    public void bye() {
        System.out.println("This is bye method in ProxyInterfacesImpl");
        say();
    }
}
```

例如这里的bye方法，它调用了say方法，但是如果我们调用bye方法时可以看到，AOP代理只在执行bye方法的时候执行了一遍，而在调用say方法时没有触发AOP代理的拦截方法。这里的理解很简单，就是对于调用bye方法而言，它是通过动态代理的invoke或intercept方法执行的，而拦截的流程都在invoke和intercept方法之中（此时的this对象指的是代理对象）；而对于bye调用的say方法，它是由bye方法执行的，此时的this对象是被代理的目标类，所以就导致了在执行say方法时没有触发AOP拦截。而如果使用了ExposeProxy，那么会将当前的AOP代理对象放置到AOP上下文中，在每次调用方法的时候都使用这个AOP上下文对象（即代理对象）来调用方法，这样就可以使得调用say的时候以代理类的身份调用它了，从而也就能触发AOP拦截了。