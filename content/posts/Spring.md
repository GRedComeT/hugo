---
title: Spring 主要源码分析 
subtitle:
date: 2024-05-13T11:26:20+08:00
slug: 6e2a775
draft: false
author:
  name: Shiping Guo
  link:
  email:
  avatar:
description:
keywords:
license:
comment: false
weight: 0
tags:
  - Java
  - Spring
categories:
  - Java
hiddenFromHomePage: false
hiddenFromSearch: false
hiddenFromRss: false
hiddenFromRelated: false
summary:
resources:
  - name: featured-image
    src: featured-image.jpg
  - name: featured-image-preview
    src: featured-image-preview.jpg
toc: true
math: false
lightgallery: false
password:
message:
repost:
  enable: true
  url:

# See details front matter: https://fixit.lruihao.cn/documentation/content-management/introduction/#front-matter
---

Bean的生命周期为：BeanFactory初始化 - Bean注册 - 实例化 - 属性注入 - 初始化 - 后处理

# Bean的注册

1.  扫描：Spring通过配置（XML配置或Java配置）或自动扫描（`@ComponentScan`）来发现应用中定义的Bean。对于自动扫描，Spring会在指定的包路径下查找标注了`@Component`、`@Service`、`@Repository`、`@Controller`等注解的类
2.  解析：一旦Bean被发现，Spring将解析Bean的定义信息，包括Bean的作用域（如单例、原型）、生命周期回调（如`@PostConstruct`、`@PreDestroy`注解方法）、依赖注入的需求（通过`@Autowired`、`@Resource`等注解标记）等 ------ `BeanDefinition`
3.  注册：Spring将Bean的定义信息注册到`BeanDefinitionRegistry`中。这是一个重要步骤，因为注册后的Bean定义将被用于后续的Bean实例化和依赖注入过程。此时，Bean还没有被实例化。

## BeanDefinition

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/BeanDefinition.webp"
alt="BeanDefinition" />
<figcaption aria-hidden="true">BeanDefinition</figcaption>
</figure>

Spring在初始化过程中，先收集所有bean的元数据信息并注册，bean的元数据描述为接口`BeanDefinition`，该接口定义了你能想到的一切有关bean的属性信息

**BeanDefinition**衍生出一系列实现类
+ AbstractBeanDefinition: 如同其他Spring类，大部分**BeanDefinition**接口的逻辑都由该抽象类实现
+ GenericBeanDefinition: 是一站式、用户可见的bean definition；可见的bean definition意味着可以在该bean definition上定义post-processor来对bean进行操作
+ RootBeanDefinition: 当bean definition存在父子关系的时候，**RootBeanDefinition**用来承载父元数据的角色（也可独立存在），同时它也作为一个可合并的bean definition使用，在Spring初始化阶段，所有的bean definition均会被（向父级）合并为**RootBeanDefinition**，子bean definition（**GenericBeanDefinition**/**ChildBeanDefinition**）中的定义会覆盖其父bean definition（由**parentName**指定）的定义
+ AnnotatedBeanDefinition: 用来定义注解Bean Definition

**BeanDefinitionHolder**只是简单捆绑了BeanDefinition、bean-name、bean-alias，用于注册BeanDefinition及别名alias \^BeanDefinitionHolder

## BeanRegistry

Bean的注册逻辑分为两步，一为**BeanDefinition**的注册，二为别名的注册
+ **BeanDefinition**注册的定义在***BeanDefinitionRegistry#registerBeanDefinition***，其实现使用一个***Map\<String, BeanDefinition\>*** 来保存bean-name和BeanDefinition的关系
- 别名的注册定义在***AliasRegistry#registerAlias***，其实现同样使用一个***Map\<String, String\>*** 来保存别名alias-name和bean-name（或另一个别名alias-name）的关系

注意Bean的注册时机，通常应该在应用上下文的刷新过程之前进行(`onRefresh()`)。一旦上下文被刷新，对Bean定义的任何修改可能不会被识别，或者可能会导致不一致的状态

# Bean的实例化

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/BigMap.webp"
alt="BigMap" />
<figcaption aria-hidden="true">BigMap</figcaption>
</figure>

## BeanFactory

![BeanFactory](https://minio.dionysunrsshub.top/myimages/2024-img/BeanFactory.webp)
几个核心接口：
- **AliasRegistry**
bean别名注册和管理
- **BeanDefinitionRegistry**
bean元数据注册和管理
- **SingletonBeanRegistry**
单例bean注册和管理
- **BeanFactory**
bean工厂，提供各种bean的获取及判断方法

通过上述的类依赖图，对于Bean的实例化，核心实现是在**DefaultListableBeanFactory**

### DefaultListableBeanFactory - AbstractBeanFactory

bean的实例化过程发生在**getBean**调用阶段（对于singleton则发生在首次调用阶段），***getBean***的实现方法众多，我们追根溯源，找到最通用的方法`AbstractBeanFactory#doGetBean`

#### `doGetBean`

``` java
// org.springframework.beans.factory.support.AbstractBeanFactory
protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
        @Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

    // 1. 获取真正的beanName
    final String beanName = transformedBeanName(name);
    Object bean;

    // 2. 尝试获取(提前曝光的)singleton bean实例（为了解决循环依赖）
    Object sharedInstance = getSingleton(beanName);
    
    // 3. 如果存在
    if (sharedInstance != null && args == null) { ... }
    
    // 4. 如果不存在
    else { ... }
    
    // 5. 尝试类型转换
    if (requiredType != null && !requiredType.isInstance(bean)) { ... }
    
    return (T) bean;
}
```

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/doGetBean.webp"
alt="doGetBean" />
<figcaption aria-hidden="true">doGetBean</figcaption>
</figure>

##### Bean Name的转换

在使用bean-name获取bean的时候，除了可以使用原始bean-name之外，还可以使用alias别名等，bean-name的转换则是将传入的'bean-name'一层层转为最原始的bean-name

- 函数**canonicalName**的作用则是利用别名注册***aliasMap***，将别名alias转为原始bean-name
- 函数**transformedBeanName**比较特殊，其是将**FactoryBean**的bean-name前缀 '&' 去除
  ![bean-name-transform](https://minio.dionysunrsshub.top/myimages/2024-img/bean-name-transform.webp)
  
##### 尝试获取单例

拿到原始的bean-name之后，便可以实例化bean或者直接获取已经实例化的singleton-bean

在获取singleton-bean的时候一般存在三种情况：1. 还未实例化(或者不是单例)；2. 已经实例化；3. 正在实例化；
- 对于 "1. 还未实例化" ，返回null即可，后续进行实例化动作
- 对于 "2. 已经实例化"，直接返回实例化的singleton-bean
- 对于 "3. 正在实例化"，会存在循环依赖问题

Spring中对于singleton-bean，有一个***sharedInstance***的概念，在调用`getSingleton`函数时，返回的不一定是完全实例化的singleton-bean，有可能是一个中间状态（创建完成，但未进行属性依赖注入及其他后处理逻辑），这种中间状态会通过**getSingleton**函数提前曝光出来，目的是为了解决循环依赖

因此，Spring通过提供三层缓存来解决循环依赖问题，并且可以通过这种机制实现诸多的**PostProcessor**增强Bean，例如AOP
+ **singletonObjects**
缓存已经实例化完成的singleton-bean

- **earlySingletonObjects**
  缓存正在实例化的、提前曝光的singleton-bean，用于处理循环依赖

- **singletonFactories**
  缓存用于生成earlySingletonObject的 ObjectFactory

> **ObjectFactory**，定义了一个用于创建、生成对象实例的工厂方法

``` java
@FunctionalInterface
public interface ObjectFactory<T> {
    T getObject() throws BeansException;
}
```

因此getSingleton的逻辑如下：
![getSingleton](https://minio.dionysunrsshub.top/myimages/2024-img/getSingleton.webp)

**NOTE**: 在[提前暴露实体](#提前暴露实体 "wikilink")中，将相应的**ObjectFactory**放入了**singletonFactories**

##### FactoryBean的处理(sharedInstance存在的逻辑)

==**sharedInstance**不一定是我们所需要的bean实例==

例如，我们在定义Bean的时候可以通过实现**FactoryBean**接口来定制bean实例化的逻辑([实现FactoryBean](JavaSSM#^249d21 "wikilink"))，通过注册FactoryBean类型的Bean，实例化后的原始实例类型同样为FactoryBean，但我们需要的是通过FactoryBean#getObject方法得到的实例，这需要针对FactoryBean做一些处理，即**AbstractBeanFactory#getObjectForBeanInstance**

> Get the object for the given bean instance, either the bean instance itself or its created object in case of a FactoryBean.
> Now we have the bean instance, which may be a normal bean or a FactoryBean. If it's a FactoryBean, we use it to create a bean instance.

该函数要实现的逻辑比较简单，如果sharedInstance是 FactoryBean，则使用getObject方法创建真正的实例

> getObjectForBeanInstance是一个通用函数，并不只针对通过getSingleton得到的sharedInstance，任何通过缓存或者创建得到的 rawInstance，都需要经过getObjectForBeanInstance处理，拿到真正需要的 beanInstance

``` java
/**
 * @param beanInstance  sharedInstance / rawInstance，可能为FactoryBean
 * @param name            传入的未做转换的 bean name
 * @param beanName        对name做过转换后的原始 canonical bean name
 * @param mbd            合并后的RootBeanDefinition，下文会介绍
 */
protected Object getObjectForBeanInstance(
    Object beanInstance, String name, String beanName, RootBeanDefinition mbd)
```

###### getObjectBeanInstance

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/getObjectForBeanInstance.webp"
alt="getObjectForBeanInstance" />
<figcaption aria-hidden="true">getObjectForBeanInstance</figcaption>
</figure>

在这个判断逻辑中，如果入参**name**以'&'开头则直接返回，这里兼容了一种情况，如果需要获取/注入FactoryBean而不是getObject生成的实例，[则需要在bean-name/alias-name前加入\'&\'](JavaSSM#^5d8395 "wikilink")

对于singleton，FactoryBean#getObject的结果会被缓存到factoryBeanObjectCache，对于缓存中不存在或者不是singleton的情况，会通过FactoryBean#getObject生成 \^factorybeangetobject

###### `FactoryBeanRegistrySupport#getObjectFromFactoryBean`

