---
layout: post
title: Tomcat架构原理和Spring请求流程
description: Tomcat整体架构分析、Tomcat类加载器分析、Tomcat初始化流程以及结合Spring来看整个请求从浏览器发出到Controller处理的流程介绍
categories: Spring
keywords: Spring, Tomcat
---

# Tomcat架构原理和Spring请求流程

## Http请求流程

Http是浏览器和服务器之间的数据传送协议，Http协议作为应用层协议，是基于TCP/IP协议来传递数据的，Http不涉及数据包的传输，主要规定了客户端和服务端之间的通信格式。下面是Http请求的流程图：

<img src="images/Tomcat/Tomcat-Http-Request-Process.png" alt="Tomcat-Http-Request-Process" style="zoom:50%;" />

在用户发起请求时，浏览器首先会和服务器建立TCP连接（三次握手），之后会生成HTTP格式的数据包，通过TCP/IP协议发送数据到服务器，服务器解析HTTP格式的数据包并执行对应的请求，之后返回响应数据包到浏览器，浏览器解析数据包后呈现HTTP响应给用户。

Tomcat在收到HTTP请求之后会将请求交给Servlet来进行进一步的处理。但是在交给具体的Servlet之前，Tomcat会负责将TCP请求转换为HTTP请求，并且适配请求，生成ServletRequest对象来描述请求信息交付给Servlet使用；当Servlet处理结束后，需要返回的数据以ServletResponse抽象的形式交付给Tomcat，Tomcat同样将其适配为Response，并生成HTTP数据包，通过TCP协议发送消息。下面这张图就是Tomcat具体处理HTTP请求的流程：

<img src="images/Tomcat/Tomcat-Tomcat-Handle-Http-Request-Process.png" alt="Tomcat-Tomcat-Handle-Http-Request-Process" style="zoom:50%;" />

## Tomcat整体架构

Tomcat实现的两大核心功能分别是：

1. 处理Socket连接，负责网络字节流与Request和Response对象的转化；
2. 加载和管理Servlet，以及处理具体的Request请求。

为了实现这两大功能，Tomcat设计了两个核心组件连接器（Connector）和容器（Container）来分别实现上述两大功能。连接器负责对外交流，容器负责内部处理。逻辑结构如下：

<img src="images/Tomcat/Tomcat-Core-Function-Arch.png" alt="Tomcat-Core-Function-Arch" style="zoom:50%;" />

可以看到，Tomcat Server是唯一的，一个Server中可以有多个Service，一个Service中可以有多个Connector（目的是为了监听多个端口，实现多个网站的同时部署），但是每个Service中只能有一个Container。上面这个结构和Tomcat的配置文件server.xml是吻合的（Container这里实际上又细分为Engine、Host、Context和Wrapper）：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<Server port="8005" shutdown="SHUTDOWN">
  <Listener className="org.apache.catalina.startup.VersionLoggerListener" />
  <Listener className="org.apache.catalina.core.AprLifecycleListener" SSLEngine="on" />
  <Listener className="org.apache.catalina.core.JreMemoryLeakPreventionListener" />
  <Listener className="org.apache.catalina.mbeans.GlobalResourcesLifecycleListener" />
  <Listener className="org.apache.catalina.core.ThreadLocalLeakPreventionListener" />

  <GlobalNamingResources>
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="conf/tomcat-users.xml" />
  </GlobalNamingResources>

  <Service name="Catalina">
    <Connector port="8080" protocol="HTTP/1.1"
               connectionTimeout="20000"
               redirectPort="8443" />
    <Engine name="Catalina" defaultHost="localhost">
      <Realm className="org.apache.catalina.realm.LockOutRealm">
        <Realm className="org.apache.catalina.realm.UserDatabaseRealm"
               resourceName="UserDatabase"/>
      </Realm>
      <Host name="localhost"  appBase="webapps"
            unpackWARs="true" autoDeploy="true">
        <Valve className="org.apache.catalina.valves.AccessLogValve" directory="logs"
               prefix="localhost_access_log" suffix=".txt"
               pattern="%h %l %u %t &quot;%r&quot; %s %b" />
      </Host>
    </Engine>
  </Service>
