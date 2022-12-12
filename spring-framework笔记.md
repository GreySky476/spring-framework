## 源码调试
从 github fork 源码到仓库，指定版本为 5.1.x
```groovy
// 执行
./gradlew :spring-oxm:compileTestJava
```
后面可能会出现以下问题

问题
- 出现警告，但指定了 -Werror
  - 全局搜索 -Werror，然后注释掉就能解决
- 报 InstrumentationSavingAgent 不存在的错误
  - 将 optional 改成 compile

`spring-framework`项目下一共四个和 web 相关的项目
![img_2.png](png%2Fimg_2.png)

## 容器初始化

### 容器初始化（一）：Root WebApplicationContext 容器
#### 配置文件（web.xml）
web.xml 的作用
- 在项目中并不是一定需要 web.xml 文件，主要作用用来配置欢迎页、servlet、filter、listener 等以及定制 servlet、JSP、Context 等初始化参数

web.xml 的模式

- 在 <web-app> 中，有很多模式文件，由 sun 公司定义，每个都会指明使用的是哪个模式文件

web.xml 的标签加载顺序
- context-param–> listener –> filter –> servlet

Spring MVC 的 web.xml 配置
- 编写『(servlet-name)』-servlet.xml：这里的 servlet-name 代指 <servlet-name> 的值，必须相同
- 添加 servlet 定义配置 DispatcherServlet：前端控制器，接受 HTTP 请求和转发请求的类，是分发 Controller 请求的，属于 Spring 的核心要素
- 配置 contextConfigLocation 初始化参数：指定 Spring IOC 容器需要读取的定义了非 Web 层的 Bean（DAO/Service）的 XML 文件路径。可以指定多个，
可以用逗号、冒号分隔。如果没有指定，则会在 /WEB-INF/ 下查找 "servlet-name.xml" 文件加载
##### servletName-servlet.xml

```xml
<!-- 扫描包 Spring 可以自动去扫描 base-package 下面或者子包下面的 java 文件，如果扫描到有 @Component @Controller @Service 等注解
 的类，则把这些类注册为 Bean -->
<context:component-scan xmlns="http://www.springframework.org/schema/context" base-package="com.sql.*" />

<!-- 不会拦截静态资源的处理器 -->
<mvc:default-servlet-handler />

<!-- 注解驱动 -->
<mvc:annotation-driven />

```
##### web.xml
```xml
<!-- 省略非关键的配置 -->

<!-- [1] Spring配置 -->
<listener>
    <listener-class>org.springframework.web.context.ContextLoaderListener</listener-class>
</listener>


        <!-- ====================================== -->

        <!-- [2] Spring MVC配置 DispatcherServlet-->
<servlet>
<servlet-name>spring</servlet-name> <!-- 【3】 -->
<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class> <!-- 【4】 -->
<!-- 可以自定义servlet.xml配置文件的位置和名称，默认为WEB-INF目录下，名称为[<servlet-name>]-servlet.xml，如spring-servlet.xml -->
<init-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/spring-servlet.xml</param-value> // 默认
</init-param>

<load-on-startup>1</load-on-startup>
</servlet>
<!-- [3] 配置请求地址拦截 url -->
<servlet-mapping>
<servlet-name>spring</servlet-name> <!-- 【2】 -->
<url-pattern>*.do</url-pattern> <!-- 【1】 -->
</servlet-mapping>

<!-- [4] 指定Spring Bean的配置文件所在目录。默认配置在WEB-INF目录下 -->
<context-param>
<param-name>contextConfigLocation</param-name>
<param-value>classpath:config/applicationContext.xml</param-value>
</context-param>
```
- [1] 处，配置了`org.springframework.web.context.ContextLoaderListener`对象。是一个`javax.servlet.ServletContextListener`
对象，在 Servlet 容器启动时，例如 Tomcat、Jetty 启动，则会被 ContextLoaderListener 监听到，从而调用 contextInitialized，会初始化一个 Root Spring WebApplicationContext 容器。
- [2] 处，配置了 `DispatcherServlet`对象。是一个`javax.servlet.http.HttpServlet`对象，除了拦截我们
制定的`*.do`请求外，也会初始化一个属于它的 Spring WebApplicationContext 容器。并且，这个容器是以 [1] 处的 Root 容器作为父容器
- [3] 处，是 <servlet-mapping> 标签，刚好和 <servlet> 标签对应，<servlet-mapping> 中的 <servlet-name> 需要和 <servlet> 中的保持一致，
这里涉及到执行顺序，为 【1】->【2】->【3】->【4】
- [4] 处，如果不指定 contextConfigLocation，会默认寻找 classpath，也就是 WEB-INF(classpath)/servletName-servlet.xml
> - 详细的 spring mvc 配置文件解析：https://blog.csdn.net/mynewclass/article/details/78501604
> - 有了 [1] 的创建容器，为什么还需要 [2] 的创建？
  > - 可以配置多个 [2]
  > - [1] 和 [2] 分别会创建其对应的 Spring WebApplicationContext 容器，并且是父子容器关系。
