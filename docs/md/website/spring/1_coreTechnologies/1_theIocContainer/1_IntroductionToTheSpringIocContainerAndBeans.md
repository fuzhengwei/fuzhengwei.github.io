## 页面地址
https://docs.spring.io/spring-framework/reference/core/beans/introduction.html

ioc是控制反转同时也被称为依赖注入,是指对象通过属性注入、构造器注入、工厂方法注入等途径来进行依赖分配的过程。(什么是Ioc)
 
application和bean包一起实现控制反转

beanFactory接口以及一系列实现提供了根据配置来进行bean对象管理的能力

application是BeanFactory接口的子接口，除了基本的能力还增加了如下功能:

1.aop 

2.消息资源处理(国际化)

3.事件发布能力

4.特定的上下文例如XmlApplicationContext/WebApplicationContext