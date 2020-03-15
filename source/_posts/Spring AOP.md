---
title: Spring AOP
date: 2018-04-27 19:54:55
subtitle:
tags: 
	- java
	- spring
catagories: spring
---

# AOP 的基本概念

- Aspect (切面): 通常是一个类，里面可以定义切入点和通知


- JointPoint (连接点): 程序执行过程中明确的点，一般是方法的调用


- Advice (通知): AOP 在特定的切入点上执行的增强处理，有 `before`, `after`, `afterReturning`, `afterThrowing`, `around`


- Pointcut (切入点): 就是带有通知的连接点，在程序中主要体现为书写切入点表达式


- AOP 代理: AOP 框架创建的对象，代理就是目标对象的加强。Spring 中的AOP 代理可以使 JDK 动态代理，也可以是 CGLIB 代理，前者基于接口，后者基于子类

  

```java
package com.offves.aop;

import org.junit.Test;
import org.springframework.context.annotation.*;

/**
 *
 * @author offves
 * @since 2018-3-19 22:07
 */
@EnableAspectJAutoProxy
@Configuration
@ComponentScan("com.offves.aop")
public class AopConfig {
    @Bean
    public Calculator calculator() {
        return new Calculator();
    }

    @Bean
    public LogAspect logAspects(){
        return new LogAspect();
    }

    @Test
    public void testAop() {
        ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AopConfig.class);
        Calculator calculator = applicationContext.getBean(Calculator.class);
        int sum = calculator.sum(1, 1);

        applicationContext.close();
    }

}
```

```java
package com.offves.aop;

/**
 *
 * @author offves
 * @since 2018-3-19 22:09
 */
public class Calculator {
    public int sum(int i, int j) {
        return i + j;
    }
}
```

```java
package com.offves.aop;

import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.annotation.*;

import java.util.Arrays;

/**
 *
 * @author offves
 * @since 2018-3-19 22:10
 */
@Aspect
public class LogAspect {

    @Pointcut("execution(public int com.offves.aop.Calculator.*(..))")
    public void pointCut() {}

    @Before("pointCut()")
    public void before(JoinPoint joinPoint) {
        Object[] args = joinPoint.getArgs();
        System.out.println(joinPoint.getSignature().getName() + "运行。。。@Before:参数列表是：{" + Arrays.asList(args) + "}");
    }

    @After("pointCut()")
    public void after(JoinPoint joinPoint) {
        System.out.println(joinPoint.getSignature().getName() + "结束。。。@After");
    }

    @AfterReturning(value = "pointCut()", returning = "result")
    public void afterReturning(JoinPoint joinPoint, Object result) {
        System.out.println(joinPoint.getSignature().getName() + "正常返回。。。@AfterReturning:运行结果：{" + result + "}");
    }

    @AfterThrowing(value = "pointCut()", throwing = "exception")
    public void afterThrowing(JoinPoint joinPoint, Exception exception) {
        System.out.println(joinPoint.getSignature().getName() + "异常。。。异常信息：{" + exception + "}");
    }
}
```

```java
sum运行。。。@Before:参数列表是：{[1, 1]}
sum结束。。。@After
sum正常返回。。。@AfterReturning:运行结果：{2}
```



# 注册 AspectJ 自动代理创建器定义信息