</Server>
```

### Coyote连接器

Tomcat中的连接器名为Coyote，是Tomcat服务器提供的供客户端访问的外部接口。客户端通过Coyote与服务器建立连接、发送请求并接受响应。

Coyote封装了底层的网络通信（Socket请求及响应处理），为Catalina容器提供了统一的接口，使Catalina容器与具体的请求协议以及IO操作方式完全解耦。Coyote将Socket输入转换封装为Request对象，交由Catalina容器进行处理，处理请求完成后，Catalina通过Coyote提供的Response对象将结果写入输出流。要注意这里的Request和Response与Servlet没有任何关系，只是Coyote对网络请求与响应的抽象，而如果要转换成Servlet可以使用的ServletRequest以及ServletResponse，中间需要经过适配器进行适配。下面就是Coyote和Catalina交互的示意图：

<img src="images/Tomcat/Tomcat-Coyote-Catalina-Communication-Arch.png" alt="Tomcat-Coyote-Catalina-Communication-Arch" style="zoom:50%;" />

由于Coyote的工作主要负责Socket的读写操作，以及将TCP请求适配为HTTP请求，所以就比如存在多种适配关系，例如IO操作中的BIO（已被移除）、NIO、AIO（NIO2）、APR（需要独立运行库，但是能大幅度提高性能）等；网络协议包括HTTP/1.1、HTTP/2、AJP（用于和Web服务器集成，以实现对静态资源的优化以及集群部署）。

所以将上述两类协议分层，就能得到Coyote内部的核心组件了：

应用层（Processor）：HTTP/1.1	AJP	HTTP/2

传输层（Endpoint）：NIO	NIO2	APR

<img src="images/Tomcat/Tomcat-Coyote-Detail-Arch.png" alt="Tomcat-Coyote-Detail-Arch" style="zoom:50%;" />

由于Endpoint和Processor有多种不同的组合方式，所以额外定义了一个ProtocolHandler来对其进行包装，比较常用的ProtocolHandler包括`Http11NioProtocal`、`Http11Nio2Protocal`、`Http11AprProtocal`等。ProtocalHandler输出的结果即Request类型对象，经过`CoyoteAdapter`的适配，最终转换为了可以被Catalina容器识别的ServletRequest对象。

接着来具体解释一下Coyote中各个部件的作用：

#### Endpoint

1. Eodpoint是Coyote的通信端点，即通信监听的接口，是具体Socket接收和发送处理器，**是对传输层的抽象**，因此Endpoint**是用来实现TCP/IP协议的**。
2. Tomcat实现Endpoint时提供了一个抽象的`AbstractEndpoint`类，里面定义了两个内部类`Acceptor`和`SocketProcessor`，`Acceptor`用来监听Socket请求，`SocketProcessor`用于处理接收到的Socket请求，它实现了`Runnable`接口，实际上是由线程池调用进行处理的，在处理时就会调用`Processor`组件进行处理。

#### Processor

Processor是Coyote的协议处理接口，如果说Endpoint是用来实现TCP/IP协议的，那么Processor就是用来实现HTTP协议的，Processor接收来自Endpoint的Socket，读取字节流并解析成Tomcat Request和Response对象，并通过Adapter提交到容器进行处理，**Processor是对应用层的抽象**。

#### ProtocalHandler

Coyote协议接口，通过Endpoint和Processor，实现针对具体协议的处理能力。Tomcat 按照协议和I/O 提供了6个实现类 ：`AjpNioProtocol` ，` AjpAprProtocol`， `AjpNio2Protocol` ， `Http11NioProtocol` ，`Http11Nio2Protocol`， `Http11AprProtocol`。我们在配置tomcat/conf/server.xml 时 ， 至少要指定具体的 ProtocolHandler , 当然也可以指定协议名称 ， 如 ： HTTP/1.1 ，如果安装了APR，那么 将使用Http11AprProtocol ， 否则使用 Http11NioProtocol 。

#### Adapter

由于协议不同，客户端发过来的请求信息也不尽相同，Tomcat定义了自己的Request类来“存放”这些请求信息。ProtocolHandler接口负责解析请求并生成Tomcat Request类。 但是这个Request对象不是标准的ServletRequest，也就意味着，不能用Tomcat Request作为参数来调用容器。Tomcat设计者的解决方案是引入`CoyoteAdapter`，这是适配器模式的经典运用，连接器调用CoyoteAdapter的Sevice方法，传入的是Tomcat Request对象，CoyoteAdapter负责将Tomcat Request转成ServletRequest，再调用容器的Service方法。

### Catalina容器

Tomcat 本质上就是一款 Servlet 容器， 因此Catalina 才是 Tomcat 的核心 ， 其他模块 都是为Catalina 提供支撑的。 比如 ： 通过Coyote 模块提供链接通信，Jasper 模块提供 JSP引擎，Naming 提供JNDI 服务，Juli 提供日志服务：

<img src="images/Tomcat/Tomcat-Catalina-Arch.png" alt="Tomcat-Catalina-Arch" style="zoom:50%;" />

Catalina容器的主要组件如下：

<img src="images/Tomcat/Tomcat-Catalina-Total-Container-Arch.png" alt="Tomcat-Catalina-Total-Container-Arch" style="zoom:50%;" />

如上图所示，Catalina负责管理Server，而Server表示着整个服务器。Server下面有多个服务Service，每个服务都包含着多个连接器组件Connector（Coyote 实现）和一个容器组件Container。在Tomcat 启动的时候，会初始化一个Catalina的实例。我们将Connector和Container合起来称为Service，这个Service也仅仅只是对二者进行一个包装，并没有任何实际的作用。

Catalina内部设计了四种容器：Engine、Host、Context和Wrapper，Tomcat正是通过这种分层的架构，让Servlet容器具备了更好的灵活性。

<img src="images/Tomcat/Tomcat-Catalina-Total-Container-Arch.png" alt="Tomcat-Catalina-Total-Container-Arch" style="zoom:50%;" />

各个组件的作用如下：

| 容器    | 描述                                                         |
| ------- | ------------------------------------------------------------ |
| Engine  | 表示整个Catalina的Servlet引擎，用来管理多个虚拟站点，一个Service 最多只能有一个Engine，但是一个引擎可包含多个Host。 |
| Host    | 代表一个虚拟主机，或者说一个站点，可以给Tomcat配置多个虚拟主机地址，而一个虚拟主机下可包含多个Context |
| Context | 表示一个Web应用程序， 一个Web应用可包含多个Wrapper           |
| Wrapper | 表示一个Servlet，Wrapper 作为容器中的最底层，不能包含子容器  |

这里也就理清了Tomcat、Servlet容器、Servlet之间的关系：Tomcat是最顶层的Server，它包含一个Catalina容器（即Engine），而一个Catalina容器内又可以有多个Host用来表示不同的站点，每个站点都可以启动一个Web应用，一个Context就代表一个Web应用，每个Web应用都有自己的全局ServletContext，一个Web应用内部也可以有若干个Servlet，每个独立的ServletContext就用Wrapper表示。

## Tomcat类加载器模型

Tomcat的类加载器模型是一种典型的破坏双亲委派模型的类加载器模型，它的破坏主要体现在对于每个Web应用而言，它们需要的类库版本可能并不一致（例如Web应用1需要Spring4.0，Web应用B需要Spring5.0），那么如果采用双亲委派模型，所有类都尝试由顶层加载，那么非常可能出现由于一个类已经存在（全限定名一致）而导致另一个版本的类无法被加载的问题。所以Tomcat的类加载器被设计为每个Web项目由自己的类加载器来加载所需的类库，Web项目间不共享，这就打破了原有的顶层父类加载的双亲委派模型了，下面来看具体的结构：

<img src="images/Tomcat/Tomcat-ClassLoader-Arch.png" alt="Tomcat-ClassLoader-Arch" style="zoom:50%;" />

可以看到，前三个类加载器和Java中的类加载器结构一致，目的就是为了加载依赖于Java语言的公共类库，这部分类库不应该在每个Web应用中都存储一个副本，这样会导致虚拟机内存的膨胀。后面的CommonClassLoader、CatalinaClassLoader、SharedClassLoader、WebappClassLoader、JasperLoader则是Tomcat定义的类加载器，它们分别加载不同路径下的类库：

- CommonClassLoader：Tomcat最基本的类加载器，加载/common/路径下的类库，加载路径中的class可以被Tomcat容器本身以及各个Webapp访问；
- CatalinaClassLoader：Tomcat容器私有的类加载器，加载/server/路径下的类库，加载路径中的class对于Webapp不可见；
- SharedClassLoader：各个Webapp共享的类加载器，加载/shared/路径下的类库（已经合并到lib中了），加载路径中的class对于所有Webapp可见，但是对于Tomcat容器不可见；
- WebappClassLoader：各个Webapp私有的类加载器，加载/WEB-INF/lib/路径下的类库，加载路径中的class只对当前Webapp可见；
- JasperLoader：是每一个Jsp文件私有的类加载器，目的是为了实现热加载，当Jsp文件发生改变后就立即丢弃创建一个新的类加载器来进行加载。

> CommonClassLoader能加载的类都可以被Catalina ClassLoader和SharedClassLoader使用，从而实现了公有类库的共用，而CatalinaClassLoader和SharedClassLoader自己能加载的类则与对方相互隔离。
>
> WebAppClassLoader可以使用SharedClassLoader加载到的类，但各个WebAppClassLoader实例之间相互隔离。
>
> 而JasperLoader的加载范围仅仅是这个JSP文件所编译出来的那一个.Class文件，它出现的目的就是为了被丢弃：当Web容器检测到JSP文件被修改时，会替换掉目前的JasperLoader的实例，并通过再建立一个新的Jsp类加载器来实现JSP文件的HotSwap功能。

上面是早期版本的Tomcat类加载机制，虽然是早期版本，但是仍然有很强的借鉴意义，而由于Tomcat目录结构的变化，整个类加载模型也有了一点小变化，但是这不是重点，重点是Tomcat破坏了双亲委派模型的原因。我们同样给出新的Tomcat类加载器架构图：

<img src="images/Tomcat/Tomcat-ClassLoader-New-Arch.png" alt="Tomcat-ClassLoader-New-Arch" style="zoom:80%;" />

**Bootstrap 引导类加载器 ：**加载JVM启动所需的类，以及标准扩展类（位于jre/lib/ext下），将JVM前三个类加载器概括到一起了。

**System** **系统类加载器** **：**加载tomcat启动的类，比如bootstrap.jar，通常在catalina.bat或者catalina.sh中指定。位于CATALINA_HOME**/bin**下。

**Common** **通用类加载器：**加载tomcat使用以及应用通用的一些类，位于CATALINA_HOME**/lib**下，比如servlet-api.jar。

**webapp应用类加载器**：每个应用在部署后，都会创建一个唯一的类加载器。该类加载器会加载位于 WEB-INF/lib下的jar文件中的class和 WEB-INF/classes下的class文件。

**JSP类加载器** **：**tomcat会为每一个JSP创建一个类加载器。

## Tomcat处理Http请求的流程

### Tomcat初始化流程

Tomcat的初始化流程和它的父子层级关系接近一致，初始化的流程也是按照这个顺序进行的，由当前组件负责初始化它的子组件，最顶层的入口为Bootstrap类：

![Tomcat-StartupAndInit-Process](images/Tomcat/Tomcat-StartupAndInit-Process.png)

Bootstrap首先构造Classloader，接着调用CatalinaClassloader加载和反射创建Catalina对象，接着Bootstrap调用load方法开始逐级加载：首先利用反射调用Catalina的load方法，Catalina首先创建唯一的Server对象实例，在创建前会读取server.xml文件并解析内容，根据配置内容来配置Server实例，配置结束后则调用Server的init方法。

要注意，Tomcat在这里为所有组件定义了一个统一的生命周期接口`Lifecycle`，所有基本组件例如`Connector`、`StandardServer`、`StandardService`、`StandardEngine`、`StandardContext`等都继承了`LifecycleBase`类（`Lifecycle`接口的一个实现类），通过统一的方法逻辑来实现调用。

在Tomcat中Server默认使用的是`StandardServer`类，它的init方法根据配置文件的定义，可以初始化若干个`StandardService`实例，同样通过调用其继承的init方法，而`StandardService`类则顺序初始化一个`StandardEngine`、若干个线程池`Executor`以及`Connector`以及若干个`Connector`，Engine还会负责其层次关系下级的Host、Context的初始化。

而Connector则会继续初始化ProtocolHandler，这里会根据配置文件的配置，选择对应的Handler，例如`AbstractHttp11ProtocolHandler`，而它们实际上在`AbstractProtocolHandler`中继续初始化Endpoint，Endpoint则在此时设置监听的端口、域名等信息，最后打开ServerSocket，等待accept请求。由于Tomcat只实现了抽象的`AbstractEndpoint`类，所以究竟使用哪个实例，取决于配置，默认使用`NioEndpoing`，当然也可以配置使用`Nio2Endpint`和`AprEndpoint`，由这些具体的Endpoint来负责实际的请求监听工作。

在上述初始化过程结束后，Bootstrap会调用其start方法开启所有服务器服务，这个开启过程也是逐级开启的，主要也是依赖Lifecycle接口中定义的start方法，以及LifecycleBase实现的部分方法来实现的。中间这些启动过程就不分析了，这里重点来看Connector的启动流程：Connector会启动ProtocolHandler，以HTTP为例，ProtocolHandler会调用其实现了`AbstractProtocolHandler`来启动对应的Endpoint，以NioEndpoint为例:

NioEndpint为了实现多路复用，内部包含三个组件，一个是Acceptor线程组用来接收请求，这个大家都一样；第二个是Poller线程组，用于监听 Socket 事件，当 Socket 可读或可写时，将 Socket 封装一下添加到 worker 线程池的任务队列中；最后一个就是Worker线程组。用于对请求进行处理，包括分析请求报文并创建 Request 对象，调用容器的 pipeline 进行处理（也就是上面架构中说过的Processor）。具体的图示为：

![Tomcat-NioEndpoint-Handle-Process](images/Tomcat/Tomcat-NioEndpoint-Handle-Process.jpeg)

所以在Init阶段，NioEndpoint会初始化Acceptor、Poller以及Worker（即前面说的Executor），启动顺序是Poller、Executor、Acceptor。Acceptor独立在另一个线程中启动，启动后就开始进行监听，也即监听线程会阻塞在ServerSocketChannel的accept方法处。

### Tomcat的请求流程

Tomcat的初始化并不是重点，下面我们结合源码和Spring来看一下，一个请求是怎么样从TCP请求到达Servlet的：

为了实现将请求交给具体的Servlet处理，Tomcat设计了Mapper组件，其类似于一个多层次的Map，保存了访问路径到容器组件之间的映射关系，当一个请求到来时，Mapper通过解析请求域名和路径，再到自己保存的Map中去寻找就能找到对应的Servlet了：

![Tomcat-Request-Process](images/Tomcat/Tomcat-Request-Process.png)

如果使用的是Spring的Controller，那实际上具体的请求是交给DispatcherServlet进行处理的，但是DispatcherServlet处理的请求范围需要在web.xml中的servlet-mapping中配置，这个配置就是上图中Wrapper匹配的路径了。

上面的这张图从URL请求的角度分析了Http请求到达Servlet的流程，下面从架构的流程上来分析一下：

![Tomcat-Request-Handle-Process](images/Tomcat/Tomcat-Request-Handle-Process.png)

#### Acceptor

我们从Tomcat初始化流程中的启动流程知道，请求接收实际上发生在Acceptor中的run方法里（实现了Runnable接口，实际由对应Endpoint创建新线程，在线程中阻塞等待）。先来看Acceptor处理请求的时序图：

![Tomcat-Acceptor-NioEndpoint-Handle-Process](images/Tomcat/Tomcat-Acceptor-NioEndpoint-Handle-Process.png)

当Acceptor收到请求时，Acceptor所在线程就不再阻塞，ServerSocket返回一个ServerSocketChannel对象，接着Acceptor会调用Endpoint中的`setSocketOptions`方法来通知对应的Endpoint接收和处理数据，以NioEndpoint为例，`setSocketOptions`会将Socket封装到NioChannel中，并注册到Poller中，由于启动过程中启动了不止一个Poller，所以这里注册的过程采用的是轮询的方法，公平地分配给每一个Poller。而Poller收到注册请求后，会构建流到达事件，并将其加入到给Poller的PollerEvent队列中，到此为止Acceptor的任务就完成了。此时如果Acceptor没有收到停止指令，那么就会继续阻塞，等待下一个请求到来。

#### Poller

Poller执行的时序图如下：

![Tomcat-Poller-Handle-Request-Process](images/Tomcat/Tomcat-Poller-Handle-Request-Process.png)

1. Poller的启动在NioEndpoint中，Poller自身实现了Runnable接口，而在NioEndpoint中为每个Poller新建了一个线程来执行。Poller线程会首先会检查当前PollerEvent队列中是否存在未处理的事件，如果存在则调用events方法进行处理，如果不存在则会调用内部Selector的select方法进行阻塞，直到Selector中有可用Channel时才会从阻塞中恢复（所有Poller共用一个Selector，其实现类是 sun.nio.ch.EPollSelectorImpl）。当存在事件时会调用events方法进行处理。

   这里具体实现的时候是通过让select每阻塞1秒来进行检查的，检查过程就是一个死循环，直到检测到停止标志时才会退出这个死循环。

2. events方法会将通过 addEvent() 方法添加到事件队列中的 Socket 注册到 EPollSelectorImpl，当 Socket 可读时，Poller 才对其进行处理。

3. 当Poller中的select收到数据时，会调用processKey方法依次处理每个就绪的数据，createSocketProcessor() 方法将 Socket 封装到 SocketProcessor 中，SocketProcessor 实现了 Runnable 接口。worker 线程通过调用其 run() 方法来对 Socket 进行处理。

4. execute(SocketProcessor) 方法将 SocketProcessor 提交到线程池，放入线程池的 workQueue 中。workQueue 是 BlockingQueue 的实例。到此 Poller 的任务就完成了。

从上面这两个部分可以看出，Acceptor主要是用于接收连接请求，当Acceptor收到连接请求后就将其封装为PollerEvent放入到Poller的时间队列中，Poller则将添加的这个事件加入到EPOLL监听队列（红黑树）中，每隔1秒会检查依次PollerEvent队列是否有新的事件（连接时间不是IO事件）（**这里要明确select设置了默认超时时间1秒，在1秒内如果selector没有收到数据，那么会因为超时返回，此时会判断是否有可读写的数据，如果没有再进入events方法判断有没有新的连接请求要处理；而如果1秒内selector收到了数据，那么会直接返回，此时会先处理selector收到的数据，再重新调用events方法判断有没有新的连接到来，这是两个执行流程，但无论如何最迟1秒都会检查一次PollerEvent队列**）。当selector收到了数据之后，会交给下一阶段的Executor（worker）进行处理。

**总而言之，Acceptor的职责是负责处理连接时间，Poller的职责是监听IO完成事件，Worker则进一步对IO获取的数据进行处理。**

#### Worker

![Tomcat-Worker-Handle-Request](images/Tomcat/Tomcat-Worker-Handle-Request.jpeg)

这里的Executor实际也在NioEndpoint中创建，类型是ThreadPoolExecutor，内部是以一个LinkedBlockingQueue的形式存储的。ThreadPoolExecutor的原理这里就不解释了，内部维护了一个WorkerQueuu即工作队列，工作队列中的每个Worker启动后由于没有任务则在runWorker方法中阻塞。Worker实际执行的就是Poller向线程池提交的SocketProcessor任务：

- createProcessor() 会创建一个 Http11Processor, 它用来解析 Socket，将 Socket 中的内容封装到 Request 中。注意这个 Request 是临时使用的一个类，它的全类名是 org.apache.coyote.Request，接着将这个Coyote的Request传递给CoyoteAdapter，由CoyoteAdapter将org.apache.coyote.Request封装到 org.apache.catalina.connector.Request；

- connector.getService().getMapper().map()，用来在 Mapper 中查询 URL 的映射关系。映射关系会保留到 org.apache.catalina.connector.Request 中，Container 处理阶段 request.getHost() 是使用的就是这个阶段查询到的映射主机，以此类推 request.getContext()、request.getWrapper() 都是使用的这里解析出来的映射关系。

- connector.getService().getContainer().getPipeline().getFirst().invoke() 会将请求传递到 Container 处理，当然了 Container 处理也是在 Worker 线程中执行的。

#### Container

在经过上面这些过程后，Coyote的全部工作就完成了，它负责监听连接请求（Acceptor），监听IO完成情况（Poller）、将TCP数据封装为HTTP数据并转换为容器可以识别的Request（Worker），最后交给Catalina容器的就是这个已经封装好的Request对象。这个传递过程是通过管道Pipeline来完成的，实现类为`StandardPipeline`，而Container内部每个组件包括Engine、Host、Context、Wrapper之间都是使用Pipeline来连接的：

![Tomcat-Container-Request-Process](images/Tomcat/Tomcat-Container-Request-Process.jpeg)

从时序图可以看出这就是一级一级的向下执行，因为在CoyoteAdapter中说过，Mapper已经提前被封装在Request对象中了，所以这里只要获取就好了，然后调用管道流，调用方法即可：

- request.getHost().getPipeline().getFirst().invoke() 先获取对应的 StandardHost，并执行其 pipeline。
- request.getContext().getPipeline().getFirst().invoke() 先获取对应的 StandardContext,并执行其 pipeline。
- request.getWrapper().getPipeline().getFirst().invoke() 先获取对应的 StandardWrapper，并执行其 pipeline。

这里我们说一下`StandardWrapper`的Basic Valve`StandardWrapperValve`（我们之后再说什么是Valve）：

1. 首先调用allocate方法加载并初始化 Servlet，如果是单例模式还会有特殊处理；
2. 接着createFilterChain() 方法会从 StandardContext 中获取到所有的过滤器，然后将匹配 Request URL 的所有过滤器挑选出来添加到 filterChain 中。
3. doFilter() 执行过滤链,当所有的过滤器都执行完毕后调用 Servlet 的 service() 方法

到这里整个Tomcat的请求处理流程就结束了，这里接着就会调用用户自定义的Servlet或Spring的DispatcherServlet方法了。

接下来来解释一下Valve机制：

#### Pipeline和Valve

我们知道Catalina容器的实现中，最上层的是`ContainerBase`，接着是`StandardEngine`、`StandardHost`、`StandardContext`和`StandardWrapper`（按照顺序逐级排列）。

而这里Tomcat为了实现更多的自定义操作，允许用户在管道上定义自己的处理器，这些处理器就是继承了`ValveBase`的类。而每个管道的末尾就是Tomcat定义的标准Valve，即Container时序图中包含的`StandardEngineValve`、`StandardHostValve`、`StandardContextValve`、`StandardWrapperValve`，这些标准Valve。Tomcat中的Valve实现了具体业务逻辑单元。可以定制化Valve，然后配置在server.xml里。每层容器都可以配置相应的Valve，当只在其作用域内有效。例如engine容器里的valve只对其包含的所有host里的应用有效。

之所以定义了Pipeline和Valve是为了方便我们对每一级容器做一些定制的处理，也为了将中间的调用逻辑和容器本身解耦合，交给Valve来完成具体的业务逻辑。加上Pipeline和Valve后的容器架构图如下：

![Tomcat-Container-With-PipelineAndValve-Arch](images/Tomcat/Tomcat-Container-With-PipelineAndValve-Arch.png)

从Container的时序图也可以看出来，CoyoteAdapter将ServletRequest包装好后通过StandardPipeline交给StandardEngine的管道，具体逻辑由StandardEngineValve进行处理，并由StandardEngineValve交给下一级容器处理，最后，请求在StandardWrapperVavle中构造并执行过滤器链，然后交给具体的Servlet执行业务逻辑。

### Servlet和Spring的请求流程

好了到这里整个Tomcat的请求流程就结束了，Tomcat将请求封装为ServletRequest对象并调用Mapper中匹配的Servlet的service方法进行处理。

如果是我们直接实现Servlet接口，那么直接进入我们定义的service方法进行处理；如果我们继承自HttpServlet方法，那么实际上会由HttpServlet的service方法根据请求的具体方法（GET、POST等）适配到对应的方法执行（如doGet、doPost等）。下面我们以Spring为例，简单分析一下请求的执行过程：

我们知道，DispatcherServlet的初始化在ContextLoaderListener中进行，而在web.xml中，我们配置过DispatcherServlet的servlet-mapping，它规定了哪些请求会转交给DispatcherServlet处理（上面我们知道在CoyoteAdapter中已经将请求路径封装到了Request中了）。由DispatcherServlet管理的一定是Bean形式的控制器（**自定义的Servlet不由DispatcherServlet管理，在CoyoteAdapter中就能直接找到对应的Servlet**）。

具体流程如下：

1. Tomcat中的StandardWrapperValve会负责调用已经解析出来的需要调用的Servlet中的service方法，这里调用的是DispatcherServlet父接口HttpServlet中的service方法；
2. HttpServlet中的service方法会根据请求的方法调用对应的方法，这里以GET方法为例，HttpServlet会调用doGet方法，这里调用的doGet方法实际上是FrameServlet中实现的doGet方法；
3. FrameServlet中的doGet方法通过调用processRequest方法，进而调用抽象的doService方法，这个doService方法由子类DispatcherServlet实现；
4. DispatcherServlet中的doService方法接着调用doDispatcher方法，doDispatcher方法通过获取启动时收集的HandlerMapping信息，并通过适配器HandlerAdapter来逐步调用对应的Controller中的处理方法。

以上就是Spring控制器的请求流程了，结合Tomcat来看就可以获得Spring处理请求的完整流程：

1. Tomcat的Acceptor负责监听连接请求，当存在连接请求时，将其添加到Poller的PollerEvent中；
2. Poller每隔1秒检查PollerEvent队列，当发现有新的连接请求建立时，将这个请求添加到多路复用器中，并阻塞至多1秒等待多路复用器收到就绪的IO；
3. Poller将就绪的IO流（SocketChannel）注册到Worker线程池中，并为每一个Worker绑定一个SocketProcessor处理器，交给Worker线程池进行处理；
4. SocketProcessor负责将收到的TCP请求转换为对应的HTTP请求的数据结构，并通过CoyoteAdapter适配器将Coyote层次中的Request适配为Catalina容器层级的Request，并且寻找并保存当前请求对应每一级容器，接着将Request交给Catalina容器最顶层的Enging处理
5. Catalina容器通过Pipeline和Valve的方式逐级调用，最终在WrapperValve中构造过滤器链并执行过滤器链，接着在调用对应的Servlet的service方法进行具体处理
6. DispatcherServlet的service方法被调用后会由HttpServlet的service方法开始逐级调用，最后调用到DispatcherServlet中的doDispatcher方法，通过DispatcherServlet保存的HandlerMapping交给具体控制器的具体方法执行。

