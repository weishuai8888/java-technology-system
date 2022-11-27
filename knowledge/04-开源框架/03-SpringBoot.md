# SpringBoot
## SpringBoot的优势
- 支持注解开发，可以避免冗余的配置文件；
- “约定大于配置”。SpringBoot Starter和SpringBoot Jpa都是“约定大于配置”的体现。SpringBoot Starter在启动的时候根据约定的信息对资源进行
初始化；SpringBoot JPA通过约定的方式来自动生成sql，避免大量的无效代码编写。 

## SpringBoot在启动时的工作过程
- Springboot在启动时会按照约定去读取Springboot Starter的配置信息，再根据配置信息对资源进行初始化，并注入到Spring容器中。这样SpringBoot
在启动完毕后就已经准备好了一切资源，使用过程中直接注入对应的Bean即可。（配置文件在resources/META-INF/spring.factories文件中）

## SpringBoot的自动装配只如何实现的
SpringBoot的启动注解@SpringBootApplication由三个注解组成：@SpringBootConfigration、@ComponentScan、	@EnableAutoConfigration。
其中@EnableAutoConfigration是自动配置的入口，通过@Import注解导入AutoConfigrationImportSelector，该类中加载了META-INF/spring.factories
的配置信息，然后筛选出以type.getName()为key的数据，加载到IOC容器中，实现自动配置的功能。

## Spring Boot自动配置原理
1. 从spring.factories配置文件中加载自动配置类；
2. 加载的自动配置类中排除掉@EnableAutoConfiguration 注解的 exclude 属性指定的自动配置类；
3. 然后再用AutoConfigurationImportFilter接口去过滤自动配置类是否符合其标注注解（若有标注的话）@ConditionalOnClass **,** @ConditionalOnBean 和 @ConditionalOnWebApplication 的条件，若都符合的话则返回匹配结果；
4. 然后触发 AutoConfigurationImportEvent 事件，告诉 ConditionEvaluationReport 条件评估报告器对象来分别记录符合条件和exclude 的自动配置类；
5. 最后 spring 再将最后筛选后的自动配置类导入 IOC 容器中。

## 为什么导入dependency时不需要指定版本？
spring-boot-starter-parent通过继承spring-boot-dependencies从而实现了SpringBoot的版本依赖管理。所以我们的SpringBoot工程继承了
spring-boot-starter-parent后已经具备版本锁定等配置了。这就是在SpringBoot项目中部分依赖不需要指定版本好的原因。

## spring-boot-start-parent父依赖启动器的主要作用是进行版本统一管理，那么项目运行依赖的jar包从何而来？
spring-boot-starter-web依赖启动器的主要作用是打包了web开发场景所需的底层所有依赖，所以在pom.xml中引入spring-boot-starter-web依赖启动
容器时，就可以实现web场景开发而不需要导入tomcat服务器以及其他web依赖文件等。同时SpringBoot除了提供上述介绍的web依赖容器外，还提供了许多其他
开发场景的相关依赖。具体可参考Spring boot官方文档。

## Spring Boot启动过程
- 第一步：获取并启动监听器
- 第二步：构造应用上下文环境
  包括计算机的环境，Java环境，Spring的运行环境，Spring项目的配置等等。
- 第三步：初始化应用上下文
- 第四步：刷新应用上下文前的准备阶段
- 第五步：刷新应用上下文
- 第六步：刷新应用上下文后的扩展接口

## 为什么要自定义starter
在我们的日常开发工作中，经常会有一些独立于业务之外的配置模块，我们经常将其放到一个特定的包下，然后如果另一个工程需要复用这块功能的时候，需要
将代码硬拷贝到另一个工程，重新集成一遍，麻烦至极。如果我们将这些可独立于业务代码之外的功配置模块封装成一个个starter，复用的时候只需要将其在pom
中引用依赖即可，再由SpringBoot为我们完成自动装配，就非常轻松了。

