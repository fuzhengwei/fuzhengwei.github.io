#### 基于xml
```xml
<beans>
    <!--1.属性注入-->
    <!--通过property、value、ref进行注入,该方式需要提供set方法-->
    <bean class="com.test.ioc.module.Distributor" name="zhangsan">
        <property name="name" value="zhangsan"/>
        <property name="age" value="18"/>
    </bean>
    
<!--    2.构造器注入-->
<!--    通过constructor-arg 标签进行-->
    <bean class="com.test.ioc.module.DistributionTeam">
        <constructor-arg name="distributor" ref="zhangsan"/>
    </bean>
        
<!--    3.工厂方法注入-->
<!--    工厂方法是静态方法时只需要指定factory-bean即可，如果非静态方法需要同时指定factory-bean-->
    <bean class="com.test.ioc.module.DistributorFactoryBean" name="distributorFactoryBean"/>
    <bean class="com.test.ioc.module.Distributor" name="lisi" factory-method="getObject" factory-bean="distributorFactoryBean"/>
</beans>
```


#### 基于注解
@Bean注解声明在方法上可以将返回值作为bean注入到容器，可与@ConditionalOnMissingBean搭配；

```text
    @Value("${}")
    可以获取对应属性文件中定义的属性值。例如
    user:
        name: togally
    可以使用@Value("${user.name}")的方式给属性注入值togally
    
    @Value("#{}")
    表示 SpEl 表达式通常用来获取 bean 的属性，或者调用 bean 的某个方法,例如
    @Value("#{systemProperties['os.name']}")
    表示获取名为systemProperties的Bean中的os属性的name属性
    
    
    @Value("") 
    直接注入字符串
```


#### 基于配置
```text
    @Configuration 
    配置类声明
    @ConfigurationProperties(prefix="jdbc") 
    批量注入属性
    @PropertySource("classpath:jdbc.properties") 
    指定外部属性文件。在类上添加
```