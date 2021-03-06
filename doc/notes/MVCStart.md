## MVC 启动过程

首先，`ServletContext` 在 `web` 容器中被初始化时将造成 `ServletContextListener` 内定义的 `contextInitialized` 方法被执行，但是在这之前会触发类的初始化，先执行静态变量、静态块，在 Spring MVC 中就利用了这一点进行根上下文以及其它一些工作的初始化，`MVC` 中配置的是继承自 `ServletContextListener` 的 `ContextLoaderListener`，在其父类 `ContextLoader` 中存在以下静态块：

```java
private static final Properties defaultStrategies;

static {
    // 加载默认策略。
    try {
        // 读取策略配置文件 - org/springframework/web/context/ContextLoader.properties。
        ClassPathResource resource 
            = new ClassPathResource(DEFAULT_STRATEGIES_PATH, ContextLoader.class);
        // 返回一个 Properties 对象。
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    }
    catch (IOException ex) {
        throw new IllegalStateException("Could not load 'ContextLoader.properties': " 
                                        + ex.getMessage());
    }
}
```

`org/springframework/web/context/ContextLoader.properties` 内容：

```properties
org.springframework.web.context.WebApplicationContext
=org.springframework.web.context.support.XmlWebApplicationContext
```

这里配置了应用将会初始化哪种 `ApplicationContext` 的实现类。进入下一步：`contextInitialized` 方法的调用：

```java
public void contextInitialized(ServletContextEvent event) {
    // 这里调用其父类 ContextLoader 的实现。
    initWebApplicationContext(event.getServletContext());
}
```