> **`@EnableAspectJAutoProxy`** 开启 AspectJ 自动代理

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import(AspectJAutoProxyRegistrar.class) // 导入 AspectJAutoProxyRegistrar Bean 定义注册器
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
    boolean exposeProxy() default false;
}
```

```java
/**
 * Bean 定义注册器, IOC 容器启动时会调用 Bean 定义注册器的 registerBeanDefinitions(),
 * 该方法可以手动注册 Bean 定义 到 IOC 容器中
 */
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

    /**
     * Register, escalate, and configure the AspectJ auto proxy creator based on the value
     * of the @{@link EnableAspectJAutoProxy#proxyTargetClass()} attribute on the importing
     * {@code @Configuration} class.
     */
    @Override
    public void registerBeanDefinitions(
        AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {

        // 向 IOC 容器中注册一个 AspectJ 自动代理创建器的 Bean 定义信息
        // org.springframework.aop.config.internalAutoProxyCreator = AnnotationAwareAspectJAutoProxyCreator
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);

        AnnotationAttributes enableAspectJAutoProxy =
            AnnotationConfigUtils.attributesFor(importingClassMetadata, EnableAspectJAutoProxy.class);
        if (enableAspectJAutoProxy.getBoolean("proxyTargetClass")) {
            AopConfigUtils.forceAutoProxyCreatorToUseClassProxying(registry);
        }
        if (enableAspectJAutoProxy.getBoolean("exposeProxy")) {
            AopConfigUtils.forceAutoProxyCreatorToExposeProxy(registry);
        }
    }
}
```

```java
public abstract class AopConfigUtils {
    public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry) {
        return registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry, null);
    }

    public static BeanDefinition registerAspectJAnnotationAutoProxyCreatorIfNecessary(BeanDefinitionRegistry registry, Object source) {
        return registerOrEscalateApcAsRequired(AnnotationAwareAspectJAutoProxyCreator.class, registry, source);
    }

    private static BeanDefinition registerOrEscalateApcAsRequired
        (Class << ? > cls, BeanDefinitionRegistry registry, Object source) {

            Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
            if (registry.containsBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME)) {
                BeanDefinition apcDefinition = registry.getBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME);
                if (!cls.getName().equals(apcDefinition.getBeanClassName())) {
                    int currentPriority = findPriorityForClass(apcDefinition.getBeanClassName());
                    int requiredPriority = findPriorityForClass(cls);
                    if (currentPriority < requiredPriority) {
                        apcDefinition.setBeanClassName(cls.getName());
                    }
                }
                return null;
            }
            // 创建一个 Bean 定义信息
            RootBeanDefinition beanDefinition = new RootBeanDefinition(cls);
            beanDefinition.setSource(source);
            beanDefinition.getPropertyValues().add("order", Ordered.HIGHEST_PRECEDENCE);
            beanDefinition.setRole(BeanDefinition.ROLE_INFRASTRUCTURE);
            /**
             * 向 BeanFactory 中注册 Bean 定义信息
             * org.springframework.aop.config.internalAutoProxyCreator = AnnotationAwareAspectJAutoProxyCreator
             */
            registry.registerBeanDefinition(AUTO_PROXY_CREATOR_BEAN_NAME, beanDefinition);
            return beanDefinition;
        }
}
```

## 创建 IOC 容器

```java
// 创建 IOC 容器
ApplicationContext applicationContext = new AnnotationConfigApplicationContext(AopConfig.class);

