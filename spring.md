# Spring
## IOC
### BeanDefinition
描述bean的信息
1. 作用域 单例 Prototype 还是
2. bean的名称，class等信息
3. 依赖关系，依赖于哪些bean
4. 是否是primary
5. 如果是@bean生成的，方法名称又是哪些
6. 初始化方法名
7. destroy方法名
.....

### 获取bean
1. 先解析名称，然后根据名称获取BeanDefinition
2. 根据BeanDefinition的DependsOn首先检查是否是循环的依赖，然后从容器获取依赖的bean
3. 创建bean
3. 调用MergedBeanDefinitionPostProcessor#postProcessMergedBeanDefinition可以对当前的BeanDefinition进行一些操作，比如AutowiredAnnotationBeanPostProcessor就会在这里进行@Autowired和@Value注解进行解析，合并到BeanDefinition中，告诉BeanDefinition需要注入哪些字段
4. 根据之前的解析，注入字段和方法
5. 调用InstantiationAwareBeanPostProcessor#postProcessProperties进行字段和方法的注入
6. 处理Aware的一些子接口，例如BeanFactoryAware，调用setBeanFactory方法
7. 调用org.springframework.beans.factory.config.BeanPostProcessor#postProcessBeforeInitialization方法，例如ApplicationContextAwareProcessor对EnvironmentAware进行方法调用注入
8. 调用一些初始化的方法，例如如果bean是InitializingBean的子类，则调用afterPropertiesSet方法或者是@bean的自定义init方法
9. 调用org.springframework.beans.factory.config.BeanPostProcessor#postProcessAfterInitialization方法，例如BeanValidationPostProcessor
就会在这里进行hibernate的注解校验

### 销毁beanfactory
1. 调用 org.springframework.beans.factory.DisposableBean#destroy
2. 标记了@PreDestroy的方法

## Event

### ApplicationEvent

#### ContextClosedEvent
容器关闭事件
#### ContextRefreshedEvent
容器刷新事件，例如在@Scheduled注解中，所有的bean装配完成后，需要启动定时任务，就会接收这个事件，然后开始注册任务，开始执行
#### ContextStoppedEvent
容器停止事件
#### ContextStartedEvent
容器已经开始的事件

## Resources

### Resource
对于资源的抽象，类似于vfs，与ResourceLoader搭配使用
#### WritableResource
#### ContextResource
#### UrlResource
#### FileUrlResource
#### FileSystemResource
#### ClassPathResource
classpath: a.xml
#### ByteArrayResource
#### InputStreamResource

## i18n

### MessageSource
存储信息
### ResourceBundleMessageSource
读取文件信息  根据baseName+locale
## Validation

### hibernate validation
使用hibernate的validate，框架进行bean的校验

## Type Conversion

### Converter 
两种类型的转换
例如 org.springframework.core.convert.support.StringToNumberConverterFactory.StringToNumber ，将string 转为 数字类型

### ConversionService

#### DefaultConversionService
提供一些默认的转换，例如StringToArray，根据","分割
#### FormattingConversionService
在 Converter api之外提供了 Formatter api，提供了string到另一个类型互转的能力，例如DateFormatter同时提供国际化能力


## SpEL
提供动态执行方法能力，类似mybatis的ognl
在@value取值就是用的 spel



## WebMVC
### FrameworkServlet
### DispatcherServlet
1. 首先DispatcherServlet是FrameworkServlet的子类
2. 然后在FrameworkServlet中，dopost，doget方法，会调用processRequest方法，然后会调用DispatcherServlet#doService，最后到DispatcherServlet#doDispatch
3. 首先检查是否是文件上传
4. 根据url找到具体的处理映射，如果找不到返回404
5. 根据处理映射找到能处理的HandlerAdapter
6. 进行拦截器的预处理
7. HandlerAdapter对具体方法的参数进行解析，并且调用具体的处理方法，然后对返回值进行解析
8. 如果处理器是异步处理的方法，则直接返回
9. 调用拦截器的处理方法
10. 如果是异步处理的方法，调用异步拦截器AsyncHandlerInterceptor#afterConcurrentHandlingStarted方法
11. 否则如果是文件上传的话，清除文件上传的缓存文件，释放资源


#### 主要的类与接口

##### HandlerMapping
用于url与具体的处理类的映射
例如`RequestMappingHandlerMapping`
就是处理url到@Controller类的映射根据url找到具体的处理controller

##### HandlerAdapter
1. 用于适配handlerMapping，如果handlerMapping是自己可以处理的，则处理
2. 例如`RequestMappingHandlerAdapter`，用来处理`RequestMappingHandlerMapping`
3. 处理handlerMapping的方法参数
4. 处理handlerMapping的方法返回值

##### HandlerInterceptor
拦截器

##### MappedInterceptor
mapping映射的拦截器

##### AsyncHandlerInterceptor
异步处理的拦截器

##### HandlerMethodArgumentResolver
用来解析`处理方法`参数
例如标记了`@RequestParam`注解的方法，就会使用RequestParamMethodArgumentResolver类去解析，根据@RequestParam#value的值从请求中获取，并且调用处理方法的时候将参数赋值到方法参数中。

##### HandlerMethodReturnValueHandler
处理调用具体处理方法的返回值
例如标记`@RequestBody`的处理方法，就会使用
RequestResponseBodyMethodProcessor类进行处理，对返回值进行json格式化