```java
// class ContextLoader

// 根上下文，为什么此类实例要持有此根上下文呢？
// 原因是，为了在 ServletContext 销毁时执行根上下文的 close。
private WebApplicationContext context;

public WebApplicationContext initWebApplicationContext(ServletContext servletContext) {
    // 判断根上下文是否已经存在，存在则抛出异常。
    // 判断的原因如异常 message 描述。
    if (servletContext
        .getAttribute(WebApplicationContext.ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE) != null) {
        throw new IllegalStateException(
            "Cannot initialize context because there is already" 
            +" a root application context present - " 
            + "check whether you have multiple ContextLoader* definitions in your web.xml!");
    }
	// 开始初始化根上下文。
    //... 这里省略一些逻辑无关的代码。
    try {
        // 判断当前实例持有的 上下文是否存在，不存在则创建。
        if (this.context == null) {
            // 创建根上下文，此方法的作用在后面有详细说明，可先去看看。
            this.context = createWebApplicationContext(servletContext);
        }
        // 判断实例化的上下文是否是 ConfigurableWebApplicationContext 类型的
        if (this.context instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac 
                = (ConfigurableWebApplicationContext) this.context;
            // 判断上下文是否被初始化，没有则进行初始化操作。
            if (!cwac.isActive()) {
                // 
                if (cwac.getParent() == null) {
                    // 这里直接返回的 null
                    ApplicationContext parent = loadParentContext(servletContext);
                    cwac.setParent(parent);
                }
                // 初始化根上下文，包括资源读取，资源解析，内部 beanFactory 的初始化，
                // beanFactory 刷新（核心）等。
                // 在下面进行说明。
                configureAndRefreshWebApplicationContext(cwac, servletContext);
            }
        }
        // 将根上下文添加到 servletContext 中去。
        servletContext.setAttribute(WebApplicationContext
                          .ROOT_WEB_APPLICATION_CONTEXT_ATTRIBUTE, 
                          this.context);

      
        ClassLoader ccl = Thread.currentThread().getContextClassLoader();
        if (ccl == ContextLoader.class.getClassLoader()) {
            currentContext = this.context;
        }
        else if (ccl != null) {
            currentContextPerThread.put(ccl, this.context);
        }

        // ... 省略无关逻辑的代码

        return this.context;
    }
    // ... 省略一些列 catch
}

protected WebApplicationContext createWebApplicationContext(ServletContext sc) {
    // 首先获取根上下文类型：
    // 1. 若在 web.xml 中配置了 contextClass，则使用此配置类型。
    // 2. 若不存在则使用前面读取的默认策略中配置的类型。
    Class<?> contextClass = determineContextClass(sc);
    // 判断 contextClass 是否为 ConfigurableWebApplicationContext 的实现类或者其本身类型。
    // 若不是则抛出异常。
    if (!ConfigurableWebApplicationContext.class.isAssignableFrom(contextClass)) {
        throw new ApplicationContextException("Custom context class [" 
                  + contextClass.getName() 
                  + "] is not of type [" 
                  + ConfigurableWebApplicationContext.class.getName() + "]");
    }
    // 通过工具类初始化根上下文的实例。
    // 这里会判断 contextClass 是否属与 kotlin 类型，是则使用 kotlin 的方式获取构造方法。
    // 接下来通过构造方法实例化一个实例。
    // 在这个实例化过程中，其所有的上级类都没有什么特别的操作，所以就不写实例化干了些什么了。
    return (ConfigurableWebApplicationContext) BeanUtils.instantiateClass(contextClass);
}

protected void configureAndRefreshWebApplicationContext(ConfigurableWebApplicationContext wac, ServletContext sc) {
    // 判断上下文的 id 是否等于（className + 对象hash的十六进制）。
    // 是则表示为默认 Id，将设置为更有用的值。
    if (ObjectUtils.identityToString(wac).equals(wac.getId())) {
        // 读取 web.xml 中配置的 contextId。
        String idParam = sc.getInitParameter(CONTEXT_ID_PARAM);
        if (idParam != null) {
            wac.setId(idParam);
        }
        else {
            // 生成默认 Id。
            wac.setId(ConfigurableWebApplicationContext.APPLICATION_CONTEXT_ID_PREFIX +
                      ObjectUtils.getDisplayString(sc.getContextPath()));
        }
    }
	// 将 ServletContext 设置到上下文中去。
    wac.setServletContext(sc);
    // 读取 web.xml 中配置的 contextConfigLocation。
    String configLocationParam = sc.getInitParameter(CONFIG_LOCATION_PARAM);
    if (configLocationParam != null) {
        // 将 configLocation 设置到上下文中去。
        wac.setConfigLocation(configLocationParam);
    }
	// 这里获取上下文环境，会进行一系列环境属性资源的载入。
    // 在后面进行说明。
    // The wac environment's #initPropertySources will be called in any case when the context
    // is refreshed; do it eagerly here to ensure servlet property sources are in place for
    // use in any post-processing or initialization that occurs below prior to #refresh
    ConfigurableEnvironment env = wac.getEnvironment();
    // 判断上下文环境对象是否是 ConfigurableWebEnvironment 类型的，若是则执行
    if (env instanceof ConfigurableWebEnvironment) {
        // 因为前面进行环境内属性资源的载入时，servletConfigInitParams，servletContextInitParams
        // 都没有实际的值，在这里载入值。
        // 这里调用 StandardServletEnvironment 内的方法、。
        // 在后面进行说明。
        ((ConfigurableWebEnvironment) env).initPropertySources(sc, null);
    }
    // 在下面进行说明。
    customizeContext(sc, wac);
    // 执行根上下文的初始化操作，这是所有上下文初始化的核心方法。
    // 该方法实现在 AbstractApplicationContext 中。
    // 在后面进行说明。
    wac.refresh();
}

// 这里是用户定制化上下文的解析过程。
protected void customizeContext(ServletContext sc
      , ConfigurableWebApplicationContext wac) {
    // 这里去解析获取 ApplicationContextInitializer 列表，这个列表即用户自己实现了
    // ApplicationContextInitializer 接口进行自定义的根上下文初始化的操作。
    // 需要配置在 web.xml 中，属性名为：globalClassNames，contextInitializerClasses
    // 此方法在后面进行说明。
    List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>>
          initializerClasses = determineContextInitializerClasses(sc);
    // 遍历 initializerClasses，做一个简单的检查
    for (Class<ApplicationContextInitializer<ConfigurableApplicationContext>> 
          initializerClass : initializerClasses) {
        Class<?> initializerContextClass =
            GenericTypeResolver
            .resolveTypeArgument(initializerClass, ApplicationContextInitializer.class);
            
        if (initializerContextClass != null && !initializerContextClass.isInstance(wac)) {
            throw new ApplicationContextException(String.format(
                "Could not apply context initializer [%s] since its generic parameter [%s] " 
                + "is not assignable from the type of application context used by this " 
                + "context loader: [%s]", initializerClass.getName(),
                initializerContextClass.getName(),
                wac.getClass().getName()));
        }
        // 实例化 initializerClass 并添加到 contextInitializers 中去。
        this.contextInitializers.add(BeanUtils.instantiateClass(initializerClass));
    }
    
	// 根据优先级进行排序。
    AnnotationAwareOrderComparator.sort(this.contextInitializers);
    // 遍历执行自定义实现的 initialize 方法。
    for (ApplicationContextInitializer<ConfigurableApplicationContext> initializer : this.contextInitializers) {
        initializer.initialize(wac);
    }
}

protected List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>>
			determineContextInitializerClasses(ServletContext servletContext) {
			
	List<Class<ApplicationContextInitializer<ConfigurableApplicationContext>>> 
		classes = new ArrayList<>();
	// 从 servletContext 中获取 globalClassNames 值
	String globalClassNames = 
		servletContext.getInitParameter(GLOBAL_INITIALIZER_CLASSES_PARAM);
	if (globalClassNames != null) {
		// 按分隔符截取并遍历
		for (String className 
		                   : StringUtils.tokenizeToStringArray(globalClassNames, 
		                                     INIT_PARAM_DELIMITERS)) {
			// 加入列表中
			classes.add(loadInitializerClass(className));
		}
	}
	// 从 ServletContext 中获取 contextInitializerClasses 值
	String localClassNames = servletContext.getInitParameter(CONTEXT_INITIALIZER_CLASSES_PARAM);
	if (localClassNames != null) {
		// 按分隔符截取并遍历
		for (String className 
		                 : StringUtils.tokenizeToStringArray(localClassNames,
                                                        INIT_PARAM_DELIMITERS)) {
		    // 加入列表中
			classes.add(loadInitializerClass(className));
		}
	}

	return classes;
}
```