```

## 容器刷新

```java
public class AnnotationConfigApplicationContext extends GenericApplicationContext implements AnnotationConfigRegistry {
    public AnnotationConfigApplicationContext(Class << ? > ...annotatedClasses) {
        this();
        register(annotatedClasses);
        // 刷新 IOC 容器
        refresh();
    }
}
```

## 执行 BeanFactory 的后置理器

> `BeanPostProcessor ` Bean 后置处理器，Bean 创建对象初始化前后调用
>
> `BeanFactoryPostProcessor` BeanFactory 的后置处理器, 在 BeanFactory 标准初始化之后调用，来定制和修改 BeanFactory 的内容, 所有的 Bean 定义已经保存加载到 BeanFactory，但是 Bean 的实例还未创建时调用postProcessBeanFactory(ConfigurableListableBeanFactory)
>
> `BeanDefinitionRegistryPostProcessor` extends `BeanFactoryPostProcessor`, 在所有 Bean 定义信息将要被加载，Bean 实例还未创建时先调用 postProcessBeanDefinitionRegistry(BeanDefinitionRegistry), 再调用postProcessBeanFactory(ConfigurableListableBeanFactory)

```java
public abstract class AbstractApplicationContext
extends DefaultResourceLoader implements ConfigurableApplicationContext, DisposableBean {
    @Override
    public void refresh() throws BeansException, IllegalStateException {
        synchronized(this.startupShutdownMonitor) {
            ......
            try {
                // Allows post-processing of the bean factory in context subclasses.
                postProcessBeanFactory(beanFactory);

                // Invoke factory processors registered as beans in the context.
                /*
                 * 1 执行 BeanFactory 的后置理器
                 * 1.1 先执行实现了 BeanDefinitionRegistryPostProcessor 的 postProcessBeanDefinitionRegistry() 方法
                 * 1.2 再执行实现了 BeanDefinitionRegistryPostProcessor 的 postProcessBeanFactory() 方法, BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor
                 * 1.3 再执行实现了 BeanFactoryPostProcessor 的 postProcessBeanFactory() 方法
                 */
                invokeBeanFactoryPostProcessors(beanFactory);

                // Register bean processors that intercept bean creation.
                registerBeanPostProcessors(beanFactory);
                ......
                // Instantiate all remaining (non-lazy-init) singletons.
                finishBeanFactoryInitialization(beanFactory);

                // Last step: publish corresponding event.
                finishRefresh();
            } catch (BeansException ex) {
                ......
            } finally {
                ......
            }
        }
    }

    protected void invokeBeanFactoryPostProcessors
        (ConfigurableListableBeanFactory beanFactory) {
            PostProcessorRegistrationDelegate.invokeBeanFactoryPostProcessors(beanFactory, getBeanFactoryPostProcessors());
            ......
        }
}
```

```java
class PostProcessorRegistrationDelegate {
    public static void invokeBeanFactoryPostProcessors(ConfigurableListableBeanFactory beanFactory, List < BeanFactoryPostProcessor > beanFactoryPostProcessors) {

        // Invoke BeanDefinitionRegistryPostProcessors first, if any.
        Set < String > processedBeans = new HashSet < String > ();

        if (beanFactory instanceof BeanDefinitionRegistry) {
            BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
            List < BeanFactoryPostProcessor > regularPostProcessors = new LinkedList < BeanFactoryPostProcessor > ();
            List < BeanDefinitionRegistryPostProcessor > registryPostProcessors = new LinkedList < BeanDefinitionRegistryPostProcessor > ();

            for (BeanFactoryPostProcessor postProcessor: beanFactoryPostProcessors) {
                if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
                    BeanDefinitionRegistryPostProcessor registryPostProcessor = (BeanDefinitionRegistryPostProcessor) postProcessor;
                    registryPostProcessor.postProcessBeanDefinitionRegistry(registry);
                    registryPostProcessors.add(registryPostProcessor);
                } else {
                    regularPostProcessors.add(postProcessor);
                }
            }

            // Do not initialize FactoryBeans here: We need to leave all regular beans
            // uninitialized to let the bean factory post-processors apply to them!
            // Separate between BeanDefinitionRegistryPostProcessors that implement
            // PriorityOrdered, Ordered, and the rest.
            /*
             * 1 从 IOC 容器中拿出所有实现了 BeanDefinitionRegistryPostProcessor 接口的 Bean, 
             * 挨个执行 BeanDefinitionRegistryPostProcessor.postProcessBeanDefinitionRegistry()
             * 1.1 检查该 Bean 是否实现了 PriorityOrdered, 如果实现了就优先执行,
             * 1.2 其次实现了 Ordered 的执行,
             * 1.3 再执行普通的,
             * 2 执行 BeanDefinitionRegistryPostProcessor.postProcessBeanFactory()
             */
            String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);

            // First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
            List < BeanDefinitionRegistryPostProcessor > priorityOrderedPostProcessors = new ArrayList < BeanDefinitionRegistryPostProcessor > ();
            for (String ppName: postProcessorNames) {
                if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                    priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
            registryPostProcessors.addAll(priorityOrderedPostProcessors);
            // 1.1 先执行实现 PriorityOrdered 接口的 Bean
            invokeBeanDefinitionRegistryPostProcessors(priorityOrderedPostProcessors, registry);

            // Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
            postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
            List < BeanDefinitionRegistryPostProcessor > orderedPostProcessors = new ArrayList < BeanDefinitionRegistryPostProcessor > ();
            for (String ppName: postProcessorNames) {
                if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
                    orderedPostProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
                    processedBeans.add(ppName);
                }
            }
            sortPostProcessors(beanFactory, orderedPostProcessors);
            registryPostProcessors.addAll(orderedPostProcessors);
            // 1.2 再执行实现 Ordered 接口的 Bean
            invokeBeanDefinitionRegistryPostProcessors(orderedPostProcessors, registry);

            // Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
            // 1.3 最后执行普通的 Bean
            boolean reiterate = true;
            while (reiterate) {
                reiterate = false;
                postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
                for (String ppName: postProcessorNames) {
                    if (!processedBeans.contains(ppName)) {
                        BeanDefinitionRegistryPostProcessor pp = beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class);
                        registryPostProcessors.add(pp);
                        processedBeans.add(ppName);
                        pp.postProcessBeanDefinitionRegistry(registry);
                        reiterate = true;
                    }
                }
            }

            // Now, invoke the postProcessBeanFactory callback of all processors handled so far.
            // 2. 执行 BeanDefinitionRegistryPostProcessor.postProcessBeanFactory(),
            // BeanDefinitionRegistryPostProcessor extends BeanFactoryPostProcessor 
            invokeBeanFactoryPostProcessors(registryPostProcessors, beanFactory);
            invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
        } else {
            // Invoke factory processors registered with the context instance.
            invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
        }

        // Do not initialize FactoryBeans here: We need to leave all regular beans uninitialized to let the bean factory post-processors apply to them!
        /*
         * 3 从 IOC 容器中拿出所有实现了 BeanFactoryPostProcessor 接口的 Bean, 
         * 挨个执行 BeanDefinitionRegistryPostProcessor.postProcessBeanFactory()
         * 3.1 检查该 Bean 是否实现了 PriorityOrdered, 如果实现了就优先执行,
         * 3.2 其次实现了 Ordered 的执行,
         * 3.3 再执行普通的
         */
        String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

        // Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
        // Ordered, and the rest.
        List < BeanFactoryPostProcessor > priorityOrderedPostProcessors = new ArrayList < BeanFactoryPostProcessor > ();
        List < String > orderedPostProcessorNames = new ArrayList < String > ();
        List < String > nonOrderedPostProcessorNames = new ArrayList < String > ();
        for (String ppName: postProcessorNames) {
            if (processedBeans.contains(ppName)) {
                // skip - already processed in first phase above
            } else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
                priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
            } else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
                orderedPostProcessorNames.add(ppName);
            } else {
                nonOrderedPostProcessorNames.add(ppName);
            }
        }

        // First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
        sortPostProcessors(beanFactory, priorityOrderedPostProcessors);
        // 3.1 先执行实现 PriorityOrdered 接口的 Bean
        invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

        // Next, invoke the BeanFactoryPostProcessors that implement Ordered.
        List < BeanFactoryPostProcessor > orderedPostProcessors = new ArrayList < BeanFactoryPostProcessor > ();
        for (String postProcessorName: orderedPostProcessorNames) {
            orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        sortPostProcessors(beanFactory, orderedPostProcessors);
        // 3.2 再执行实现 Ordered 接口的 Bean
        invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

        // Finally, invoke all other BeanFactoryPostProcessors.
        List < BeanFactoryPostProcessor > nonOrderedPostProcessors =
            new ArrayList < BeanFactoryPostProcessor > ();
        for (String postProcessorName: nonOrderedPostProcessorNames) {
            nonOrderedPostProcessors.add(beanFactory
                .getBean(postProcessorName, BeanFactoryPostProcessor.class));
        }
        // 3.3 最后执行普通的 Bean
        invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

        // Clear cached merged bean definitions since the post-processors might have
        // modified the original metadata, e.g. replacing placeholders in values...
        beanFactory.clearMetadataCache();
    }

    /**
     * Invoke the given BeanDefinitionRegistryPostProcessor beans.
     */
    private static void invokeBeanDefinitionRegistryPostProcessors(
        Collection < ? extends BeanDefinitionRegistryPostProcessor > postProcessors, BeanDefinitionRegistry registry) {
        /**
         * 这里会有一 spring 默认注册的 BeanDefinitionRegistryPostProcessor,
         * ConfigurationClassPostProcessor 其作用是解析所有配置类(@Configuration),
         * 注册配置类中的 Bean 定义信息
         */
        for (BeanDefinitionRegistryPostProcessor postProcessor: postProcessors) {
            postProcessor.postProcessBeanDefinitionRegistry(registry);
        }
    }
}
```

## 解析配置类

> `ConfigurationClassPostProcessor` 负责解析处理所有 @Configuration 类，并将 Bean 定义注册到  BeanFactory 中

```java
/**
 * 负责解析处理所有 @Configuration 类，并将 Bean 定义注册到  BeanFactory 中
 */
