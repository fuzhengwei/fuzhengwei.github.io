## bean概述

我们通过xml文件或者注解等方式给spring提供了一些metadata config这些配置再spring容器中所代表的就是beanDefinition对象
通常来说beanDefinition包含如下对象:

```text
1.类名，通常是beanDefinition所代表的对象的实际类型
2.与bean行为相关的一些配置项（决定bean再容器中如何运行的一些配置，例如scope,lifecycle callbacks等等）
3.其他bean的引用（beanReference类）
4.Other configuration settings to set in the newly created object — for example, the size limit of the pool or the number of connections to use in a bean that manages a connection pool.
```

<table border = "1" width="500px" cellspacing = "10">
<tr>
<th align="left">Property</th>
<th align="left">desc</th>
</tr>
<tr>
    <td>Class</td>
    <td>实例化bean的类路径</td>
</tr>
<tr>
    <td>Name</td>
    <td>beanName</td>
</tr>
<tr>
    <td>scope</td>
    <td>bean作用域</td>
</tr>
<tr>
    <td>Constructor arguments</td>
    <td>构造器参数</td>
</tr>
<tr>
    <td>Properties</td>
    <td>属性</td>
</tr>
<tr>
    <td>Autowiring mode</td>
    <td>自动注入方式(byName byType ...)</td>
</tr>
<tr>
    <td>Lazy initialization mode</td>
    <td>是否启动懒加载</td>
</tr>
<tr>
    <td>initialization method</td>
    <td>初始化方法(lifecycle callbacks中的一种可以再初始化时或者设置属性后回调方法)</td>
</tr>
<tr>
    <td>Destruction method</td>
    <td>销毁方法(实例销毁后去执行的方法例如DisposableBean接口)</td>
</tr>
</table>

除了常规的再代码中或者xml中描述beanDefinition然后由容器自动创建管理bean，
我们也可以自己向容器中注册bean或者beanDefinition；

该方法由DefaultListableBeanFactory提供，分别是registerSingleton和registerBeanDefinition

## bean的命名
一般bean会有一个唯一标识，如果需要多个标识可以使用别名
在xml中我们使用id和name属性来标识一个beanDefinition，id是唯一标识，name可以指定多个别名，用, ; 或者空格来分割

正常情况下如果你不指定id和name那么容器会给你自行分配一个唯一的name，但是如果你需要使用ref或者其他方式引用bean ，那么你就需要给那个
bean来指定一个name，因为内部组装的方式和name有关.

我们一般使用驼峰的方式来命名bean




https://docs.spring.io/spring-framework/reference/core/beans/definition.html