```java
// class AbstractApplicationContext

// 获取上下文环境
public ConfigurableEnvironment getEnvironment() {
    // 为空则创建
    if (this.environment == null) {
        // 这里调用的是 AbstractRefreshableApplicationContext 的实现。
        // 在后面进行说明。
        this.environment = createEnvironment();
    }
    return this.environment;
}
```

```java
// class AbstractRefreshableApplicationContext

protected ConfigurableEnvironment createEnvironment() {
    // 初始化一个 StandardServletEnvironment，
    // 初始化过程将会调用其父类的父类（AbstractEnvironment）的构造方法。
    // 在后面进行说明。
    return new StandardServletEnvironment();
}
```

```java
// class AbstractEnvironment

public AbstractEnvironment() {
    // 这里调用其子类的子类 StandardServletEnvironment 的实现。
    // 在后面进行说明。
    customizePropertySources(this.propertySources);
    if (logger.isDebugEnabled()) {
        logger.debug("Initialized " + getClass().getSimpleName() 
                     + " with PropertySources " + this.propertySources);
    }
}
```

```java
// class StandardServletEnvironment

protected void customizePropertySources(MutablePropertySources propertySources) {
    // 添加一个名为 servletConfigInitParams 的 PropertySource，其内 source 属性无有效值。
    propertySources.addLast(new StubPropertySource(SERVLET_CONFIG_PROPERTY_SOURCE_NAME));
    // 添加一个名为 servletContextInitParams 的 PropertySource，其内 source 属性无有效值。
    propertySources.addLast(new StubPropertySource(SERVLET_CONTEXT_PROPERTY_SOURCE_NAME));
    // 判断Java EE 环境中的默认的 JNDI 环境是否能被 JVM 获取，是则将其添加到 propertySources 中。
    if (JndiLocatorDelegate.isDefaultJndiEnvironmentAvailable()) {
        // 创建的对象中包含一个 JndiTemplate 类型的数据，但其内部的 environment 为null。
        propertySources.addLast(new JndiPropertySource(JNDI_PROPERTY_SOURCE_NAME));
    }
    // 这里调用其父类 StandardEnvironment 的实现。
    super.customizePropertySources(propertySources);
}

public void initPropertySources(@Nullable ServletContext servletContext
                    , @Nullable ServletConfig servletConfig) {
    // 这里实现在 WebApplicationContextUtils 中，直接在下面说明。
    WebApplicationContextUtils.initServletPropertySources(getPropertySources()
          , servletContext, servletConfig);
}

// class WebApplicationContextUtils
public static void initServletPropertySources(MutablePropertySources sources,
			@Nullable ServletContext servletContext, @Nullable ServletConfig servletConfig) {

    Assert.notNull(sources, "'propertySources' must not be null");
    // servletContextInitParams
    String name = StandardServletEnvironment.SERVLET_CONTEXT_PROPERTY_SOURCE_NAME;
    if (servletContext != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
        // 替换名为 servletContextInitParams 的属性源为 ServletContextPropertySource，
        // 其实就是把 servletContext 内的 context 搬了过来。
        sources.replace(name, new ServletContextPropertySource(name, servletContext));
    }
    // servletConfigInitParams
    name = StandardServletEnvironment.SERVLET_CONFIG_PROPERTY_SOURCE_NAME;
    // 前面传入的 servletConfig 为 null，所以不执行替换了。
    if (servletConfig != null && sources.contains(name) && sources.get(name) instanceof StubPropertySource) {
        sources.replace(name, new ServletConfigPropertySource(name, servletConfig));
    }
}
```