``` java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {  
    if (factory.isSingleton() && this.containsSingleton(beanName)) {  
        synchronized(this.getSingletonMutex()) {  
            Object object = this.factoryBeanObjectCache.get(beanName);  
            if (object == null) {  
                object = this.doGetObjectFromFactoryBean(factory, beanName);  
                Object alreadyThere = this.factoryBeanObjectCache.get(beanName);  
                if (alreadyThere != null) {  
                    object = alreadyThere;  
                } else {  
                    if (shouldPostProcess) {  
                        if (this.isSingletonCurrentlyInCreation(beanName)) {  
                            return object;  
                        }  
  
                        this.beforeSingletonCreation(beanName);  
  
                        try {  
                            object = this.postProcessObjectFromFactoryBean(object, beanName);  
                        } catch (Throwable var14) {  
                            throw new BeanCreationException(beanName, "Post-processing of FactoryBean's singleton object failed", var14);  
                        } finally {  
                            this.afterSingletonCreation(beanName);  
                        }  
                    }  
  
                    if (this.containsSingleton(beanName)) {  
                        this.factoryBeanObjectCache.put(beanName, object);  
                    }  
                }  
            }  
  
            return object;  
        }  
    } else {  
        Object object = this.doGetObjectFromFactoryBean(factory, beanName);  
        if (shouldPostProcess) {  
            try {  
                object = this.postProcessObjectFromFactoryBean(object, beanName);  
            } catch (Throwable var17) {  
                throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", var17);  
            }  
        }  
  
        return object;  
    }  
}
```