public class ConfigurationClassPostProcessor implements BeanDefinitionRegistryPostProcessor,
PriorityOrdered, ResourceLoaderAware, BeanClassLoaderAware, EnvironmentAware {
    /**
     * Derive further bean definitions from the configuration classes in the registry.
     */
    @Override
    public void postProcessBeanDefinitionRegistry(BeanDefinitionRegistry registry) {
        ......
        processConfigBeanDefinitions(registry);
    }

    /**
     * Build and validate a configuration model based on the registry of
     * {@link Configuration} classes.
     */
    public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
        // 候选配置类集合
        List <BeanDefinitionHolder> configCandidates = new ArrayList <BeanDefinitionHolder> ();
        // 1 获取到当前 IOC 容器中已有的 Bean 定义信息的 name
        String[] candidateNames = registry.getBeanDefinitionNames();

        // 2 遍历拿到 Bean 的定义, 如果该 Bean 是配置类, 加入到配置类集合中
        for (String beanName: candidateNames) {
            BeanDefinition beanDef = registry.getBeanDefinition(beanName);
            // 2.1 如果 BeanDefinition 中的 configurationClass 属性为 full 或者 lite  ,则意味着已经处理过了,直接跳过
            if (ConfigurationClassUtils.isFullConfigurationClass(beanDef) ||
                ConfigurationClassUtils.isLiteConfigurationClass(beanDef)) {
                if (logger.isDebugEnabled()) {
                    logger.debug("Bean definition has already been processed as a configuration class: " + beanDef);
                }
            } else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
                // 2.2 如果是配置类, 则加入到候选配置类集合
                configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
            }
        }

        // Return immediately if no @Configuration classes were found
        // 3 不存在配置类,则直接 return
        if (configCandidates.isEmpty()) {
            return;
        }

        // Sort by previously determined @Order value, if applicable
        // 4 把候选配置类进行排序, 按照 @Order 配置的值进行排序
        Collections.sort(configCandidates, new Comparator <BeanDefinitionHolder> () {
            @Override
            public int compare(BeanDefinitionHolder bd1, BeanDefinitionHolder bd2) {
                int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
                int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
                return (i1 < i2) ? -1 : (i1 > i2) ? 1 : 0;
            }
        });

        // Detect any custom bean name generation strategy supplied through the enclosing application context
        SingletonBeanRegistry sbr = null;
        if (registry instanceof SingletonBeanRegistry) {
            sbr = (SingletonBeanRegistry) registry;
            if (!this.localBeanNameGeneratorSet && sbr.containsSingleton(CONFIGURATION_BEAN_NAME_GENERATOR)) {
                BeanNameGenerator generator = (BeanNameGenerator) sbr.getSingleton(CONFIGURATION_BEAN_NAME_GENERATOR);
                this.componentScanBeanNameGenerator = generator;
                this.importBeanNameGenerator = generator;
            }
        }

        // Parse each @Configuration class
        /**
         * 5 解析每个配置类(@Configuration), 对配置类中的信息进行处理(要注册哪些 Bean 定义到容器中等)
         */
        // 5.1 配置类解析器
        ConfigurationClassParser parser = new ConfigurationClassParser(
            this.metadataReaderFactory, this.problemReporter, this.environment,
            this.resourceLoader, this.componentScanBeanNameGenerator, registry);

        // 对配置类进行去重
        Set <BeanDefinitionHolder> candidates = new LinkedHashSet <BeanDefinitionHolder> (configCandidates);
        // 标识以处理的配置类
        Set <ConfigurationClass> alreadyParsed = new HashSet <ConfigurationClass> (configCandidates.size());
        // 5.2 进行解析
        do {
            parser.parse(candidates);
            parser.validate();

            Set <ConfigurationClass> configClasses = new LinkedHashSet <ConfigurationClass> (parser.getConfigurationClasses());
            configClasses.removeAll(alreadyParsed);

            // Read the model and create bean definitions based on its content
            if (this.reader == null) {
                this.reader = new ConfigurationClassBeanDefinitionReader(
                    registry, this.sourceExtractor, this.resourceLoader, this.environment,
                    this.importBeanNameGenerator, parser.getImportRegistry());
            }
            // 加载配置类中的 Bean 定义信息
            this.reader.loadBeanDefinitions(configClasses);
            alreadyParsed.addAll(configClasses);
            ......
        } while (!candidates.isEmpty());
    }
}
```

```java
class ConfigurationClassBeanDefinitionReader {
    /**
     * Read {@code configurationModel}, registering bean definitions
     * with the registry based on its contents.
     */
    public void loadBeanDefinitions(Set <ConfigurationClass > configurationModel) {
        TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
        for (ConfigurationClass configClass: configurationModel) {
            loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
        }
    }