```java
// class StandardEnvironment

protected void customizePropertySources(MutablePropertySources propertySources) {
    // 添加一个名为 systemProperties 的 PropertySource，其内 source 的值就是 getSystemProperties
    // 方法返回的值，是一个 hashTable 的结构。其内包含如 os.name 等属性信息。
    propertySources.addLast(new MapPropertySource(SYSTEM_PROPERTIES_PROPERTY_SOURCE_NAME
         , getSystemProperties()));
    
    // 添加一个名为 systemEnvironment 的 PropertySource，其内 source 的值就是 getSystemEnvironment
    // 方法返回的值，是一个 map 类型的结构。其内包含如 JAVA_OPTS 等属性信息。
    propertySources.addLast(
        new SystemEnvironmentPropertySource(SYSTEM_ENVIRONMENT_PROPERTY_SOURCE_NAME
        , getSystemEnvironment()));
}
```

```java
// class AbstractApplicationContext

public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
     // 刷新前的准备工作：
     // 1. 关闭上下文。
     // 2. 初始化 servletContextInitParams，servletConfigInitParams，
     // 与前面初始化调用的是相同的方法，虽然方法调用相同，但是此时由于条件判断，所以不再进行初始化。
     // 3. 验证需要的属性值是否存在，不存在将抛出异常，这里需要的属性值为空。
     prepareRefresh();

     // 刷新内部 beanFactory。
     // 完成了 beanFactory 的初始化及 bean 定义的创建，包括 components-scan 等都是在此完成。
     // 在下面进行说明。
     ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

     // 为 bean factory 在上下文中被使用做准备。
     // 将一些上下文的标准配置设置到 beanFactory 中去。
     prepareBeanFactory(beanFactory);

      try {
          // 这里调用的是 AbstractRefreshableWebApplicationContext 中的实现。
          // 这里设置了一些 scope: request, session。
          // 设置了一些特殊依赖类型及对应的装配值。
          // 需要注意的是在此实现中还添加了一个 ServletContextAwareProcessor，
          // 实现自 BeanPostProcessor, BeanPostProcessor 内定义了两个方法，
          // 分别表示 bean 初始化之前，及初始化之后，可实现此接口定义自己的 bean 初始化
          // 前后的逻辑处理。
          // beanFactory 中持有了此类型的一个 list。在 bean 初始化前后都会遍历此 list 中的
          // processor 并调用对应的方法。
          // 而此处添加的 ServletContextAwareProcessor，会为实现了
          // ServletContextAware 接口或 ServletConfigAware 接口的 bean 注入对应的资源：
          // ServletContext, ServletConfig。
          postProcessBeanFactory(beanFactory);

          // 如前面对 processor 的说明，这里是对 processor 的调用。但是与上面不同。如下：
          // BeanFactoryPostProcessor 接口，表示上下文内部的 bean 定义注册表的
          // 标准初始化之后的自定义行为。
          // 这里默认存在一个实现了接口的 processor：ConfigurationClassPostProcessor，
          // 此实现目标是为含 @Configuration 注解的 bean 进行处理。
          invokeBeanFactoryPostProcessors(beanFactory);

          // 注册另外一些 BeanPostProcessor，如：
          // AutowiredAnnotationBeanPostProcessor -> @autowire 相关
          // RequiredAnnotationBeanPostProcessor 等。
          registerBeanPostProcessors(beanFactory);

          // 初始化 message source。
          // 在下面进行说明。
          initMessageSource();

          // 在下面进行说明。
          initApplicationEventMulticaster();

          // 在这里仅仅是初始化了一个 themeSource
          onRefresh();

          // 将 ApplicationEvent 类型的 bean 添加到名为 applicationEventMulticaster 的对象中。
          // 广播 earlyApplicationEvents 内持有的事件。
          registerListeners();

          // 完成 beanFactory 的初始化。
          // 1. 初始化上下文的类型转换服务。
          // 2. 注册一个默认的嵌入式值解析器，
          // 3. 预先初始单例 bean，以及配置了 lazy-init=false 的 bean。
          finishBeanFactoryInitialization(beanFactory);

          // 开启生命周期，启动生命周期 processor。
          // 发布刷新完成事件。
          finishRefresh();
      }

      catch (BeansException ex) {
          if (logger.isWarnEnabled()) {
              logger.warn("Exception encountered during context initialization - " +
                          "cancelling refresh attempt: " + ex);
          }

          // Destroy already created singletons to avoid dangling resources.
          destroyBeans();

          // Reset 'active' flag.
          cancelRefresh(ex);

          // Propagate exception to caller.
          throw ex;
      }

      finally {
          // Reset common introspection caches in Spring's core, since we
          // might not ever need metadata for singleton beans anymore...
          resetCommonCaches();
      }
   }
}

protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
    // 此方法由子类实现，这里指向的是 AbstractRefreshableApplicationContext。
    // 此方法完成了bean factory 的创建及初始化，包括对 bean 定义的创建。
    // 在下面进行说明。
    refreshBeanFactory();
    // 获取创建好的 bean factory。
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    if (logger.isDebugEnabled()) {
        logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
    }
    return beanFactory;
}

protected void initMessageSource() {
    // 获取内部 bean factory。
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 判断当前 beanfactory 是否包含名为 messageSource 的 bean。
    if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
        // 获取 messageSource
        this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
        // 如果此上下文存在父级上下文且 messageSource 是有层次的 messageSource，咋进行下面的操作。
        if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
            HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
            if (hms.getParentMessageSource() == null) {
                // 设置父级上下文的 messageSource 为当前 messageSource 的父级 source
                hms.setParentMessageSource(getInternalParentMessageSource());
            }
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Using MessageSource [" + this.messageSource + "]");
        }
    }
    else {
        // 实例化一个 DelegatingMessageSource，此类型的MessageSource，不会处理任何逻辑，
        // 它只会委派其父级进行处理，若无父级将无法处理任何逻辑。
        DelegatingMessageSource dms = new DelegatingMessageSource();
        // 设置其父级。
        dms.setParentMessageSource(getInternalParentMessageSource());
        // 将 dms 赋值给 messageSource。
        this.messageSource = dms;
        // 将 messageSource 以单例模式注册到 bean factory 中去。
        beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate MessageSource with name '" 
                         + MESSAGE_SOURCE_BEAN_NAME +
                         "': using default [" + this.messageSource + "]");
        }
    }
}

protected void initApplicationEventMulticaster() {
    // 获取 bean factory
    ConfigurableListableBeanFactory beanFactory = getBeanFactory();
    // 判断是否存在名为 applicationEventMulticaster 的 bean。
    if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
        // 获取名为 applicationEventMulticaster 的 bean。
        this.applicationEventMulticaster =
            beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME,
                                ApplicationEventMulticaster.class);
        if (logger.isDebugEnabled()) {
            logger.debug("Using ApplicationEventMulticaster [" 
                         + this.applicationEventMulticaster + "]");
        }
    }
    else {
        // 实例化一个默认的 ApplicationEventMulticaster。
        this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
        // 注册一个 ApplicationEventMulticaster 的单例 bean
        beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, 
                                      this.applicationEventMulticaster);
        if (logger.isDebugEnabled()) {
            logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
                         APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
                         "': using default [" + this.applicationEventMulticaster + "]");
        }
    }
}
```