对于Singleton:
+ 首先从缓存中尝试获取，如获取失败，调用[doGetObjectFromFactoryBean](#`FactoryBeanRegistrySupport doGetObjectFromFactoryBean` "wikilink")，其中内核是调用[FactoryBean#getObject()](#^factorybeangetobject "wikilink")方法
+ 对于需要后处理的Bean，首先判断是否处于正在创建状态(`isSingletonCurrentlyInCreation`)，并且通过`this.beforeSingletonCreate()` `this.afterSingletonCreation()`将实际的`BeanPostProcessor`过程保护
+ 对于`BeanPostProcessor`，调用`this.postProcessObjectFromFactoryBean`，其具体实现在[AbstractAutowireCapableBeanFactory#applyBeanPostProcessorAfterInitialization](#`AbstractAutowireCapableBeanFactory applyBeanPostProcessorAfterInitialization` "wikilink")

###### `FactoryBeanRegistrySupport#doGetObjectFromFactoryBean`

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/FactoryBeanRegistrySupport_doGetObjectFromFactoryBean.webp"
alt="FactoryBeanRegistrySupport_doGetObjectFromFactoryBean" />
<figcaption
aria-hidden="true">FactoryBeanRegistrySupport_doGetObjectFromFactoryBean</figcaption>
</figure>

###### `AbstractAutowireCapableBeanFactory#applyBeanPostProcessorAfterInitialization`

postProcessAfterInitialization函数可以对现有bean instance做进一步的处理，甚至可以返回新的bean instance，这就为bean的增强提供了一个非常方便的扩展方式

##### 加载Bean实例 (sharedInstance不存在的逻辑)

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/createBeanInstance.webp"
alt="createBeanInstance" />
<figcaption aria-hidden="true">createBeanInstance</figcaption>
</figure>

Bean的加载/创建分为三大部分
1. 将BeanDefinition合并为RootBeanDefinition，类似类继承，子BeanDefinition属性会覆盖父BeanDefinition
2. 依次加载所依赖的bean，对于有依赖的情况，优先递归加载依赖的bean
3. 按照不同的bean类型，根据BeanDefinition的定义进行加载/创建

###### BeanDefinition合并 (RootBeanDefinition)

在`AbstractBeanFactory#getMergedLocalBeanDefinition`中执行核心逻辑

###### 加载dependes-On beans

``` java
String[] dependsOn = mbd.getDependsOn();
if (dependsOn != null) {
    // 遍历所有的依赖
    for (String dep : dependsOn) {
        // 检测循环依赖
        if (isDependent(beanName, dep)) { /* throw exception */ }
        // 注册依赖关系
        registerDependentBean(dep, beanName);
        // 递归getBean，加载依赖bean
        try { getBean(dep); }
        catch (NoSuchBeanDefinitionException ex) { /* throw exception */ }
    }
}
```

该过程中涉及两个中间态
+ dependentBeanMap
存储哪些bean依赖了我（哪些bean里注入了我）
如果 beanB -\> beanA, beanC -\> beanA，key为beanA，value为\[beanB, beanC\]

- dependenciesForBeanMap
  存储我依赖了哪些bean（我注入了哪些bean）
  如果 beanA -\> beanB, beanA -\> beanC，key为beanA，value为\[beanB, beanC\]

###### 加载singleton bean实例

``` java
if (mbd.isSingleton()) {
    sharedInstance = getSingleton(beanName, () -> {
        // singletonFactory - ObjectFactory
        try { return createBean(beanName, mbd, args); }
        catch (BeansException ex) {    destroySingleton(beanName);    throw ex; }
    });
    bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
}
```

其中核心为**createBean**与**getObjectForBeanInstance**

- **createBean**
  根据BeanDefinition的内容，创建/初始化 bean instance
- **[#getObjectBeanInstance](#getObjectBeanInstance "wikilink")**
  主要处理FactoryBean

**createBean**被包装在lambda(singletonFactory)，重写`ObjectFactory#getObject()`，作为[getSingleton](#`DefaultSingletonBeanRegistry getSingleton(String, ObjectFactory)` "wikilink")的参数

###### `DefaultSingletonBeanRegistry#getSingleton(String, ObjectFactory)`

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/createSingletonBean.webp"
alt="createSingletonBean" />
<figcaption aria-hidden="true">createSingletonBean</figcaption>
</figure>

同样的，会先在缓存中查找该singleton，如果不存在，创建的核心逻辑在于[createBean](#AbstractAutowireCapableBeanFactory createBean "wikilink")

###### `AbstractAutowireCapableBeanFactory#createBean`

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/createBean.webp"
alt="createBean" />
<figcaption aria-hidden="true">createBean</figcaption>
</figure>

1.  **resolveBeanClass**
    这一步骤用于锁定bean class，在没有显示指定beanClass的情况下，使用className加载beanClass
2.  **验证method overrides**
    ==在BeanDefinitionReader 中有提到过lookup-method及replace-method，该步骤是为了确认以上两种配置中的method是否存在==
3.  **执行InstantiationAwareBeanPostProcessor前处理器**(**postProcessBeforeInstantiation**)
    如果这个步骤中生成了"代理"bean instance，则会有一个短路操作，`直接返回`该bean instance而不再执行doCreate，其中的核心逻辑为调用`this.applyBeanPostProcessorsBeforeInstantiation()` \^fcb215

``` java
try {
    // Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
    Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    if (bean != null) {
          // 如果这里生成了代理的bean instance会直接返回
        return bean;
    }
} cache (Throwable ex) { // throw exception }

try {
  // 创建bean instance
  Object beanInstance = doCreateBean(beanName, mbdToUse, args);
  // ...
}
```

4.  **[doCreateBean](#`doCreateBean` "wikilink")** (AbstractAutowireCapableBeanFactory)
    真正bean的创建及初始化过程在此处实现

###### `doCreateBean`

``` java
protected Object doCreateBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) throws BeanCreationException {  
    BeanWrapper instanceWrapper = null;  
    if (mbd.isSingleton()) {  
        instanceWrapper = (BeanWrapper)this.factoryBeanInstanceCache.remove(beanName);  
    }  
  
    if (instanceWrapper == null) {  
        instanceWrapper = this.createBeanInstance(beanName, mbd, args);  
    }  
  
    Object bean = instanceWrapper.getWrappedInstance();  
    Class<?> beanType = instanceWrapper.getWrappedClass();  
    if (beanType != NullBean.class) {  
        mbd.resolvedTargetType = beanType;  
    }  
  
    synchronized(mbd.postProcessingLock) {  
        if (!mbd.postProcessed) {  
            try {  
                this.applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);  
            } catch (Throwable var17) {  
                throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Post-processing of merged bean definition failed", var17);  
            }  
  
            mbd.markAsPostProcessed();  
        }  
    }  
  
    boolean earlySingletonExposure = mbd.isSingleton() && this.allowCircularReferences && this.isSingletonCurrentlyInCreation(beanName);  
    if (earlySingletonExposure) {  
        if (this.logger.isTraceEnabled()) {  
            this.logger.trace("Eagerly caching bean '" + beanName + "' to allow for resolving potential circular references");  
        }  
  
        this.addSingletonFactory(beanName, () -> {  
            return this.getEarlyBeanReference(beanName, mbd, bean);  
        });  
    }  
  
    Object exposedObject = bean;  
  
    try {  
        this.populateBean(beanName, mbd, instanceWrapper);  
        exposedObject = this.initializeBean(beanName, exposedObject, mbd);  
    } catch (Throwable var18) {  
        if (var18 instanceof BeanCreationException bce) {  
            if (beanName.equals(bce.getBeanName())) {  
                throw bce;  
            }  
        }  
  
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, var18.getMessage(), var18);  
    }  
  
    if (earlySingletonExposure) {  
        Object earlySingletonReference = this.getSingleton(beanName, false);  
        if (earlySingletonReference != null) {  
            if (exposedObject == bean) {  
                exposedObject = earlySingletonReference;  
            } else if (!this.allowRawInjectionDespiteWrapping && this.hasDependentBean(beanName)) {  
                String[] dependentBeans = this.getDependentBeans(beanName);  
                Set<String> actualDependentBeans = new LinkedHashSet(dependentBeans.length);  
                String[] var12 = dependentBeans;  
                int var13 = dependentBeans.length;  
  
                for(int var14 = 0; var14 < var13; ++var14) {  
                    String dependentBean = var12[var14];  
                    if (!this.removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {  
                        actualDependentBeans.add(dependentBean);  
                    }  
                }  
  
                if (!actualDependentBeans.isEmpty()) {  
                    throw new BeanCurrentlyInCreationException(beanName, "Bean with name '" + beanName + "' has been injected into other beans [" + StringUtils.collectionToCommaDelimitedString(actualDependentBeans) + "] in its raw version as part of a circular reference, but has eventually been wrapped. This means that said other beans do not use the final version of the bean. This is often the result of over-eager type matching - consider using 'getBeanNamesForType' with the 'allowEagerInit' flag turned off, for example.");  
                }  
            }  
        }  
    }  
  
    try {  
        this.registerDisposableBeanIfNecessary(beanName, bean, mbd);  
        return exposedObject;  
    } catch (BeanDefinitionValidationException var16) {  
        throw new BeanCreationException(mbd.getResourceDescription(), beanName, "Invalid destruction signature", var16);  
    }  
}
```

可以将该流程细分为如下：
1. [创建Bean实体](#创建Bean实体 `AbstractAutowireCapableBeanFactory createBeanInstance` "wikilink")
2. [BeanDefinition后处理](#BeanDefinition后处理 - `AbstractAutowireCapableBeanFactory applyMergedBeanDefinitionPostProcessors` "wikilink")
3. [提前暴露实体](#提前暴露实体 "wikilink")
4. [属性注入](#属性注入 - `AbstractAutowireCapableBeanFactory populateBean` "wikilink")
5. [初始化](#初始化 - `AbstractAutowireCapableBeanFactory initializeBean` "wikilink")
6. [注册Disposable](#注册Disposable - `AbstractBeanFactory registerDisposableBeanIfNecessary` "wikilink")

###### 创建Bean实体 - `AbstractAutowireCapableBeanFactory#createBeanInstance`

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/createBeanInstance_2.webp"
alt="createBeanInstance_2" />
<figcaption aria-hidden="true">createBeanInstance_2</figcaption>
</figure>

1.  instanceSupplier
    从上面的流程图可以看出，创建bean实体不一定会使用到构造函数，可以使用Supplier的方式
2.  factory method
    [工厂模式](JavaSSM#^249d21 "wikilink")
    @Configuration + @Bean的实现方式就是factory-bean + factory-method
    [对应的参数获取](#`ConstructorResolver resolvePreparedArguments` "wikilink")
3.  有参构造函数
    **AbstractAutowireCapableBeanFactory#autowireConstructor** -\> **[ConstructorResolver#autowireConstructor](#`**ConstructorResolver autowireConstructor**` "wikilink")**
4.  无参构造函数
    与有参构造创建过程一致，除了不需要参数的依赖注入，使用默认无参构造函数进行实例化

###### `ConstructorResolver#resolvePreparedArguments`

使用指定（类）bean的（静态）方法创建bean实体的逻辑在ConstructorResolver#instantiate(String, RootBeanDefinition, Object, Method, args)，而真正的逻辑在SimpleInstantiationStrategy#instantiate(RootBeanDefinition, String, BeanFactory, Object, Method, Object...)，其核心的执行逻辑非常简单，有了方法factoryMethod(factoryBean)及入参args，便可以调用该方法创建bean实体

``` java
Object result = factoryMethod.invoke(factoryBean, args);
```

factoryBean可以通过beanFactory.getBean获取到（正是当前在讲的逻辑），factoryMethod可以通过反射获取到，而入参args就从`ConstructorResolver#resolvePreparedArguments`中获取，即是Spring中**依赖注入**的核心实现

该函数的作用是将BeanDefinition中定义的入参转换为需要的参数(==将BeanDefinitionReader中封装的对象转换==)

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/resolvePreparedArguments.webp"
alt="resolvePreparedArguments" />
<figcaption aria-hidden="true">resolvePreparedArguments</figcaption>
</figure>

**[More in blogs](https://segmentfault.com/a/1190000022309143#item-3-4)**

###### `ConstructorResolver#autowireConstructor`

同样的，调用**ConstructorResolver#resolvePreparedArguments**进行参数的解析和转换(参数的依赖注入)，然后调用 **[ConstructorResolver#instantiate](#`ConstructorResolver instantiate` "wikilink")** 来创建Bean实例

###### `ConstructorResolver#instantiate`

内部并没有统一利用反射技术直接使用构造函数创建，而是通过`InstantiationStrategy.instantiate`进行创建

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/Instantiate.webp"
alt="Instantiate" />
<figcaption aria-hidden="true">Instantiate</figcaption>
</figure>

- 没有设置override-method时，直接使用构造函数创建
- 设置了override-method时，使用cglib技术构造代理类，并代理override方法

Spring默认的实例化策略为**CglibSubclassingInstantiationStrategy**

###### BeanDefinition后处理 - `AbstractAutowireCapableBeanFactory#applyMergedBeanDefinitionPostProcessors`

在属性注入之前提供一次机会来对BeanDefinition进行处理，内部执行所有注册**MergedBeanDefinitionPostProcessor**的**postProcessMergedBeanDefinition**方法

> \[!hint\] **MergedBeanDefinitionPostProcessor**
> `MergedBeanDefinitionPostProcessor` 是一个特定类型的 `BeanPostProcessor`。`MergedBeanDefinitionPostProcessor` 的 `postProcessMergedBeanDefinition` 方法允许在实例化bean之后但在设置bean属性之前，对bean的定义（`BeanDefinition`）进行后处理。这个阶段是用于修改或增强bean定义的，例如，可以解析注解并相应地修改 `BeanDefinition` 的属性。

对于`MergedBeanDefinitionPostProcessor`的实现类`AutowiredAnnotationBeanPostProcessor`，其内部方法AutowiredAnnotationBeanPostProcessor#buildAutowiringMetadata 实现了两个注解类的解析 @Value 及 @Autowired ，找到注解修饰的Filed或者Method并缓存，具体逻辑在[属性注入](#属性注入 - `AbstractAutowireCapableBeanFactory populateBean` "wikilink") \^autowiredAnnotationBeanPostProcessor1

###### 提前暴露实体

通过将**AbstractAutowireCapableBeanFactory#getEarlyBeanReference**封装为ObjectFactory，调用**DefaultSingletonBeanRegistry#addSingletonFactory**，将该ObjectFactory缓存在**DefaultSingletonBeanRegistry.singletonFactories**中，在`getBean`逻辑中的`getSingleton`会执行ObjectFactory将singleton提前暴露
==此处即为何时添加ObjectFactory进入singletonFactories中，解决循环依赖==

> 此时暴露的singleton-bean仅完成了bean的实例化，属性注入、初始化等逻辑均暂未执行

###### 属性注入 - `AbstractAutowireCapableBeanFactory#populateBean`

在[创建Bean实体](#创建Bean实体 - `AbstractAutowireCapableBeanFactory createBeanInstance` "wikilink")中介绍了factory method方式及有参构造函数方式的参数注入逻辑，除此之外还有一种注入便是属性注入

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/populateBean.webp"
alt="populateBean" />
<figcaption aria-hidden="true">populateBean</figcaption>
</figure>

流程中出现了两次**InstantiationAwareBeanPostProcessor**，在第一次出现中调用的`postProcessorAfterInstantiation`也与前面的**InstantiationAwareBeanPostProcessor.postProcessBeforeInstantiation**相同，拥有短路操作：如果该步骤生成了"代理"bean instance，`直接返回`该bean instance而不再执行后续的doCreate；如果有任意一个InstantiationAwareBeanPostProcessor的postProcessAfterInstantiation方法返回false，则会跳出属性注入的逻辑，官方对此的解释如下

> Give any InstantiationAwareBeanPostProcessors the opportunity to modify the state of the bean before properties are set. This can be used, for example, to support styles of field injection.

**autowireByName**及**autowireByType**方法作为"候补"补充BeanDefinition的propertyValues

**PropertyValue**中记录了需要注入的属性信息及需要注入的属性值，那BeanDefinition的propertyValues都来自哪里？xml中的bean配置、自定义的BeanDefinition等

通过注解修饰的属性(方法)通过**InstantiationAwareBeanPostProcessor#postProcessProperties**进行注入 -\> ==AutowiredAnnotationBeanPostProcessor#postProcessProperties & CommonAnnotationBeanPostProcessor#postProcessProperties==

最后，通过AbstractAutowireCapableBeanFactory#applyPropertyValues 将**PropertyValue**中记录的需要注入的属性，已经依赖的类型（String、RuntimeBeanReference、等），根据不同的类型解析依赖的bean并设置到对应的属性上（==此过程与DefaultListableBeanFactory#doResolveDependency相似==）

###### 初始化 - `AbstractAutowireCapableBeanFactory#initializeBean`

以上，完成了bean实例的创建和属性注入，之后还有一些初始化的方法，比如各种**Aware**的**setXxx**是如何调用的、@PostConstruct是怎么调用的？

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/initializeBean.webp"
alt="initializeBean" />
<figcaption aria-hidden="true">initializeBean</figcaption>
</figure>

###### 注册Disposable - `AbstractBeanFactory#registerDisposableBeanIfNecessary`

至此，终于完成了bean实例的创建、属性注入以及之后的初始化，此后便可以开始使用了

在使用Spring的过程中经常还会碰到设置销毁逻辑的情况，如数据库连接池、线程池等等，在Spring销毁bean的时候还需要做一些处理，类似于C++中的析构

在bean的创建逻辑中，最后一个步骤则是注册bean的销毁逻辑（DisposableBean）

销毁逻辑的注册有几个条件

1.  非prototype（singleton或者注册的scope）
2.  非NullBean
3.  指定了destroy-method（如xml中指定或者BeanDefinition中直接设置）或者存在**@PreDestroy** 注解的方法（**CommonAnnotationBeanPostProcessor.requiresDestruction**）

``` java
if (!mbd.isPrototype() && requiresDestruction(bean, mbd))
```

满足以上条件的bean会被封装为**DisposableBeanAdapter**，并注册在**DefaultSingletonBeanRegistry.disposableBeans**中
###### 加载prototype bean实例

``` java
else if (mbd.isPrototype()) {
    Object prototypeInstance = null;
    try {
        beforePrototypeCreation(beanName);
        prototypeInstance = createBean(beanName, mbd, args);
    }
    finally { afterPrototypeCreation(beanName);    }
    bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
}
```

prototype bean的创建与singleton bean类似，只是不会缓存创建完成的bean

###### 加载其他scope bean实例

scope，即作用域，或者可以理解为生命周期

上文介绍了singleton-bean及prototype-bean的创建过程，严格意义上讲以上两种都是一种特殊的scope-bean，分别对应ConfigurableBeanFactory#SCOPE_SINGLETON及ConfigurableBeanFactory#SCOPE_PROTOTYPE，前者作用域为整个IOC容器，也可理解为单例，后者作用域为所注入的bean，每次注入(每次触发getBean)都会重新生成

Spring中还提供很多其他的scope，如WebApplicationContext#SCOPE_REQUEST或WebApplicationContext#SCOPE_SESSION，前者作用域为一次web request，后者作用域为一个web session周期

自定义scope的bean实例创建过程与singleton bean的创建过程十分相似，需要实现Scope的get方法(org.springframework.beans.factory.config.Scope#get)

``` java
else {
    String scopeName = mbd.getScope();
    final Scope scope = this.scopes.get(scopeName);
    if (scope == null) { /* throw exception */ }
    try {
        Object scopedInstance = scope.get(beanName, () -> {
            beforePrototypeCreation(beanName);
            // createBean被封装在Scope#get函数的lambda参数ObjectFactory中
            try { return createBean(beanName, mbd, args); }
            finally { afterPrototypeCreation(beanName); }
        });
        bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
    } catch (IllegalStateException ex) { /* throw exception */}
}
```

Scope接口除了get方法之外，还有一个remove方法，前者用于定义bean的初始化逻辑，后者用于定义bean的销毁逻辑

``` java
public interface Scope {
  /**
   * Return the object with the given name from the underlying scope
   */
  Object get(String name, ObjectFactory<?> objectFactory);
  
   /**
   * Remove the object with the given name from the underlying scope.
   */
  Object remove(String name);
}
```

**WebApplicationContext#SCOPE_SESSION**对应的Scope实现见**org.springframework.web.context.request.SessionScope**

**WebApplicationContext#SCOPE_REQUEST**对应的Scope实现见**org.springframework.web.context.request.RequestScope**

以上两种Scope实现都较为简单，前者将初始化的bean存储在request attribute中，后者将初始化的bean存储在http session中

##### 尝试类型转换

以上，完成了bean的创建、属性的注入、dispose逻辑的注册，但获得的bean类型与实际需要的类型可能依然不相符，在最终交付bean之前（getBean）还需要进行一次类型转换，使用**PropertyEditor**进行类型转换，将bean转换为真正需要的类型后，便完成了整个getBean的使命

#### Bean销毁过程

bean的创建过程始于**DefaultListableBeanFactory#getBean**，销毁过程则终于ConfigurableApplicationContext#close，跟踪下去，具体的逻辑在**DefaultSingletonBeanRegistry#destroySingletons**

1.  **DefaultSingletonBeanRegistry.disposableBeans**
    需要注册销毁逻辑的bean会被封装为**DisposableBeanAdapter**并缓存在此处
2.  **DefaultSingletonBeanRegistry.dependentBeanMap**
    对于存在依赖注入关系的bean，会将bean的依赖关系缓存在此处（dependentBeanMap: 哪些bean依赖了我; dependenciesForBeanMap: 我依赖了哪些bean）

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/destory.webp"
alt="destory" />
<figcaption aria-hidden="true">destory</figcaption>
</figure>

从上图中可以看出，bean的销毁顺序与创建顺序正好相反，如果有 beanA --dependsOn--\> beanB --\> beanC ，创建（getBean）时一定是beanC -\> beanB -\> beanA，销毁时一定是 beanA -\> beanB -\> beanC，以此避免因为依赖关系造成的一些异常情况

#### 循环依赖

**earlySingletonObject**是用来解决循环依赖的问题，具体时机是在实例化完后属性注入之前，会提前将当前的bean实体暴露出来，以防止在属性注入过程中所注入的bean又依赖当前的bean造成的类似"死锁"的状态

但是存在以下情况，Spring依旧会陷入循环依赖死锁：
+ 显式设置dependsOn的循环依赖

``` java
@DependsOn("beanB")
@Component
public class BeanA {}

@DependsOn("beanC")
@Component
public class BeanB {}

@DependsOn("beanA")
@Component
public class BeanC {}
```

- 构造函数循环依赖

``` java
@Component
public class BeanA {
    public BeanA(BeanB beanB) {
    }
}

@Component
public class BeanB {
    public BeanB(BeanC beanC) {
    }
}

@Component
public class BeanC {
    public BeanC(BeanA beanA) {
    }
}
```

- factory-method循环依赖

``` java
@Bean
public BeanA beanA(BeanB beanB) {
    return new BeanA();
}

@Bean
public BeanB beanB(BeanC beanC) {
    return new BeanB();
}

@Bean
public BeanC beanC(BeanA beanA) {
    return new BeanC();
}
```

- 上述三种依赖混合

只要一个循环依赖中的所有bean，其依赖关系都需要在创建bean实例之前进行解决，此循环依赖则一定无解

要打破无解的循环依赖，在构成循环依赖的一个环中，只需要保证其中至少一个Bean的依赖在该Bean创建且暴露**earlySingleton**之后处理即可，即在属性注入阶段进行属性依赖的处理

``` java
@Component
public class BeanA {
    @Autowired
    private BeanB beanB;
}

@Component
public class BeanB {
    public BeanB(BeanC beanC) {
    }
}

@Bean
public BeanC beanC(BeanA beanA) {
    return new BeanC();
}
```

以"bean创建且暴露earlySingleton"为节点，在此之前处理依赖的有`instance supplier parameter`、`factory method parameter`、`constructor parameter`、等，在此之后处理的依赖有 `class property`、`setter parameter`等

# ApplicationContext

**BeanFactory**实现了IoC的基础能力，而**ApplicationContext**是**BeanFactory**的子类，除了继承IoC的基础能力外
+ 支持国际化 (MessageSource)
+ 支持资源访问 (ResourcePatternResolver)
+ 事件机制 (ApplicationEventPublisher)
+ 默认初始化所有Singleton
+ 提供扩展能力

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/ApplicationContext.webp"
alt="ApplicationContext" />
<figcaption aria-hidden="true">ApplicationContext</figcaption>
</figure>

无论何种功能的ApplicationContext，在做完基本的初始化后均会调用**AbstractApplicationContext#Refresh**

## `AbstractApplicationContext#Refresh`

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/refresh.webp"
alt="refresh" />
<figcaption aria-hidden="true">refresh</figcaption>
</figure>

### 准备上下文 - `AbstractApplicationContext#prepareRefresh`

该部分主要实现对上下文的准备工作，其主要涉及到两个接口**AbstractApplicationContext#initPropertySources**及**ConfigurablePropertyResolver#validateRequiredProperties**，前者由子类实现，用于初始化PropertySource；后者用于对必要属性进行验证

``` java
public class MyClasspathXmlApplicationContext extends ClassPathXmlApplicationContext {
    @Override
    protected void initPropertySources() {
        super.initPropertySources();
        getEnvironment().setRequiredProperties("runtimeEnv");
    }
}
```

重写**initPropertySources**方法，并添加runtimeEnv为必须的环境变量属性，如此在系统启动的时候便会进行检测，对于不存在任何一个必要环境变量的情况均会抛出异常终止启动

### 加载BeanFactory - `AbstractApplicationContext#obtainFreshBeanFactory`

该函数内部实现比较简单，重点在**refreshBeanFactory**，该函数同样由子类实现

对于**AbstractRefreshableApplicationContext**，refreshBeanFactory基本步骤为
1. 创建BeanFactory (**DefaultListableBeanFactory**)
2. 设置BeanFactory
3. **[加载BeanDefinition](#Bean的注册 "wikilink")**

在第3步中，AbstractXmlApplicationContext的实现则是对xml配置文件的解析及加载；AnnotationConfigWebApplicationContext的实现则是对class文件的扫描并加载，以及其他基于AbstractRefreshableApplicationContext的ApplicationContext实现

对于**GenericApplicationContext**，BeanFactory的创建及BeanDefinition的加载在**refresh**调用之前早已完成，refreshBeanFactory的实现则是对BeanFactory加载状态的简单校验

#### AbstractRefreshableApplicationContext & GenericApplicationContext

##### AbstractRefreshableApplicationContext

对于继承自 `AbstractRefreshableApplicationContext` 的上下文，例如 `ClassPathXmlApplicationContext` 或 `AnnotationConfigApplicationContext`，它们通过覆盖 `refreshBeanFactory()` 方法来实现具体的 BeanDefinition 加载逻辑。这些上下文类型专门用于从外部资源（如 XML 文件、Java 配置类等）加载配置信息，并将这些配置信息解析为一组 BeanDefinition，然后注册到内部的 BeanFactory 中。这个过程通常发生在上下文的 `refresh()` 方法调用过程中（我们正在讨论的），这个方法不仅负责加载和注册 BeanDefinition，还包括初始化单例bean、处理别名定义、注册BeanPostProcessor等一系列容器启动时的活动。

> \[!QUOTE\] refresh()关键步骤 \^configurerRelated
> 1. **创建 BeanFactory**：`AbstractRefreshableApplicationContext` 首先会创建一个新的 `BeanFactory` 实例，这通常是一个 `DefaultListableBeanFactory` 实例。这个 `BeanFactory` 实现了 `BeanDefinitionRegistry` 接口，使得它能够注册 BeanDefinition。
> 2. ==**加载 BeanDefinition**：接着，上下文会调用特定的方法（例如，对于基于 XML 的配置，会使用 `XmlBeanDefinitionReader`；对于基于注解的配置，会使用 `AnnotatedBeanDefinitionReader` 和 `ClassPathBeanDefinitionScanner`）来加载 BeanDefinition。这些 Reader 和 Scanner 实现了 `BeanDefinitionRegistry` 接口的 `registerBeanDefinition` 方法来实际完成注册工作。==
> 3. **刷新 BeanFactory**：加载完所有 BeanDefinition 后，`AbstractRefreshableApplicationContext` 会对 `BeanFactory` 进行刷新，这涉及到预实例化单例、注册 `BeanPostProcessor`、初始化剩余的非懒加载单例等一系列操作。
> 4. **发布事件**：在整个容器刷新过程中，还会发布各种应用事件，如 `ContextRefreshedEvent`，允许应用中的其他组件对这些事件作出响应。
>
> 通过上述步骤，`AbstractRefreshableApplicationContext` 完成了 BeanDefinition 的加载、注册以及整个 Spring 容器的初始化和刷新工作。在这个过程中，`BeanDefinitionRegistry` 接口扮演了 BeanDefinition 注册的关键角色
> ##### GenericApplicationContext

`GenericApplicationContext` 直接实现了 `BeanDefinitionRegistry` 接口，使得它可以在运行时动态注册 BeanDefinition。与 `AbstractRefreshableApplicationContext` 的子类不同，`GenericApplicationContext` 并不专门依赖于外部资源来加载 BeanDefinition。相反，它提供了一套程序化的接口，允许开发者直接在代码中通过调用 `registerBeanDefinition(String beanName, BeanDefinition beanDefinition)` 方法来注册 BeanDefinition。这种方式使得 `GenericApplicationContext` 非常灵活，适用于那些需要在运行时动态调整 Spring 配置的场景。

##### 关系和区别

- **加载方式的区别**：`AbstractRefreshableApplicationContext` 的子类通常通过解析配置资源（XML、注解等）来加载 BeanDefinition，而 `GenericApplicationContext` 允许以编程方式直接注册 BeanDefinition。
- **使用场景的区别**：`AbstractRefreshableApplicationContext` 的子类适合于静态配置资源的场景，其中配置信息在应用启动时已经确定。`GenericApplicationContext` 更适合于动态配置的场景，比如基于条件的 BeanDefinition 注册或运行时的配置调整。
- **刷新容器的能力**：虽然两者都可以通过 `refresh()` 方法来刷新应用上下文，但 `AbstractRefreshableApplicationContext` 的子类通常在设计时就考虑了完整的容器刷新流程（包括重新加载配置资源），而 `GenericApplicationContext` 刷新主要是为了应用新注册的 BeanDefinition。==前者会重置BeanFactory而后者不会==

### 填充部分扩展 - `AbstractApplicationContext#prepareBeanFactory`

该函数执行以下逻辑
1. 设置BeanFactory的ClassLoader
2. 注册默认**BeanExpressionResolver**，用于依赖注入时SpEL的支持
3. 注册默认**PropertyEditor**，用于依赖注入时对参数的解析转换
4. 注册几个特殊Aware的处理逻辑
5. 注册AspectJ相关的几个处理器，用于AOP的支持
6. 注册几个特殊的BeanDefinition

==2-3 的核心逻辑在于解析依赖的值，`DefaultListableBenFactory#doResolveDependency`==

#### 注册几个特殊Aware的处理逻辑

在Bean实例化、注入依赖之后会对Bean进行[最后的初始化](#初始化 - `AbstractAutowireCapableBeanFactory initializeBean` "wikilink")，调用相应的setter方法分别针对**BeanNameAware**、**BeanClassLoaderAware**、**BeanFactoryAware**进行处理

在该函数中，会注册几个特殊的**BeanPostProcessor**

``` java
beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
```

其实现了**postProcessBeforeInitialization**方法，内部调用**ApplicationContextAwareProcessor#invokeAwareInterfaces**针对另外的几类Aware进行了处理

除此之外，Spring会将上述几类Aware设置为**ignoreDependencyInterface**，这意味着以上几类Bean的注入只能通过Aware的方式而不能通过其他属性依赖注入的方式（属性注入、函数参数注入等）

#### 注册特殊的Bean

在使用Spring时，是否有过直接注入**BeanFactory**亦或是**ResourceLoader**，这些bean正是在这里被Spring注册进去的，除以上外Spring还注入了

- BeanFactory
- ResourceLoader
- ApplicationEventPublisher
- ApplicationContext
- Environment
- *systemProperties* - Environment#.getSystemProperties:Map\<String, Object\>
- systemEnvironment - Environment#.getSystemEnvironment:Map\<String, Object\>
  ### `AbstractApplicationContext#refresh#postProcessBeanFactory()`

对于不同的实现类，注册相应的`BeanPostProcessor`，例如`ServletWebServerApplicationContext`

### 激活BeanFactoryPostProcessor - `AbstractApplicationContext#invokeBeanFactoryPostProcessors`

其内部实现在**PostProcessorRegistrationDelegate#invokeBeanFactoryPostProcessors**

**BeanFactoryPostProcessor**的定义非常简单，其**postProcessBeanFactory**方法允许在bean实例化前对BeanFactory做一些额外的设置

``` java
public interface BeanFactoryPostProcessor {
    /**
     * Modify the application context's internal bean factory after its standard
     * initialization. All bean definitions will have been loaded, but no beans
     * will have been instantiated yet. This allows for overriding or adding
     * properties even to eager-initializing beans.
     */
    void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException;
}
```

核心逻辑如下

<figure>
<img
src="https://minio.dionysunrsshub.top/myimages/2024-img/invokeBeanFactoryPostProcessors.webp"
alt="invokeBeanFactoryPostProcessors" />
<figcaption
aria-hidden="true">invokeBeanFactoryPostProcessors</figcaption>
</figure>

其中涉及两种类型，**BeanDefinitionRegistryPostProcessor**及**BeanFactoryPostProcessor**，前者为后者的子类，**BeanDefinitionRegistryPostProcessors**提供了额外的接口**postProcessBeanDefinitionRegistry**，用于更加方便地**动态**地注册额外的BeanDefinition (`registryProcessor.postProcessBeanDefinitionRegistry(registry)`)，如读取配置文件（json、properties、yml）并解析（或者任何其他的形式），并通过该接口注册相应的BeanDefinition，基于Spring Boot Starter的很多框架均使用该方式进行bean的注册

以上流程图可以看出，优先执行**BeanDefinitionRegistryPostProcessor#postProcessBeanDefinitionRegistry**，再执行**BeanFactoryPostProcessor#postProcessBeanFactory**，各自内部优先执行**PriorityOrdered**实现，再执行**Ordered**实现，最后执行无任何排序的实现

### 注册BeanPostProcessor - `AbstractApplicationContext#registerBeanPostProcessors`

其内部实现在**PostProcessorRegistrationDelegate#registerBeanPostProcessors**
b
#### BeanPostProcessor

``` java
public interface BeanPostProcessor {
    /**
     * Apply this BeanPostProcessor to the given new bean instance before any bean
     * initialization callbacks (like InitializingBean's afterPropertiesSet
     * or a custom init-method). 
     * The returned bean instance may be a wrapper around the original.
     */
    @Nullable
    default Object postProcessBeforeInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }

    /**
     * Apply this BeanPostProcessor to the given new bean instance after any bean
     * initialization callbacks (like InitializingBean's afterPropertiesSet
     * or a custom init-method).
     * The returned bean instance may be a wrapper around the original.
     */
    @Nullable
    default Object postProcessAfterInitialization(Object bean, String beanName) throws BeansException {
        return bean;
    }
}
```

1.  postProcessBeforeInitialization方法在调用bean的init-method之前执行
2.  postProcessAfterInitialization方法在调用bean的init-method之后执行
3.  任何一个方法可对现有bean实例做进一步的修改
4.  任何一个方法可返回新的bean实例，用来替代现有的bean实例

第四点即是AOP生成当前Bean代理的方法

##### InstantiationAwareBeanPostProcessor

该接口继承自**BeanPostProcessor**，其同样有两个方法，一个在创建bean实例之前调用([createBean](#`AbstractAutowireCapableBeanFactory createBean` "wikilink"))，一个在创建bean实例之后、属性注入之前调用([属性注入](#属性注入 - `AbstractAutowireCapableBeanFactory populateBean` "wikilink"))

``` java
public interface InstantiationAwareBeanPostProcessor extends BeanPostProcessor {  
    @Nullable  
    default Object postProcessBeforeInstantiation(Class<?> beanClass, String beanName) throws BeansException {  
        return null;  
    }  
  
    default boolean postProcessAfterInstantiation(Object bean, String beanName) throws BeansException {  
        return true;  
    }  
  
    @Nullable  
    default PropertyValues postProcessProperties(PropertyValues pvs, Object bean, String beanName) throws BeansException {  
        return pvs;  
    }  
}
```

**AbstractApplicationContext#registerBeanPostProcessors**，其内部逻辑与BeanFactoryPostProcessor的注册逻辑类似：
1. 找到所有BeanPostProcessor并实例化
2. 按照实现的Ordered接口分别放入priorityOrderedPostProcessors、orderedPostProcessors、nonOrderedPostProcessors并各自排序
3. 如果实现了MergedBeanDefinitionPostProcessor则放入internalPostProcessors并排序
4. 按顺序依次注册priorityOrderedPostProcessors、orderedPostProcessors、nonOrderedPostProcessors
5. 最后注册internalPostProcessors

**MergedBeanDefinitionPostProcessor**其有一个接口**postProcessMergedBeanDefinition**，在bean实例化完成后属性注入之前被调用，可以用来对当前的BeanDefinition做进一步的修改，如增加PropertyValue等，实现特殊的属性依赖注入，参考[BeanDefinition后处理](#BeanDefinition后处理 - `AbstractAutowireCapableBeanFactory applyMergedBeanDefinitionPostProcessors` "wikilink")与[属性注入](#属性注入 - `AbstractAutowireCapableBeanFactory populateBean` "wikilink")

### 初始化MessageSource - `AbstractApplicationContext#initMessageSource`

Spring的**MessageSource**提供了国际化能力，在开发者未注册MessageSource的情况下Spring会提供一个默认的**DelegatingMessageSource**

### 初始化ApplicationEventMulticaster - `AbstractApplicationContext#initApplicationEventMulticaster`

Spring提供了一套事件（ApplicationEvent）的发布&订阅机制，开发者可自定义事件（继承**ApplicationEvent**），注册事件监听器来订阅消费事件（实现**ApplicationListener** 或使用`@EventListener` 注解），并使用**ApplicationEventPublisher**（直接依赖注入或者使用**ApplicationEventPublisherAware**）发送事件，使用示例可参考[https://www.baeldung.com/spri...](https://link.segmentfault.com/?enc=4cCMPoMwXwyBCW1s98GYow%3D%3D.zKZN5tVMJt9hm6zq2%2B3EFcY86QBjsZcPTtzSUaDvc8amL7BSpuMwZBbobUSICoS%2B)

其实ApplicationContext实现了ApplicationEventPublisher，跟踪其publishEvent方法会发现，最终调用了**AbstractApplicationContext#applicationEventMulticaster.multicastEvent**，开发者可以自行注册一个**ApplicationEventMulticaster**，如果没有Spring会提供一个默认的**SimpleApplicationEventMulticaster**

**SimpleApplicationEventMulticaster#multicastEvent**的逻辑比较简单，会根据事件的类型找到可以处理的所有**ApplicationListener**，依次调用它们的**onApplicationEvent**方法消费事件

``` java
@Override
public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
    ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
    Executor executor = getTaskExecutor();
    for (ApplicationListener<?> listener : getApplicationListeners(event, type)) {
        if (executor != null) {
      // 设置了executor，则异步执行
            executor.execute(() -> invokeListener(listener, event));
        }
        else {
      // 否则同步执行
            invokeListener(listener, event);
        }
    }
}
```

默认情况下会同步、顺序的调用listeners的onApplicationEvent方法，只有设置了executor才会异步调用，不过这样的控制粒度比较粗，要么全部同步消费要么全部异步消费，比较细粒度的控制事件的消费有几种常用方法

1.  使用@Async注解，独立控制某一listener异步消费（[https://www.baeldung.com/spri...](https://link.segmentfault.com/?enc=4rWP3QJ9vlr387BZsONaMg%3D%3D.W43YbA7ycrHpfL3%2FPNVlG%2BEx21WzzhgH1DTfOK4vDavSLj8oIqkTtiWfo2eOrQGz)）
2.  自行编码，将**onApplicationEvent**逻辑放在线程中执行
3.  注册自定义的**ApplicationEventMulticaster**，内部实现自己的同步、异步Event处理逻辑

### 注册ApplicationListener - `AbstractApplicationContext#registerListeners`

这里的逻辑比较简单

1.  在BeanFactory中找到ApplicationListener类型的bean并实例化
2.  调用**ApplicationEventMulticaster#addApplicationListenerBean**方法将ApplicationListeners注册进去

### 初始化所有非Lazy Bean - `AbstractApplicationContext#finishBeanFactoryInitialization`

对于Singleton Bean而言，实例化发生在首次getBean，但你是否有疑惑，我们只是注册了众多Singleton Bean，但在Spring初始化完成后所有的Singleton Bean（Lazy Bean除外）均已经完成实例化

回到**AbstractApplicationContext#finishBeanFactoryInitialization**，该函数会实现几个逻辑

1.  如果自定义了**ConversionService**(另一种注入类型转换的方式)类型bean且bean-name为*conversionService*，则将其注册到BeanFactory中
2.  如果BeanFactory中不存在**EmbeddedValueResolver**（**PropertyResourceConfigurer**会注册一个**PlaceholderResolvingStringValueResolver**到BeanFactory中），则会注册一个默认的**StringValueResolver**用来处理 `${ ... }`类型的值（**Environment#resolvePlaceholders**）
3.  找到所有非Lazy的Singleton BeanDefinition进行实例化（**getBean**）
    1.  如果是FactoryBean，则在bean name前加上'&'，并实例化该FactoryBean，随后实例化真实的bean
    2.  如果不是FactoryBean，则直接实例化该bean
4.  执行**SmartInitializingSingleton**实现类的**afterSingletonsInstantiated**方法

### Refresh的后续动作 - `AbstractApplicationContext#finishRefresh`

除了一些中间状态需要清理外，还有两件比较特殊的地方

#### LifecycleProcessor - `AbstractApplicationContext#initLifecycleProcessor`

Spring提供了**LifecycleProcessor**用于监听BeanFactory的refresh及close，在BeanFactory的各阶段会调用**LifecycleProcessor**的**onFresh**及**onClose**方法

开发者可以自行注册**LifecycleProcessor**类型的bean，bean-name必须为"lifecycleProcessor"，否则Spring会提供一个默认的**DefaultLifecycleProcessor**

之后则会触发**LifecycleProcessor**的**onFresh**方法

> 除此之外，还可以监听**ContextRefreshedEvent**及**ContextClosedEvent**消息

#### refresh事件

在BeanFactory初始化完成后，则会发出**ContextRefreshedEvent**事件

### BeanFactory的销毁 - `AbstractApplicationContext#registerShutdownHook`

该函数用来注册BeanFactory的销毁逻辑

``` java
public void registerShutdownHook() {  
    if (this.shutdownHook == null) {  
        this.shutdownHook = new Thread("SpringContextShutdownHook") {  
            public void run() {  
                synchronized(AbstractApplicationContext.this.startupShutdownMonitor) {  
                    AbstractApplicationContext.this.doClose();  
                }  
            }  
        };  
        Runtime.getRuntime().addShutdownHook(this.shutdownHook);  
    }  
  
}
```

其直接使用了java的**addShutdownHook**函数，在jvm进程**正常**退出的时候触发

**AbstractApplicationContext#doClose**函数定义了BeanFactory具体的销毁过程

1.  发出**ContextClosedEvent**事件
2.  触发**LifecycleProcessor**的**onClose**方法
3.  销毁bean，细节参考[Bean销毁过程](#Bean销毁过程 "wikilink")
4.  由子类实现的**AbstractApplicationContext#closeBeanFactory**及**AbstractApplicationContext#onClose**方法

## ASIDE

- BeanDefinition的加载在[AbstractApplicationContext#obtainFreshBeanFactory](#加载BeanFactory - `AbstractApplicationContext obtainFreshBeanFactory` "wikilink")中实现
- TODO
  - `#{ ... }`类型值的解析由**StandardBeanExpressionResolve**实现
  - `${ ... }`类型值的解析由**PlaceholderResolvingStringValueResolver**实现
  - Spring提供了众多默认的PropertyEditor，若需要自定义PropertyEditor可以通过注册**CustomEditorConfigurer**实现
  - Spring提供了众多Aware，若需要自定义Aware可以通过**BeanPostProcessor**实现
  - BeanFactoryPostProcessor用于在实例化bean之前对BeanFactory做额外的动作
    如，**PropertyResourceConfigurer**用来将**PlaceholderResolvingStringValueResolver**注册到BeanFactory的embeddedValueResolvers中
- [BeanDefinitionRegistryPostProcessor](#激活BeanFactoryPostProcessor - `AbstractApplicationContext invokeBeanFactoryPostProcessors` "wikilink")用于在实例化bean之前（动态）注册额外的BeanDefinition \^fa1ce8
- [BeanPostProcessor](#BeanPostProcessor "wikilink")用于在调用bean的init-method前后，对实例化完成的bean做一些额外的干预
  如，**CommonAnnotationBeanPostProcessor**用来处理@PostConstructor，**AbstractAdvisingBeanPostProcessor**用来实现AOP

# ApplicationContext具体实现类 - AnnotationConfigApplicationContext

``` java
public AnnotationConfigApplicationContext(Class<?>... componentClasses) { 
    this(); //1. 首先会调用自己的无参构造 
    register(componentClasses); //2. 然后注册我们传入的配置类 
    refresh(); //3. 最后进行刷新操作（关键） 
}
```

## 无参构造

``` java
public AnnotationConfigApplicationContext() {
        StartupStep createAnnotatedBeanDefReader = this.getApplicationStartup().start("spring.context.annotated-bean-reader.create");
      //创建AnnotatedBeanDefinitionReader对象，用于后续处理 @Bean 注解
        this.reader = new AnnotatedBeanDefinitionReader(this);
        createAnnotatedBeanDefReader.end();
      //创建ClassPathBeanDefinitionScanner对象，用于扫描类路径上的Bean
        this.scanner = new ClassPathBeanDefinitionScanner(this);
}
```

### AnnotatedBeanDefinitionReader

``` java
public AnnotatedBeanDefinitionReader(BeanDefinitionRegistry registry, Environment environment) {
        Assert.notNull(registry, "BeanDefinitionRegistry must not be null");
        Assert.notNull(environment, "Environment must not be null");
        this.registry = registry;
        this.conditionEvaluator = new ConditionEvaluator(registry, environment, null);
      //这里注册了注解处理配置相关的后置处理器
        AnnotationConfigUtils.registerAnnotationConfigProcessors(this.registry);
}
```

这里会将ConfigurationClassPostProcessor后置处理器加入到BeanFactory中，它继承自BeanFactoryPostProcessor，也就是说一会会在BeanFactory初始化完成之后进行后置处理，同时这里也会注册一个[AutowiredAnnotationBeanPostProcessor](#^autowiredAnnotationBeanPostProcessor1 "wikilink")后置处理器到BeanFactory，它继承自BeanPostProcessor，用于处理后续生成的Bean对象，其实看名字就知道，这玩意就是为了处理@Autowire、@Value这种注解，用于自动注入

## 注册传入的配置类 - `register`

``` java
@Override
public void register(Class<?>... componentClasses) {
        Assert.notEmpty(componentClasses, "At least one component class must be specified");
        StartupStep registerComponentClass = this.getApplicationStartup().start("spring.context.component-classes.register")
                .tag("classes", () -> Arrays.toString(componentClasses));
      //使用我们上面创建的Reader注册配置类
        this.reader.register(componentClasses);
        registerComponentClass.end();
}
```

## [Refresh](#`AbstractApplicationContext Refresh` "wikilink")

# ==TODO==

- ☒ Spring AOP
- ☐ 注解运行逻辑
  - [`@Component`与`@Bean`的区别](https://www.jianshu.com/p/3fbfbb843b63)
  - [JavaSSM#\^473168](JavaSSM#^473168 "wikilink")
  - ☐ @Bean 在处理属性注入时？
- ☒ AnnotationConfigApplicationContext - 与 配置类的关系 - 具体例子
- ☐ BeanDefinitionReader和BeanDefinitionRegistry
- ☐ 完善调用链图
  # 配置类的注册 - `ConfigurationClassPostProcessor`

ConfigurationClassPostProcessor继承自[BeanDefinitionRegistryPostProcessor](#^fa1ce8 "wikilink") -\> BeanFactoryPostProcessor，这个后置处理器是Spring中提供的，这是专门用于处理配置类的后置处理器，其中`ImportBeanDefinitionRegistrar`，还有`ImportSelector`都是靠它来处理

## `ConfigurationClassPostProcessor#postProcessBeanDefinitionRegistry`

内部调用 **processConfigBeanDefinitions(BeanDefinitionRegistry)** 方法

``` java
public void processConfigBeanDefinitions(BeanDefinitionRegistry registry) {
    // 将Spring认为可能是配置类的候选类加入candidates，例如@Configuration、@Component
    // @ComponentScan、@Import，以及通过实现ImportSelector或ImportBeanDefinitionRegistrar间接引入的配置
    List<BeanDefinitionHolder> configCandidates = new ArrayList<>();
    // 直接取出所有已注册Bean的名称
    String[] candidateNames = registry.getBeanDefinitionNames();
    for (String beanName : candidateNames) {
       // 依次拿到对应的Bean定义，然后进行判断
       BeanDefinition beanDef = registry.getBeanDefinition(beanName);
       if (beanDef.getAttribute(ConfigurationClassUtils.CONFIGURATION_CLASS_ATTRIBUTE) != null) {
          ...
       }
       // 检查一个Bean定义是否符合作为配置类的条件，即使它没有直接使用@Configuration注解
       else if (ConfigurationClassUtils.checkConfigurationClassCandidate(beanDef, this.metadataReaderFactory)) {
          configCandidates.add(new BeanDefinitionHolder(beanDef, beanName));
       }
    }
    // 如果一个打了 @Configuration 的类都没发现，直接返回
    if (configCandidates.isEmpty()) {
       return;
    }
    // 对所有的配置类依据 @Order 进行排序
    configCandidates.sort((bd1, bd2) -> {
       int i1 = ConfigurationClassUtils.getOrder(bd1.getBeanDefinition());
       int i2 = ConfigurationClassUtils.getOrder(bd2.getBeanDefinition());
       return Integer.compare(i1, i2);
    });
    ...
    // 这里使用do-while语句依次解析所有的配置类
    ConfigurationClassParser parser = new ConfigurationClassParser(
          this.metadataReaderFactory, this.problemReporter, this.environment,
          this.resourceLoader, this.componentScanBeanNameGenerator, registry);
    Set<BeanDefinitionHolder> candidates = new LinkedHashSet<>(configCandidates);
    Set<ConfigurationClass> alreadyParsed = new HashSet<>(configCandidates.size());
    do {
       StartupStep processConfig = this.applicationStartup.start("spring.context.config-classes.parse");
       //这里就会通过Parser解析配置类中大部分内容，包括我们之前遇到的@Import注解
             parser.parse(candidates);
             parser.validate();
       //解析完成后读取到所有的配置类
       Set<ConfigurationClass> configClasses = new LinkedHashSet<>(parser.getConfigurationClasses());
             configClasses.removeAll(alreadyParsed);
       ... 
       //将上面读取的配置类加载为Bean
       this.reader.loadBeanDefinitions(configClasses);
       ...
    }
    while (!candidates.isEmpty());
    ...
}
```

### `ConfigurationClassParser#parse(candidates)`

``` java
public void parse(Set<BeanDefinitionHolder> configCandidates) {
    for (BeanDefinitionHolder holder : configCandidates) {
        BeanDefinition bd = holder.getBeanDefinition();
        try {
            if (bd instanceof AnnotatedBeanDefinition annotatedBeanDef) {
                parse(annotatedBeanDef, holder.getBeanName());
            }
            else if (bd instanceof AbstractBeanDefinition abstractBeanDef && abstractBeanDef.hasBeanClass()) {
                parse(abstractBeanDef.getBeanClass(), holder.getBeanName());
            }
            else {
                parse(bd.getBeanClassName(), holder.getBeanName());
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to parse configuration class [" + bd.getBeanClassName() + "]", ex);
        }
    }

    this.deferredImportSelectorHandler.process();
}
```

内部遍历candidates中的每一个BeanDefinitionHolder，调用parse的多态方法，最终调用`ConfigurationClassParser#processConfigurationClass`，最后调用`deferredImportSelectorHandler.process()`处理[DeferredImportSelector](#针对`DeferredImportSelector` "wikilink")相关的Bean注册 \^processConfigurationClass

首先判断条件注释，即处理`@Conditional`相关注解

然后将不同来源的配置类源信息通过`asSourceClass`进行封装，交给最核心的调用[doProcessConfigurationClass](#`ConfigurationClassParser doProcessConfigurationClass` "wikilink")

> 将配置类`ConfigurationClass`实例化为`SourceClass`。这样做的目的是为了让后续的处理逻辑能够通过`SourceClass`访问到配置类中定义的所有相关信息（比如注解信息，Meta-info），并进行相应的处理。例如，通过`SourceClass`可以读取配置类上的`@ComponentScan`注解，并执行组件扫描；读取`@Import`注解，并处理导入的配置类或组件；读取`@Bean`方法，并注册对应的Bean定义等。

#### `ConfigurationClassParser#doProcessConfigurationClass`

``` java
@Nullable
protected final SourceClass doProcessConfigurationClass(
        ConfigurationClass configClass, SourceClass sourceClass, Predicate<String> filter)
        throws IOException {

    if (configClass.getMetadata().isAnnotated(Component.class.getName())) {
        // Recursively process any member (nested) classes first
        processMemberClasses(configClass, sourceClass, filter);
    }

    // Process any @PropertySource annotations
    for (AnnotationAttributes propertySource : AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), org.springframework.context.annotation.PropertySource.class,
            PropertySources.class, true)) {
        if (this.propertySourceRegistry != null) {
            this.propertySourceRegistry.processPropertySource(propertySource);
        }
        else {
            logger.info("Ignoring @PropertySource annotation on [" + sourceClass.getMetadata().getClassName() +
                    "]. Reason: Environment must implement ConfigurableEnvironment");
        }
    }

    // Search for locally declared @ComponentScan annotations first.
    Set<AnnotationAttributes> componentScans = AnnotationConfigUtils.attributesForRepeatable(
            sourceClass.getMetadata(), ComponentScan.class, ComponentScans.class,
            MergedAnnotation::isDirectlyPresent);

    // Fall back to searching for @ComponentScan meta-annotations (which indirectly
    // includes locally declared composed annotations).
    if (componentScans.isEmpty()) {
        componentScans = AnnotationConfigUtils.attributesForRepeatable(sourceClass.getMetadata(),
                ComponentScan.class, ComponentScans.class, MergedAnnotation::isMetaPresent);
    }

    if (!componentScans.isEmpty()) {
        List<Condition> registerBeanConditions = collectRegisterBeanConditions(configClass);
        if (!registerBeanConditions.isEmpty()) {
            throw new ApplicationContextException(
                    "Component scan could not be used with conditions in REGISTER_BEAN phase: " + registerBeanConditions);
        }
        for (AnnotationAttributes componentScan : componentScans) {
            // The config class is annotated with @ComponentScan -> perform the scan immediately
            Set<BeanDefinitionHolder> scannedBeanDefinitions =
                    this.componentScanParser.parse(componentScan, sourceClass.getMetadata().getClassName());
            // Check the set of scanned definitions for any further config classes and parse recursively if needed
            for (BeanDefinitionHolder holder : scannedBeanDefinitions) {
                BeanDefinition bdCand = holder.getBeanDefinition().getOriginatingBeanDefinition();
                if (bdCand == null) {
                    bdCand = holder.getBeanDefinition();
                }
                if (ConfigurationClassUtils.checkConfigurationClassCandidate(bdCand, this.metadataReaderFactory)) {
                    parse(bdCand.getBeanClassName(), holder.getBeanName());
                }
            }
        }
    }

    // Process any @Import annotations
    processImports(configClass, sourceClass, getImports(sourceClass), filter, true);

    // Process any @ImportResource annotations
    AnnotationAttributes importResource =
            AnnotationConfigUtils.attributesFor(sourceClass.getMetadata(), ImportResource.class);
    if (importResource != null) {
        String[] resources = importResource.getStringArray("locations");
        Class<? extends BeanDefinitionReader> readerClass = importResource.getClass("reader");
        for (String resource : resources) {
            String resolvedResource = this.environment.resolveRequiredPlaceholders(resource);
            configClass.addImportedResource(resolvedResource, readerClass);
        }
    }

    // Process individual @Bean methods
    Set<MethodMetadata> beanMethods = retrieveBeanMethodMetadata(sourceClass);
    for (MethodMetadata methodMetadata : beanMethods) {
        configClass.addBeanMethod(new BeanMethod(methodMetadata, configClass));
    }

    // Process default methods on interfaces
    processInterfaces(configClass, sourceClass);

    // Process superclass, if any
    if (sourceClass.getMetadata().hasSuperClass()) {
        String superclass = sourceClass.getMetadata().getSuperClassName();
        if (superclass != null && !superclass.startsWith("java")) {
            boolean superclassKnown = this.knownSuperclasses.containsKey(superclass);
            this.knownSuperclasses.add(superclass, configClass);
            if (!superclassKnown) {
                // Superclass found, return its annotation metadata and recurse
                return sourceClass.getSuperClass();
            }
        }
    }

    // No superclass -> processing is complete
    return null;
}
```

该函数依次解决如下问题：
+ **处理@Component注解**
+ **处理@PropertySource和@PropertySources注解**
+ **处理@ComponentScan和@ComponentScans**
+ **处理@Import注解**
+ **处理@ImportResource注解**
+ **处理@Bean注解的方法**
+ **处理接口上的默认方法和超类**

其中的核心是处理`@Import`注解，通过调用 **[ConfigurationClassParser#processImports](#`ConfigurationClassParser processImports` "wikilink")**

#### `ConfigurationClassParser#processImports`

注意其第三个入参`Collection<SourceClass> importCandidates`，它是通过调用`getImports(sourceClass)`方法，从给定的`sourceClass`中提取所有`@Import`注解指定的类，如果sourceClass是普通的配置类，直接通过`isEmpty()`返回

``` java
private void processImports(ConfigurationClass configClass, SourceClass currentSourceClass,
        Collection<SourceClass> importCandidates, Predicate<String> filter, boolean checkForCircularImports) {

    if (importCandidates.isEmpty()) {
        return;
    }

    if (checkForCircularImports && isChainedImportOnStack(configClass)) {
        this.problemReporter.error(new CircularImportProblem(configClass, this.importStack));
    }
    else {
        this.importStack.push(configClass);
        try {
            for (SourceClass candidate : importCandidates) {
                if (candidate.isAssignable(ImportSelector.class)) {
                    // Candidate class is an ImportSelector -> delegate to it to determine imports
                    Class<?> candidateClass = candidate.loadClass();
                    ImportSelector selector = ParserStrategyUtils.instantiateClass(candidateClass, ImportSelector.class,
                            this.environment, this.resourceLoader, this.registry);
                    Predicate<String> selectorFilter = selector.getExclusionFilter();
                    if (selectorFilter != null) {
                        filter = filter.or(selectorFilter);
                    }
                    if (selector instanceof DeferredImportSelector deferredImportSelector) {
                        this.deferredImportSelectorHandler.handle(configClass, deferredImportSelector);
                    }
                    else {
                        String[] importClassNames = selector.selectImports(currentSourceClass.getMetadata());
                        Collection<SourceClass> importSourceClasses = asSourceClasses(importClassNames, filter);
                        processImports(configClass, currentSourceClass, importSourceClasses, filter, false);
                    }
                }
                else if (candidate.isAssignable(ImportBeanDefinitionRegistrar.class)) {
                    // Candidate class is an ImportBeanDefinitionRegistrar ->
                    // delegate to it to register additional bean definitions
                    Class<?> candidateClass = candidate.loadClass();
                    ImportBeanDefinitionRegistrar registrar =
                            ParserStrategyUtils.instantiateClass(candidateClass, ImportBeanDefinitionRegistrar.class,
                                    this.environment, this.resourceLoader, this.registry);
                    configClass.addImportBeanDefinitionRegistrar(registrar, currentSourceClass.getMetadata());
                }
                else {
                    // Candidate class not an ImportSelector or ImportBeanDefinitionRegistrar ->
                    // process it as an @Configuration class
                    this.importStack.registerImport(
                            currentSourceClass.getMetadata(), candidate.getMetadata().getClassName());
                    processConfigurationClass(candidate.asConfigClass(configClass), filter);
                }
            }
        }
        catch (BeanDefinitionStoreException ex) {
            throw ex;
        }
        catch (Throwable ex) {
            throw new BeanDefinitionStoreException(
                    "Failed to process import candidates for configuration class [" +
                    configClass.getMetadata().getClassName() + "]: " + ex.getMessage(), ex);
        }
        finally {
            this.importStack.pop();
        }
    }
}
```

代码遍历每一个`@Import`注解指定的候选类，根据不同类型进行处理
+ `ImportSelector`实现
+ `ImportSelector`
+ `DeferredImportSelector`
+ `ImportBeanDefinitionRegistar`实现
+ 普通的配置类

##### 针对`ImportSelector`

通过`selector.selectImports()`与`asSourceClasses()`方法将需要导入的类重新封装为`SourceClass`，递归调用`processImports`

##### 针对`DeferredImportSelector`

通过调用`ConfigurationClassParser`的内部类`DeferredImportSelectorHandler#handle()`方法，将其封装为`DeferredImportSelectorHolder` ，加入待处理的List - `deferredImportSelectors`

在`ConfigurationClassParser#parse`[处理完所有候选配置类后](#`ConfigurationClassParser parse(candidates)` "wikilink")，调用`DeferredImportSelectorHandler#process()`方法，该方法将加入`deferredImportSelectors`中的所有`DeferredImportSelectorHolder`执行内部类的`DeferredImportSelectorGroupingHandler#register`方法，得到包装好的、已经分组完毕的`DeferredImportSelectorGrouping`，然后调用`DeferredImportSelectorGroupingHandler#processGroupImports()`，处理组内所有的延迟导入 (`DeferredImportSelector`)

###### `DeferredImportSelectorGroupingHandler#register`

``` java
void register(DeferredImportSelectorHolder deferredImport) {
            Class<? extends Group> group = deferredImport.getImportSelector().getImportGroup();
            DeferredImportSelectorGrouping grouping = this.groupings.computeIfAbsent(
                    (group != null ? group : deferredImport),
                    key -> new DeferredImportSelectorGrouping(createGroup(group)));
            grouping.add(deferredImport);
            this.configurationClasses.put(deferredImport.getConfigurationClass().getMetadata(),
                    deferredImport.getConfigurationClass());
        }
```

- 首先尝试获取`DeferredImportSelector`指定的导入组 (`ImportGroup`)，如果没有指定特定的导入组，则使用`DeferredImportSelector`本身作为组的Key
- 尝试从一个名为`groupings`的映射中获取或创建一个与导入组对应的`DeferredImportSelectorGrouping`对象。如果映射中尚未存在与当前组对应的分组，那么将创建一个新的分组，并将其加入到映射中
  - 注意，此处的Group逻辑是将`DeferredImportSelector.Group`这个内部接口包装到`ConfigurationClassParser.DeferredImportSelectorGourping`这个内部类中，其内部维护了一个`DeferredImportSelector.Group`对象和`List<DeferredImportSelectorHolder>`对象
- 调用`DeferredImportSelectGrouping#add(DeferredImportSelectorHolder)`，将`DeferredImportSelectorHolder`加入内部类维护的`Grouping`中 (静态类)
- 最后，代码将当前`DeferredImportSelectorHolder`对应的配置类(`ConfigurationClass`)及其元数据添加到一个名为`configurationClasses`的映射中。这确保了后续能够快速访问到与特定`DeferredImportSelector`相关联的配置类

###### `DeferredImportSelectorGroupingHandler#processGroupImports`

``` java
void processGroupImports() {
    for (DeferredImportSelectorGrouping grouping : this.groupings.values()) {
        Predicate<String> filter = grouping.getCandidateFilter();
        grouping.getImports().forEach(entry -> {
            ConfigurationClass configurationClass = this.configurationClasses.get(entry.getMetadata());
            try {
                processImports(configurationClass, asSourceClass(configurationClass, filter),
                        Collections.singleton(asSourceClass(entry.getImportClassName(), filter)),
                        filter, false);
            }
            catch (BeanDefinitionStoreException ex) {
                throw ex;
            }
            catch (Throwable ex) {
                throw new BeanDefinitionStoreException(
                        "Failed to process import candidates for configuration class [" +
                                configurationClass.getMetadata().getClassName() + "]", ex);
            }
        });
    }
}
```

- 遍历保存在`Groups - DeferredImportSelectorGroupingHandler`中的 `DeferredImportSelectorGroup`对象，调用 `DeferredImportSelectorGroup#getImports()`方法
- `DeferredImportSelectorGroup#getImports()`方法调用`DeferredImportSelectorGroup`中维护的真实的Group - `DeferredImportSelector.Group#process`方法，然后返回含有`meta-info`的`Entry`
- 使用内部维护的`Map`(在`register`中put)，根据`Entry.meta-info`得到对应的`ConfigurationClass` ，调用`ConfigurationClassParser#processImports`，和[前面](#`ConfigurationClassParser processImports` "wikilink")一样递归调用进行处理

所以根据以上分析，`DeferredImportSelector`最终的处理逻辑在于`DeferredImportSelector.Group#process()` \^db8805

##### 针对`ImportBeanDefinitionRegistar`

``` java
public interface ImportBeanDefinitionRegistrar {  
    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry, BeanNameGenerator importBeanNameGenerator) {  
        this.registerBeanDefinitions(importingClassMetadata, registry);  
    }  
  
    default void registerBeanDefinitions(AnnotationMetadata importingClassMetadata, BeanDefinitionRegistry registry) {  
    }  
}
```

调用`ConfigurationClass#addImportBeanDefinitionRegistrar`方法，将对应的实例加入configClass对应的Collection类中，后续在[loadBeanDefinitions](#`ConfigurationClassBeanDefinitionReader loadBeanDefinitions` "wikilink")中调用其`registerBeanDefinitions`，注册相应的BeanDefinition

##### 针对普通配置类

不使用特殊机制，直接递归调用[processConfigurationClass](#^processConfigurationClass "wikilink")

### `ConfigurationClassParser#getConfigurationClasses`

返回从前面得到的所有待配置的配置类
### `ConfigurationClassBeanDefinitionReader#loadBeanDefinitions`

\^98f726

``` java
public void loadBeanDefinitions(Set<ConfigurationClass> configurationModel) {
        TrackedConditionEvaluator trackedConditionEvaluator = new TrackedConditionEvaluator();
        for (ConfigurationClass configClass : configurationModel) {
            loadBeanDefinitionsForConfigurationClass(configClass, trackedConditionEvaluator);
        }
    }

/**
 * Read a particular {@link ConfigurationClass}, registering bean definitions
 * for the class itself and all of its {@link Bean} methods.
 */
private void loadBeanDefinitionsForConfigurationClass(
        ConfigurationClass configClass, TrackedConditionEvaluator trackedConditionEvaluator) {

    if (trackedConditionEvaluator.shouldSkip(configClass)) {
        String beanName = configClass.getBeanName();
        if (StringUtils.hasLength(beanName) && this.registry.containsBeanDefinition(beanName)) {
            this.registry.removeBeanDefinition(beanName);
        }
        this.importRegistry.removeImportingClass(configClass.getMetadata().getClassName());
        return;
    }

    if (configClass.isImported()) {
        registerBeanDefinitionForImportedConfigurationClass(configClass);
    }
    for (BeanMethod beanMethod : configClass.getBeanMethods()) {
        loadBeanDefinitionsForBeanMethod(beanMethod);
    }

    loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
    loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
}
```

通过遍历每一个配置类，调用`loadBeanDefinitionsForConfigurationClass`方法 \^f6a27a

- registerBeanDefinitionForImportedConfigurationClass(configClass)
  注册配置类自身
- loadBeanDefinitionsForBeanMethod(beanMethod)
  注册@Bean注解标识的方法
- loadBeanDefinitionsFromImportedResources(configClass.getImportedResources());
  注册@ImportResource引入的XML配置文件中读取的bean定义
- loadBeanDefinitionsFromRegistrars(configClass.getImportBeanDefinitionRegistrars());
  注册configClass中经过解析后保存的所有ImportBeanDefinitionRegistrar，注册对应的BeanDefinition

# AOP

AOP的实现类是`AnnotationAwareAspectJAutoProxyCreator`，其是`BeanPostProcessor`的实现类，具体来说，是`InstantiationAwareBeanPostProcessor`的实现类，在实例化Bean过程中，通过调用[BeanPostProcessor中的实例化前处理器](#^fcb215 "wikilink")进行短路，得到相应的代理Bean

## `@EnableAspectJAutoProxy`

``` java
@Target({ElementType.TYPE})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Import({AspectJAutoProxyRegistrar.class})
public @interface EnableAspectJAutoProxy {
    boolean proxyTargetClass() default false;
    boolean exposeProxy() default false;
}
```

这个注解使用`@Import`导入了`AspectJAutoProxyRegistrar`，其是`ImportBeanDefinitionRegistrar`的实现类，会在[处理配置类](#^f6a27a "wikilink")相应`@Import`机制的时候将`AnnotationAwareAspectJAutoProxyCreator`实现类注册到容器中，即注册到BeanDefinition中，实现相应的实例化前处理器功能 (`InstantiationAwareBeanPostProcessor`)


<!--more-->