#### 如何调试
- 执行 `ContextLoaderTests#testContextLoaderListenerWithDefaultContext()`单元测试方法
#### Root WebApplicationContext 容器
![img.png](png%2Fimg.png)
- 类图如下
![ContextLoaderListener.png](png%2FContextLoaderListener.png)
- XmlWebApplicationContext 类图
![img_1.png](png%2Fimg_1.png)
##### ContextLoaderListener
- 该类在 spring-web 项目中 `spring-web/src/main/java/org/springframework/web/context/ContextLoaderListener.java`
###### 构造方法
```java
// ContextLoaderListener.java

public ContextLoaderListener() {
}

public ContextLoaderListener(WebApplicationContext context) {
	super(context);
}
```
这两个构造方法，是因为父类 ContextLoader 有这两个构造方法，所以必须重新定义。
###### contextInitialized
```java
// ContextLoaderListener.java

@Override
public void contextInitialized(ServletContextEvent event) {
    // 初始化 WebApplicationContext
	initWebApplicationContext(event.getServletContext());
}
```
- 调用父类 ContextLoader 的 `#initWebApplicationContext(ServletContext servletContext)` 方法
###### contextDestroyed
```java
// ContextLoaderListener.java

@Override
public void contextDestroyed(ServletContextEvent event) {
	closeWebApplicationContext(event.getServletContext());
	ContextCleanupListener.cleanupAttributes(event.getServletContext());
}
```
- 销毁 WebApplicationContext 的容器
##### ContextLoader
- 该类在 spring-web 包下 `spring-web/src/main/java/org/springframework/web/context/ContextLoader.java`
###### 构造方法
第一块，`defaultStrategies` 静态属性，默认的配置 Properties 对象
```java
// ContextLoader.java

private static final Properties defaultStrategies;

	static {
		// 从属性文件加载默认策略实现。这目前是严格内部的，并不意味着由应用程序开发人员自定义。
		// 当没有在 <context-param/> 中指定，配置文件默认指定 WebApplicationContext 类为 XmlWebApplicationContext 类
		// Load default strategy implementations from properties file.
		// This is currently strictly internal and not meant to be customized
		// by application developers.
		try {
			ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
			defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
		}
		catch (IOException ex) {
			throw new IllegalStateException("Could not load 'ContextLoader.properties': " + ex.getMessage());
		}
	}
```
---
第二块，`context` 属性，Root WebApplicationContext 对象
```java
// ContextLoader.java

/**
 * Root WebApplicationContext 对象
 *
 * The root WebApplicationContext instance that this loader manages.
 */
@Nullable
private WebApplicationContext context;

public ContextLoader() {
}

public ContextLoader(WebApplicationContext context) {
	this.context = context;
}
```
###### initWebApplicationContext
- 初始化 WebApplicationContext 对象，代码如下
```java
public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
		// <1> 开始初始化时，如果 ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE 对应的 WebApplicationContext 不为空则创建失败
		// 比如在 web.xml 中存在多个 ContextLoader
		if (servletContext.getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
			throw new IllegalStateException(
					"Cannot initialize context because there is already a root application context present - " +
					"check whether you have multiple ContextLoader* definitions in your web.xml!");
		}

		// <2> 日志打印
		servletContext.log("Initializing Spring root WebApplicationContext");
		Log logger = LogFactory.getLog(ContextLoader.class);
		if (logger.isInfoEnabled()) {
			logger.info("Root WebApplicationContext: initialization started");
		}
		// 记录时间
		long startTime = System.currentTimeMillis();

		try {
			// 将上下文存储在本地实例变量中，以保证它在 ServletContext 关闭时可用。
			// Store context in local instance variable, to guarantee that
			// it is available on ServletContext shutdown.
			if (this.context == null) {
				// <3> 初始化 context，即创建 context 对象
				this.context = createWebApplicationContext(servletContext);
			}
			// <4> 如果是 ConfigurableWebApplicationContext 的子类，如果未刷新，则进行配置和刷新
			if (this.context instanceof ConfigurableWebApplicationContext) {
				ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) this.context;
				// 不处于活动状态
				if (!cwac.isActive()) {
					// <4.1> 上下文尚未刷新 ->提供设置父上下文、设置应用程序上下文 ID 等服务
					// The context has not yet been refreshed -> provide services such as
					// setting the parent context, setting the application context id, etc
					if (cwac.getParent() == null) {
						// <4.2> 上下文实例是在没有显式父 -> 确定根 Web 应用程序上下文的父级（如果有）的情况下注入的。
						// The context instance was injected without an explicit parent ->
						// determine parent for root web application context, if any.
						ApplicationContext parent = loadParentContext(servletContext);
						cwac.setParent(parent);
					}
					// <4.3> 配置和刷新 Web 应用程序上下文
					configureAndRefreshWebApplicationContext(cwac, servletContext);
				}
			}
			// <5> 记录在 servletContext 中
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, this.context);

			// <6> 记录到 currentContext 或 currentContextPerThread 中
			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = this.context;
			}
			else if (ccl != null) {
				currentContextPerThread.put(ccl, this.context);
			}

			// <7> 打印日志
			if (logger.isInfoEnabled()) {
				long elapsedTime = System.currentTimeMillis() - startTime;
				logger.info("Root WebApplicationContext initialized in " + elapsedTime + " ms");
			}

			// <8> 返回 context
			return this.context;
		}
		catch (RuntimeException | Error ex) {
			// <9> 当出现异常时，记录到 WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE 中，不再重新初始化
			logger.error("Context initialization failed", ex);
			servletContext.setAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, ex);
			throw ex;
		}
	}
```
- <1>
- <2>
- <3> 处，调用 `#createWebApplicationContext(ServletContext sc)` 方法，初始化 `context`，创建 WebApplicationContext 对象
- <4> 处，如果 `context` 是 ConfigurableWebApplicationContext 的子类，如果未刷新，则进行配置和刷新
  - <4.1> 处，如果未刷新。默认情况下都是未刷新
  - <4.2> 处，无父容器，则进行加载和配置。默认情况下，`#loadParentContext(ServletContext servletContext)`方法，返回 `null`，代码如下