```java
// class AbstractRefreshableApplicationContext

protected final void refreshBeanFactory() throws BeansException {
    // 判断 beanFactory 是否存在，此时是不存在的。
    if (hasBeanFactory()) {
        // 销毁 bean
        destroyBeans();
        // 将 beanFactory 赋值为 null
        closeBeanFactory();
    }
    try {
        // 通过其父容器创建一个 DefaultListableBeanFactory 类型的 BeanFactory。
        DefaultListableBeanFactory beanFactory = createBeanFactory();
        // 为 BeanFactory 生成设置一个 serializationId
        beanFactory.setSerializationId(getId());
        // 用户自定义配置，但这里只包含两项属性：是否允许 bean 定义重写，是否允许循环引用
        customizeBeanFactory(beanFactory);
        // 加载 bean 定义，bean 定义是容器的核心数据，此处调用的是 XmlWebApplicationContext 中的实现
        // 在下面进行说明。
        loadBeanDefinitions(beanFactory);
        synchronized (this.beanFactoryMonitor) {
            this.beanFactory = beanFactory;
        }
    }
    catch (IOException ex) {
        throw new ApplicationContextException("I/O error parsing bean definition source for " 
                                              + getDisplayName(), ex);
    }
}
```

```java
// class XmlWebApplicationContext

protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
    // 创建一个 XmlBeanDefinitionReader 类型的 bean 定义读取器，
    // 读取器需要传入一个 BeanDefinitionRegistry 类型的参数，表示为 bean 定义注册表，
    // DefaultListableBeanFactory 正好实现了 BeanDefinitionRegistry 接口。
    XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

    // 为 bean 定义读取器设置环境
    beanDefinitionReader.setEnvironment(getEnvironment());
    // 将 自身实例作为 ResourceLoader 设置到 bean 定义读取器中，
    // 因为父类链中的 AbstractApplicationContext 继承了 DefaultResourceLoader。
    beanDefinitionReader.setResourceLoader(this);
    // EntityResolver，这是去查找解析 xml 文件的 DTD 声明文件的处理实现。
    // 使用 SAX 解析 xml 时，默认的会在网络环境下下载 DTD 声明文件，但是存在很多问题，如网络不可用，
    // 目标服务器挂掉了，这些可能性会造成下载无法继续，将抛出异常，造成配置文件无法正确读取。
    // 实现 EntityResolver 来定义自己的处理行为将避免这种情况发生。
    beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

    // 这是个模板方法，允许子类重写此方法提供自己的 reader
    initBeanDefinitionReader(beanDefinitionReader);
    // 通过给定 bean 定义读取器解析 bean定义。
    // 在下面进行说明。
    loadBeanDefinitions(beanDefinitionReader);
}

protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws IOException {
    // 获取配置文件路径如：/WEB-INF/dispatcher-servlet.xml
    String[] configLocations = getConfigLocations();
    if (configLocations != null) {
        for (String configLocation : configLocations) {
            // 具体实现在 AbstractBeanDefinitionReader 中。
            // 在下面进行说明。
            reader.loadBeanDefinitions(configLocation);
        }
    }
}
```

