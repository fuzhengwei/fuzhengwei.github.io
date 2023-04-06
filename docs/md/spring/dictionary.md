## beanFactory
beanFactory相关的类给我们提供了最基本的对象管理的功能，像对象的解析、beanDefinition注册、对象生成

## ApplicationContext
applicationContext以及其相关的实现类在beanFactory的基础上，进行了能力的扩充，通过刷新方法，让我们可以在
对象生成的前后去执行各种前置后置事件，去注册监听各种事件的发布。

## ioc的理解
按照定义来说，ioc就是控制反转，将对象以及对象的控制交给了spring工厂来管理。