```java
// ContextLoader.java

@Nullable
protected ApplicationContext loadParentContext(ServletContext servletContext) {
	return null;
}
```
-
    - 这是一个子类实现的方法。子类 ContextLoader 并没有重写，所以默认返回 null。
  - <4.3> 处，调用`#configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc)`方法，
配置 ConfigurableWebApplicationContext 对象，并进行刷新
- <5> 处，记录 `context` 在 ServletContext 中，如果 `web.xml` 定义了多个 ContextLoader，就会在 <1> 处报错
- <6> 处，记录到 `currentContext`或`currentContextPerThread`中，差异在于类加载器的不同
```java
// ContextLoader.java

/**
 * Map from (thread context) ClassLoader to corresponding 'current' WebApplicationContext.
 */
private static final Map<ClassLoader, WebApplicationContext> currentContextPerThread =
		new ConcurrentHashMap<>(1);
    
/**
 * The 'current' WebApplicationContext, if the ContextLoader class is
 * deployed in the web app ClassLoader itself.
 */
@Nullable
private static volatile WebApplicationContext currentContext;
```
- 
  - 在销毁 Spring WebApplicationContext 容器时会用到
- <7> 处，打印日志
- <8> 处，返回 context
- <9> 处，发生异常，记录异常到`WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE`