```java
// class AbstractBeanDefinitionReader

// 这里传入的 actualResources 为 null
public int loadBeanDefinitions(String location, @Nullable Set<Resource> actualResources) throws BeanDefinitionStoreException {
    // 获取 ResourceLoader，这里表示的是 XMLWebApplicationContext，
    // 因为其父类链中 AbstractApplicationContext 继承了DefaultResourceLoader，
    // 且 ApplicationContext 接口还继承了 ResourcePatternResolver 接口，
    // 此接口也继承了 ResourceLoader。
    // 前面 loadBeanDefinitions 方法内通过 beanDefinitionReader.setResourceLoader(this) 来
    // 设置的。
    ResourceLoader resourceLoader = getResourceLoader();
    if (resourceLoader == null) {
        throw new BeanDefinitionStoreException(
            "Cannot import bean definitions from location [" 
            + location + "]: no ResourceLoader available");
    }

    // 此处判断为 true，前面已经讲了继承关系。
    if (resourceLoader instanceof ResourcePatternResolver) {
        // Resource pattern matching available.
        try {
            // 通过配置文件路径，实例化 Resource 数组。
            Resource[] resources = ((ResourcePatternResolver) resourceLoader)
                .getResources(location);
            // 这里实际调用了 XmlBeanDefinitionReader 的 loadBeanDefinitions 方法。
            // 在下面进行说明。
            int loadCount = loadBeanDefinitions(resources);
            if (actualResources != null) {
                for (Resource resource : resources) {
                    actualResources.add(resource);
                }
            }
            if (logger.isDebugEnabled()) {
                logger.debug("Loaded " + loadCount 
                             + " bean definitions from location pattern [" + location + "]");
            }
            return loadCount;
        }
        catch (IOException ex) {
            throw new BeanDefinitionStoreException(
                "Could not resolve bean definition resource pattern [" + location + "]", ex);
        }
    }
    else {
        // Can only load single resources by absolute URL.
        Resource resource = resourceLoader.getResource(location);
        int loadCount = loadBeanDefinitions(resource);
        if (actualResources != null) {
            actualResources.add(resource);
        }
        if (logger.isDebugEnabled()) {
            logger.debug("Loaded " 
                         + loadCount 
                         + " bean definitions from location [" 
                         + location + "]");
        }
        return loadCount;
    }
}
```