    /**
     * Read a particular {@link ConfigurationClass}, registering bean definitions
     * for the class itself and all of its {@link Bean} methods.
     */
    private void loadBeanDefinitionsForConfigurationClass(ConfigurationClass configClass,
        TrackedConditionEvaluator trackedConditionEvaluator) {
        // 1. 使用条件注解判断是否需要跳过这个配置类
        if (trackedConditionEvaluator.shouldSkip(configClass)) {
            String beanName = configClass.getBeanName();
            // 1.1 跳过配置类的话在Spring容器中移除bean的注册
            if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
                this.registry.removeBeanDefinition(beanName);
            }
            // 1.2 从importRegistry 进行删除
            this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
            return;
        }

        if (configClass.isImported()) {
            registerBeanDefinitionForImportedConfigurationClass(configClass);
        }
        // 1.3 遍历配置类中标注了 @Bean 的方法, 把对应的 Bean 定义注册到容器中
        for (BeanMethod beanMethod: configClass.getBeanMethods()) {
            loadBeanDefinitionsForBeanMethod(beanMethod);
        }
        // 1.4 从导入的资源加载 Bean 定义(@ImportResource())
        loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
        // 1.5 从 ImportBeanDefinitionRegistrar 中加载 Bean 定义(@Import(AspectJAutoProxyRegistrar.class))
        loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
    }

    private void loadBeanDefinitionsFromRegistrars(Map <ImportBeanDefinitionRegistrar, AnnotationMetadata > registrars) {
        for (Map.Entry <ImportBeanDefinitionRegistrar, AnnotationMetadata > entry: registrars.entrySet()) {
            entry.getKey().registerBeanDefinitions(entry.getValue(), this.registry);
        }
    }
}
```

```java
class AspectJAutoProxyRegistrar implements ImportBeanDefinitionRegistrar {

    @Override
    public void registerBeanDefinitions(
        AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {
        // 注册 AnnotationAwareAspectJAutoProxyCreator 后置处理器的定义信息
        AopConfigUtils.registerAspectJAnnotationAutoProxyCreatorIfNecessary(registry);
        
        ......
    }
}
```

> 以上内容是 `@EnableAspectJAutoProxy` 利用 `AspectJAutoProxyRegistrar` Bean 定义注册器注册
>
> `AnnotationAwareAspectJAutoProxyCreator` AspectJ 自动代理创建器定义信息的过程.



# 创建 AspectJ 自动代理创建器

> `AnnotationAwareAspectJAutoProxyCreator` 是一个 `InstantiationAwareBeanPostProcessor` 类型后置处理器, 会在所有 Bean 实例化前后分别调用实现了该类的 postProcessBeforeInstantiation () 和 postProcessAfterInstantiation () 方法。

> UML 图

![AnnotationAwareAspectJAutoProxyCreator UML](/Users/offves/Projects/blog/source/_posts/images/AnnotationAwareAspectJAutoProxyCreator UML.png)



> `BeanFactoryAware` 实现了 BeanFactoryAware 接口的类，可以通过 setBeanFactory() 获取加载该 Bean 的 BeanFactory。
>
> `BeanPostProcessor` 在 Bean 初始化(即调用 setter )前后，会分别调用实现了该接口的类中的postProcessBeforeInitialization () 和 postProcessAfterInitialization () 方法，实现初始化的逻辑控制。
>
> `InstantiationAwareBeanPostProcessor` 在 Bean 实例化(即调用构造函数)前后，会分别调用实现了该接口的类中的 postProcessBeforeInstantiation () 和 postProcessAfterInstantiation () 方法。