###### createWebApplicationContext 方法
```java
protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
		// <1> 获得 context 的类
		Class<?> contextClass = determineContextClass(sc);
		// <2> 判断 context 的类，是否符合 ConfigurableWebApplicationContext 的类型
		if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
			throw new ApplicationContextException("Custom context class [" + contextClass.getName() +
					"] is not of type [" + ConfigurableWebApplicationContext.class.getName() + "]");
		}
		// <3> 创建 context 的类的对象
		return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
	}
```
- <1> 处，调用 determineContextClass(ServletContext servletContext) 方法，获得 context 的类
```java
protected Class<?> determineContextClass(ServletContext servletContext) {
		// 获得参数 contextClass 的值
		String contextClassName = servletContext.getInitParameter(CONTEXT_CLASS_PARAM);
		// 情况一：如果值非空，则获得该类
		if (contextClassName != null) {
			try {
				return ClassUtils.forName(contextClassName, ClassUtils.getDefaultClassLoader());
			}
			catch (ClassNotFoundException ex) {
				throw new ApplicationContextException(
						"Failed to load custom context class [" + contextClassName + "]", ex);
			}
		}
		// 情况二：从 defaultStrategies(默认策略) 获得
		else {
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
  - 分为两种情况，一个从 Servlet 配置的 context 类，一个从 ContextLoader.properties 配置的 context 类
  - 默认情况下，不会主动在 servletContext 配置 context 类，基本使用配置文件中的 XmlWebApplicationContext 类
- <2> 处，判断 context 是否符合 ConfigurableWebApplicationContext，根据以上类图可知 Web... 类继承自此类
- <3> 处，调用 BeanUtils#instantiateClass(Class<T> clazz)

###### configureAndRefreshWebApplicationContext 方法
configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) 方法，配置 ConfigurableWebApplicationContext 对象，
并进行刷新。
```java
protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
		// <1> 如果 wac 使用了默认编号，则重新设置 id 属性
		if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
			// 应用程序上下文 ID 仍设置为其原始默认值 ->根据可用信息分配更有用的 ID
			// The application context id is still set to its original default value
			// -> assign a more useful id based on available information
			// 情况一，使用 contextId 属性
			String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
			if (idParam != null) {
				wac.setId(idParam);
			}
			// 情况二，自动生成
			else {
				// Generate default id...
				wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
						ObjectUtils.getDisplayString(sc.getContextPath()));
			}
		}

		// <2> 设置 context 的 servletContext 属性
		wac.setServletContext(sc);
		// <3> 设置 context 的配置文件地址 contextConfigLocation
		String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
		if (configLocationParam != null) {
			wac.setConfigLocation(configLocationParam);
		}

		// <4> 在任何情况下，当上下文刷新时，都会调用 wac 环境的 initPropertySources;
		// 在这里急切地执行此操作，以确保 servlet 属性源已到位，以便在刷新之前在下面发生的任何后处理或初始化中使用
		// The wac environment's #initPropertySources will be called in any case when the context
		// is refreshed; do it eagerly here to ensure servlet property sources are in place for
		// use in any post-processing or initialization that occurs below prior to #refresh
		ConfigurableEnvironment env = wac.getEnvironment();
		if (env instanceof ConfigurableWebEnvironment) {
			((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
		}

		// <5> 执行自定义初始化 context
		customizeContext(sc, wac);
		
		// <6> 刷新 context，执行初始化
		wac.refresh();
	}
```
- <1> 处，如果 wac 使用了默认编号则需要重新设置 id 属性。默认情况下，不会对 wac 设置编号，所以会进去
，id 的生成规则分为：使用 contextId，在 <context-param /> 标签中设置、自动生成两种情况，默认情况走第二种
- <2> 处，设置 wac 的 ServletContext 属性
- <3> 处，设置 context 的配置文件地址，这里对应 web.xml 中
```xml
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>classpath:config/applicationContext.xml</param-value>
</context-param>
```
- <4> 待补充
- <5> 处，调用 #customizeContext(ServletContext sc, ConfigurableWebApplicationContext wac)，
执行自定义初始化 wac。非关键方法
- <6> 处，刷新 wac，执行初始化。此处会进行一些 spring 容器的初始化

###### closeWebApplicationContext
`#closeWebApplicationContext(ServletContext servletContext)`方法，关闭 WebApplicationContext 容器对象
```java
public void closeWebApplicationContext(ServletContext servletContext) {
		servletContext.log("Closing Spring root WebApplicationContext");
		try {
			// <1> 当前 context 属于 ConfigurableWebApplicationContext 的子类，一般是 XmlWebApplicationContext，则关闭
			if (this.context instanceof ConfigurableWebApplicationContext) {
				((ConfigurableWebApplicationContext) this.context).close();
			}
		}
		finally {
			// <2> 获取当前的类加载器，将其引用 currentContext 置空或从 currentContextPerThread 删除 
			ClassLoader ccl = Thread.currentThread().getContextClassLoader();
			if (ccl == ContextLoader.class.getClassLoader()) {
				currentContext = null;
			}
			else if (ccl != null) {
				currentContextPerThread.remove(ccl);
			}
			// <3> 删除 servletContext 中的 ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE，这里对应 initWebApplicationContext 方法中的判断
			servletContext.removeAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE);
		}
	}
```
- <1> 处，当前 context 属于 ConfigurableWebApplicationContext 的子类则执行 `ConfigurableWebApplicationContext#close`方法
  - `#colse()` 方法唯一实现类 AbstractApplicationContext
```java
// AbstractApplicationContext.java

public void close() {
		synchronized (this.startupShutdownMonitor) {
			// <1> 调用 doClose() 方法
			doClose();
			// <2> 如果我们注册了一个 JVM 关闭钩子，我们现在不再需要它：我们已经显式关闭了上下文。
			// If we registered a JVM shutdown hook, we don't need it anymore now:
			// We've already explicitly closed the context.
			if (this.shutdownHook != null) {
				try {
					Runtime.getRuntime().removeShutdownHook(this.shutdownHook);
				}
				catch (IllegalStateException ex) {
					// ignore - VM is already shutting down
				}
			}
		}
	}
```
-   - <1> 处，调用了 `#doClose()` 方法
      - `#doColse()` 方法
```java
protected void doClose() {
		// <1> 通过 CAS 将 closed 设置为 true
		// 检查是否需要实际的关闭尝试...
		// Check whether an actual close attempt is necessary...
		if (this.active.get() && this.closed.compareAndSet(false, true)) {
			if (logger.isDebugEnabled()) {
				logger.debug("Closing " + this);
			}
			// <2> 注销应用程序上下文
			LiveBeansView.unregisterApplicationContext(this);

			try {
				// <3> 发布事件，这里发布 ContextClosedEvent 
				// Publish shutdown event.
				publishEvent(new ContextClosedEvent(this));
			}
			catch (Throwable ex) {
				logger.warn("Exception thrown from ApplicationListener handling ContextClosedEvent", ex);
			}
			// <4> 停止所有生命周期 bean，以避免在单个销毁期间出现延迟。
			// Stop all Lifecycle beans, to avoid delays during individual destruction.
			if (this.lifecycleProcessor != null) {
				try {
					this.lifecycleProcessor.onClose();
				}
				catch (Throwable ex) {
					logger.warn("Exception thrown from LifecycleProcessor on context close", ex);
				}
			}
			// <5> 销毁上下文的 BeanFactory 中所有缓存的单例。
			// Destroy all cached singletons in the context's BeanFactory.
			destroyBeans();

			// <6> 关闭此上下文本身的状态 
			// Close the state of this context itself.
			closeBeanFactory();

			// <7> 如果子类需要，让他们做一些最后的清理......
			// Let subclasses do some final clean-up if they wish...
			onClose();

			// <8> 将本地应用程序侦听器重置为预刷新状态。
			// Reset local application listeners to pre-refresh state.
			if (this.earlyApplicationListeners != null) {
				this.applicationListeners.clear();
				this.applicationListeners.addAll(this.earlyApplicationListeners);
			}
			
			// <9> 切换到非活动状态
			// Switch to inactive.
			this.active.set(false);
		}
	}
```
-    - <2> 处，如果注册了 JVM 关闭钩子，移除掉 
### 容器初始化（二）：Servlet WebApplicationContext 容器
#### 概述
- Root WebApplicationContext 是由 ContextLoaderListener 监听到 ServletContextEvent 事件进行后续的初始化
- Servlet WebApplicationContext 容器的初始化是在 DispatcherServlet 初始化的过程中执行
- DispatcherServlet 类图
- ![img_3.png](png%2Fimg_3.png)
- HttpServletBean，负责将 ServletConfig 设置到当前 Servlet 对象中。
```java
// HttpServletBean.java

/**
 * 其HttpServlet简单扩展将其配置参数（中web.xml标记内的servlet条目）init-param视为 Bean 属性。
 * 
 * Simple extension of {@link javax.servlet.http.HttpServlet} which treats
 * its config parameters ({@code init-param} entries within the
 * {@code servlet} tag in {@code web.xml}) as bean properties.
 */
```
- FrameworkServlet，负责初始化 Spring Servlet WebApplicationContext 容器
```java
// FrameworkServlet.java

/**
 * Spring Web 框架的基本 servlet。在基于 JavaBean 的整体解决方案中提供与 Spring 应用程序上下文的集成。
 * 
 * Base servlet for Spring's web framework. Provides integration with
 * a Spring application context, in a JavaBean-based overall solution.
 * 
 */
```
- DispatcherServlet，负责初始化 Spring MVC 的各个组件，以及处理客户端的请求
```java
// DispatcherServlet.java

/**
 * HTTP 请求处理程序控制器的中央调度程序，例如用于 Web UI 控制器或基于 HTTP 的远程服务导出器。
 * 向注册处理程序发送用于处理 Web 请求，提供方便的映射和异常处理工具。
 * 
 * Central dispatcher for HTTP request handlers/controllers, e.g. for web UI controllers
 * or HTTP-based remote service exporters. Dispatches to registered handlers for processing
 * a web request, providing convenient mapping and exception handling facilities.
 */
```
#### 调试
执行 `DispatcherServletTests#configuredDispatcherServlets()`测试方法
```java

  private final MockServletConfig servletConfig = new MockServletConfig(new MockServletContext(), "simple");
  
  private DispatcherServlet simpleDispatcherServlet;
  
  private DispatcherServlet complexDispatcherServlet;

    @Before
	public void setUp() throws ServletException {
		// MockServletConfig 继承 ServletConfig，根据步骤，先初始化 ServletConfig
		MockServletConfig complexConfig = new MockServletConfig(getServletContext(), "complex");
		complexConfig.addInitParameter("publishContext", "false");
		complexConfig.addInitParameter("class", "notWritable");
		complexConfig.addInitParameter("unknownParam", "someValue");
		// 初始化简单配置的 DispatcherServlet
		simpleDispatcherServlet = new DispatcherServlet();
		//
		simpleDispatcherServlet.setContextClass(SimpleWebApplicationContext.class);
		// HttpServletBean：负责将 ServletConfig 设置到当前 Servlet 对象
		/*
		 1、调用父类 GenericServlet#init 方法，然后初始化将由 HttpServletBean 实现的 init 方法开始初始化
		 2、HttpServletBean init 方法执行完之后调用 initServletBean 方法，将初始化 Spring Servlet WebApplicationContext 容器的任务
		 	交由实现了 initServletBean 方法的 FrameworkServlet 处理 
		 */
		simpleDispatcherServlet.init(servletConfig);

		// 初始化复杂的 DispatcherServlet
		complexDispatcherServlet = new DispatcherServlet();
		complexDispatcherServlet.setContextClass(ComplexWebApplicationContext.class);
		complexDispatcherServlet.setNamespace("test");
		complexDispatcherServlet.addRequiredProperty("publishContext");
		complexDispatcherServlet.init(complexConfig);
	}

  private ServletContext getServletContext() {
          return servletConfig.getServletContext();
    }

  @Test
  public void configuredDispatcherServlets() {
          assertTrue("Correct namespace",
          ("simple" + FrameworkServlet.DEFAULT_NAMESPACE_SUFFIX).equals(simpleDispatcherServlet.getNamespace()));
          assertTrue("Correct attribute", (FrameworkServlet.SERVLET_CONTEXT_PREFIX + "simple").equals(
          simpleDispatcherServlet.getServletContextAttributeName()));
          assertTrue("Context published", simpleDispatcherServlet.getWebApplicationContext() ==
          getServletContext().getAttribute(FrameworkServlet.SERVLET_CONTEXT_PREFIX + "simple"));

          assertTrue("Correct namespace", "test".equals(complexDispatcherServlet.getNamespace()));
          assertTrue("Correct attribute", (FrameworkServlet.SERVLET_CONTEXT_PREFIX + "complex").equals(
          complexDispatcherServlet.getServletContextAttributeName()));
          assertTrue("Context not published",
          getServletContext().getAttribute(FrameworkServlet.SERVLET_CONTEXT_PREFIX + "complex") == null);

          simpleDispatcherServlet.destroy();
          complexDispatcherServlet.destroy();
    }
```
#### HttpServletBean
`spring-webmvc/src/main/java/org/springframework/web/servlet/HttpServletBean.java`，实现了 EnvironmentCapable、EnvironmentAware
接口，继承 HttpServlet 抽象类，负责将 ServletConfig 集成到 Spring 中。HttpServletBean 本身也是抽象类
##### 构造方法
```java
// HttpServletBean.java

  @Nullable
  private ConfigurableEnvironment environment;
  
  /**
   * 必须配置的属性的集合
   */
  private final Set<String> requiredProperties = new HashSet<>(4);
```
- `environment`属性，相关的方法代码
```java
// HttpServletBean.java

  /**
   * 实现自 EnvironmentAware 接口，自动注入
   */
    @Override
	public void setEnvironment(Environment environment) {
		Assert.isInstanceOf(ConfigurableEnvironment.class, environment, "ConfigurableEnvironment required");
		this.environment = (ConfigurableEnvironment) environment;
	}

    /**
     * 实现自 EnvironmentCapable 接口
     */
    @Override
    public ConfigurableEnvironment getEnvironment() {
        if (this.environment == null) {
        this.environment = createEnvironment();
        }
        return this.environment;
        }
        
    protected ConfigurableEnvironment createEnvironment() {
        return new StandardServletEnvironment();
        }
```
- 为什么`environment`属性能自动注入？
  - EnvironmentAware 接口，具体需要查看 Spring IOC
- `requiredProperties`属性，必须配置的属性的集合。可通过`#addRequiredProperty(String property)`方法添加
```java
    protected final void addRequiredProperty(String property) {
		this.requiredProperties.add(property);
	}
```
##### init
`#init()`方法，负责将 ServletConfig 设置到当前 Servlet 对象中
```java
// HttpServletBean.java

    @Override
    public final void init() throws ServletException {
        // 从初始化参数设置 Bean 属性。
        // Set bean properties from init parameters.
        // <1> 解析 <init-param /> 标签，封装到 PropertyValues pvs 中
        PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(), this.requiredProperties);
        if (!pvs.isEmpty()) {
        try {
        // <2.1> 将当前这个 Servlet 对象转化成一个 BeanWrapper 对象。从而以 Spring 的方式将 pvs 注入到 BeanWrapper 对象中
        BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
        ResourceLoader resourceLoader = new ServletContextResourceLoader(getServletContext());
        // <2.2> 注册自定义属性编辑器，一旦碰到 Resource 类型的属性，将会使用 ResourceEditor 解析
        bw.registerCustomEditor(Resource.class, new ResourceEditor(resourceLoader, getEnvironment()));
        // <2.3> 空实现，留给子类覆盖
        initBeanWrapper(bw);
        // <2.4> 以 Spring 的方式来将 pvs 注入到该 Wrapper 对象中
        bw.setPropertyValues(pvs, true);
        }
        catch (BeansException ex) {
        if (logger.isErrorEnabled()) {
        logger.error("Failed to set bean properties on servlet '" + getServletName() + "'", ex);
        }
        throw ex;
        }
        }

        // 让子类执行它们喜欢的任何初始化。
        // Let subclasses do whatever initialization they like.
        // <3> 子类来实现，实现自定义的初始化逻辑。目前有具体的实现
        initServletBean();
        }
```
- <1> 处，解析 Servlet 配置的 <init-param /> 标签，封装到 PropertyValues 中。
`ServletConfigPropertyValues`是 `HttpServletBean`的私有静态类，继承自`MutablePropertyValues`
类，ServletConfig 的 PropertyValues 实现类
```java
// HttpServletBean.java

private static class ServletConfigPropertyValues extends MutablePropertyValues {

  /**
   * 创建新的 ServletConfigPropertyValues
   *
   * Create new ServletConfigPropertyValues.
   * @param config the ServletConfig we'll use to take PropertyValues from
   * @param requiredProperties set of property names we need, where
   * we can't accept default values
   * @throws ServletException if any required properties are missing
   */
  public ServletConfigPropertyValues(ServletConfig config, Set<String> requiredProperties)
          throws ServletException {
    // 获得缺失的属性的集合
    Set<String> missingProps = (!CollectionUtils.isEmpty(requiredProperties) ?
            new HashSet<>(requiredProperties) : null);

    // <1> 遍历 ServletConfig 的初始化参数集合，添加到 ServletConfigPropertyValues 中，并从 missingProps 中移除
    Enumeration<String> paramNames = config.getInitParameterNames();
    while (paramNames.hasMoreElements()) {
      String property = paramNames.nextElement();
      Object value = config.getInitParameter(property);
      // 添加到 ServletConfigPropertyValues 中
      addPropertyValue(new PropertyValue(property, value));
      // 从 missingProps 中移除
      if (missingProps != null) {
        missingProps.remove(property);
      }
    }

    // Fail if we are still missing properties.
    // <2> 如果存在缺失的属性，抛出 ServletException 异常
    if (!CollectionUtils.isEmpty(missingProps)) {
      throw new ServletException(
              "Initialization from ServletConfig for servlet '" + config.getServletName() +
                      "' failed; the following required properties were missing: " +
                      StringUtils.collectionToDelimitedString(missingProps, ", "));
    }
  }
}
```
-   - 代码简单，实现两方面逻辑：<1> 处，遍历 ServletConfig 的初始化参数集合，添加到 ServletConfigPropertyValues 中
；<2> 处，判断要求的属性是否齐全，不齐全则排除异常
- <2.1> 处，将当前的 Servlet 转换成 BeanWrapper 对象。从而以 Spring 方式将 `pvs`
注入到该 BeanWrapper 对象中。BeanWrapper 是 Spring 提供的一个用来操作 JavaBean 属性的工具，使用它可以直接修改一个对象的属性。
- <2.2> 处，注册自定以属性编辑器，一旦碰到 Resource 类型属性，将会使用 ResourceEditor 进行解析
- <2.3> 处，空实现，留给子类覆盖，如下
```java
// HttpServletBean.java

/**
	 * 初始化此 HttpServletBean 的 BeanWrapperper，可能使用自定义编辑器。
	 * 此默认实现为空。
	 *
	 * Initialize the BeanWrapper for this HttpServletBean,
	 * possibly with custom editors.
	 * <p>This default implementation is empty.
	 * @param bw the BeanWrapper to initialize
	 * @throws BeansException if thrown by BeanWrapper methods
	 * @see org.springframework.beans.BeanWrapper#registerCustomEditor
	 */
	protected void initBeanWrapper(BeanWrapper bw) throws BeansException {
	}
```
-   - 实际上，子类暂时没有实现
- <2.4> 处，以 Spring 方式来将 `pvs` 注入到 BeanWrapper 对象中，即设置到当前 Servlet 对象中。
例子：
```xml
// web.xml

<servlet>
    <servlet-name>spring</servlet-name>
    <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
    <init-param>
        <param-name>contextConfigLocation</param-name>
        <param-value>/WEB-INF/spring-servlet.xml</param-value>
    </init-param>
    <load-on-startup>1</load-on-startup>
</servlet>
<servlet-mapping>
    <servlet-name>spring</servlet-name>
    <url-pattern>*.do</url-pattern>
</servlet-mapping>
```
-   - 此处配置了`contextConfigLocation`属性，通过 <2.4> 的逻辑，会反射设置到 `FrameworkServlet.contextConfigLocation`属性，代码如下
```java
// FrameworkServlet.java
    // 显式上下文配置位置。
    /** Explicit context config location. */
    @Nullable
    private String contextConfigLocation;
    
    public void setContextConfigLocation(@Nullable String contextConfigLocation) {
        this.contextConfigLocation = contextConfigLocation;
    }
```
-   - 玩的真花，艹
- <3> 处，调用 `#initServletBean()`，目前 FrameworkServlet 实现该方法
```java
// HttpServletBean.java

protected void initServletBean() throws ServletException {
}
```
#### FrameworkServlet