```java
// class XmlBeanDefinitionReader

public int loadBeanDefinitions(EncodedResource encodedResource) 
    							throws BeanDefinitionStoreException {
    Assert.notNull(encodedResource, "EncodedResource must not be null");
    if (logger.isInfoEnabled()) {
        logger.info("Loading XML bean definitions from " 
                    + encodedResource.getResource());
    }
    // 从本地线程中获取 EncodedResource，
    // 此时为空的。
    Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
    // 判断为空设置为一个空的 set。
    if (currentResources == null) {
        currentResources = new HashSet<>(4);
        this.resourcesCurrentlyBeingLoaded.set(currentResources);
    }
    // 添加到 currentSources 中去，若无法添加，则抛出异常。
    if (!currentResources.add(encodedResource)) {
        throw new BeanDefinitionStoreException(
            "Detected cyclic loading of " 
            + encodedResource 
            + " - check your import definitions!");
    }
    try {
        // 读取资源流，这里返回的是 ByteArrayInputStream 类型的实例。
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            // 实例化 InputSource。
            InputSource inputSource = new InputSource(inputStream);
            // 若存在编码则设置编码。
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // 这里调用的是 XmlBeanDefinitionReader 的另一个方法，返回载入 bean 定义个数。
            // 在下面进行描述。
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    catch (IOException ex) {
        throw new BeanDefinitionStoreException(
            "IOException parsing XML document from " 
            + encodedResource.getResource(), ex);
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
    try {
        // 得到文档对象。
        Document doc = doLoadDocument(inputSource, resource);
        // 注册 bean 定义，返回载入 bean 的个数。
        // 在后面进行说明。
        return registerBeanDefinitions(doc, resource);
    }
    catch (BeanDefinitionStoreException ex) {
        throw ex;
    }
    // ... 省略后面的 catch 块。
}

public int registerBeanDefinitions(Document doc, Resource resource) 
    		throws BeanDefinitionStoreException {
    // 实例化 BeanDefinitionDocumentReader 对象。
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    // 获取已经存在于上下文（准确说是 BeanDefinitionRegistry, 但是上下文实现了此接口）
    // 中的 BeanDefinition 个数。
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 完成 bean 定义的读取。不做过多的说明，这是 xml 文件的读取，
    // 读取后解析读取的信息，针对不同的元素类型会存在不同的 parser 来解析成 bean 定义，
    // 如：components-scan 就会使用 ComponentScanBeanDefinitionParser 来解析。
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
  
    return getRegistry().getBeanDefinitionCount() - countBefore;
}
```
上面的所有操作完成后标志着 根上下文初始化完成，接下来是 `DispatcherServlet` 的初始化过程，如前面 `ContextLoader` 一样，利用静态块优先读取策略文件：

```java
static {
    try {
        ClassPathResource resource = new ClassPathResource(DEFAULT_STRATEGIES_PATH
                                    , DispatcherServlet.class);
        defaultStrategies = PropertiesLoaderUtils.loadProperties(resource);
    }
    catch (IOException ex) {
        throw new IllegalStateException("Could not load '" + DEFAULT_STRATEGIES_PATH 
                                        + "': " + ex.getMessage());
    }
}
```

文件 `org\springframework\web\servlet\DispatcherServlet.properties` 内容如下：

```properties
org.springframework.web.servlet.LocaleResolver=org.springframework.web.servlet.i18n.AcceptHeaderLocaleResolver

org.springframework.web.servlet.ThemeResolver=org.springframework.web.servlet.theme.FixedThemeResolver

org.springframework.web.servlet.HandlerMapping=org.springframework.web.servlet.handler.BeanNameUrlHandlerMapping,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerMapping

org.springframework.web.servlet.HandlerAdapter=org.springframework.web.servlet.mvc.HttpRequestHandlerAdapter,\
	org.springframework.web.servlet.mvc.SimpleControllerHandlerAdapter,\
	org.springframework.web.servlet.mvc.method.annotation.RequestMappingHandlerAdapter

org.springframework.web.servlet.HandlerExceptionResolver=org.springframework.web.servlet.mvc.method.annotation.ExceptionHandlerExceptionResolver,\
	org.springframework.web.servlet.mvc.annotation.ResponseStatusExceptionResolver,\
	org.springframework.web.servlet.mvc.support.DefaultHandlerExceptionResolver

org.springframework.web.servlet.RequestToViewNameTranslator=org.springframework.web.servlet.view.DefaultRequestToViewNameTranslator

org.springframework.web.servlet.ViewResolver=org.springframework.web.servlet.view.InternalResourceViewResolver

org.springframework.web.servlet.FlashMapManager=org.springframework.web.servlet.support.SessionFlashMapManager
```

然后 `DispatcherServlet` 进行初始化：

```java
public DispatcherServlet() {
    super();
    // 设置是否将 OPTIONS 请求分派给 doService 方法。
    setDispatchOptionsRequest(true);
}
```

`web` 容器回调 `servlet` 的 `init` 方法，这里 `DispatcherServlet` 并没有实现此方法，而是其父类的父类 `HttpServletBean` 实现了此方法：

```java
// class HttpServletBean

public final void init() throws ServletException {
    if (logger.isDebugEnabled()) {
        logger.debug("Initializing servlet '" + getServletName() + "'");
    }

    // 将 ServletConfig 中的 parameters 封装为 PropertyValues，
    // 另外一个参数 requiredProperties 表示这些属性时必须存在的，
    // 若在 ServletConfig 中不存在对应的值，将抛出异常。
    PropertyValues pvs = new ServletConfigPropertyValues(getServletConfig(),
                         this.requiredProperties);
    if (!pvs.isEmpty()) {
        try {
            // 将 this 包装，这种包装可以方便属性的操作等。
            BeanWrapper bw = PropertyAccessorFactory.forBeanPropertyAccess(this);
            // 实例化一个 resourceLoader
            ResourceLoader resourceLoader 
                		= new ServletContextResourceLoader(getServletContext());
            // 通过 ResourceLoader 实例化一个 resourceEditor，
            // 将 editor 注册到 beanWrapper, 这表示为 this 中 Resource 类型的
            // 属性将使用对应的 editor 来进行处理。
            bw.registerCustomEditor(Resource.class, 
                       new ResourceEditor(resourceLoader, getEnvironment()));
            // 初始化 beanWrapper
            initBeanWrapper(bw);
            // 设置 PropertyValues
            bw.setPropertyValues(pvs, true);
        }
        // ... 省略 catch 块。
    }

    // 交给子类实现自己的初始化操作，这里调用的是其子类 FrameworkServlet 中的实现。
    // 在下面进行说明。
    initServletBean();

    if (logger.isDebugEnabled()) {
        logger.debug("Servlet '" + getServletName() + "' configured successfully");
    }
}
```

```java
// class FrameworkServlet

protected final void initServletBean() throws ServletException {
    getServletContext().log("Initializing Spring FrameworkServlet '" 
                            + getServletName() + "'");
    if (this.logger.isInfoEnabled()) {
        this.logger.info("FrameworkServlet '" + getServletName() 
                         + "': initialization started");
    }
    long startTime = System.currentTimeMillis();

    try {
        // 初始化 WebApplicationContext，这与前面的根上下文不同，
        // 这里是创建一个新的上下文。
        // 在下面进行说明。
        this.webApplicationContext = initWebApplicationContext();
        // 这里默认实现无任何操作。
        initFrameworkServlet();
    }
    catch (ServletException | RuntimeException ex) {
        this.logger.error("Context initialization failed", ex);
        throw ex;
    }

    if (this.logger.isInfoEnabled()) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        this.logger.info("FrameworkServlet '" + getServletName() 
                         + "': initialization completed in " +
                         elapsedTime + " ms");
    }
}

protected WebApplicationContext initWebApplicationContext() {
    // 获取根上下文。
    WebApplicationContext rootContext =
        WebApplicationContextUtils.getWebApplicationContext(getServletContext());
    WebApplicationContext wac = null;

    // 若存在构造时注入一个上下文的情况，直接使用。
    if (this.webApplicationContext != null) {
        // 直接赋值。
        wac = this.webApplicationContext;
        // 为其设置父级上下文。
        if (wac instanceof ConfigurableWebApplicationContext) {
            ConfigurableWebApplicationContext cwac = (ConfigurableWebApplicationContext) wac;
            if (!cwac.isActive()) {
                // The context has not yet been refreshed -> provide services such as
                // setting the parent context, setting the application context id, etc
                if (cwac.getParent() == null) {
                    // The context instance was injected without an explicit parent -> set
                    // the root application context (if any; may be null) as the parent
                    cwac.setParent(rootContext);
                }
                // 同样的进行配置和上下文的初始化。
                configureAndRefreshWebApplicationContext(cwac);
            }
        }
    }
    if (wac == null) {
        // 查看根根上下文是否存在一个bean name与变量 contextAttribute 的值相同的 bean，存在则返回。
        wac = findWebApplicationContext();
    }
    if (wac == null) {
        // 无上下文实例注入，创建一个上下文。
        // 同根上下文初始化操作，也会读取 xml 文件并初始化单例 bean。
        wac = createWebApplicationContext(rootContext);
    }

    if (!this.refreshEventReceived) {
        // Either the context is not a ConfigurableApplicationContext with refresh
        // support or the context injected at construction time had already been
        // refreshed -> trigger initial onRefresh manually here.
        onRefresh(wac);
    }

    if (this.publishContext) {
        // Publish the context as a servlet context attribute.
        String attrName = getServletContextAttributeName();
        getServletContext().setAttribute(attrName, wac);
        if (this.logger.isDebugEnabled()) {
            this.logger.debug("Published WebApplicationContext of servlet '" 
                              + getServletName() +
                              "' as ServletContext attribute with name [" + attrName + "]");
        }
    }

    return wac;
}
```

这里应用就已经启动完成了。总结一下：

1）根上下文初始化。

2）`DispatcherServlet` 上下文初始化。

3）这两个上下文初始化完成的基本内容相同。

4）根上下文会作为`DispatcherServlet` 上下文的父级上下文。

