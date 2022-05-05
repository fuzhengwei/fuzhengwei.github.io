# HTTP参数发送和接收

笔者之前对于HTTP参数的理解很落后，基本上属于调包侠级别的掌握程度。于是想了解一下HTTP请求参数的本质，本文会从理论和实践来讲解各种请求参数之间的不同之处。了解了HTTP请求参数的本质，我们才能更好的拓展与建设我们的Web应用。

## HTTP请求传参的几种格式

### URL传参

![image-20220505141923657](http://img.jjjzzzqqq.top/image-20220505141923657.png)

### Body传参

body中的数据格式与请求头中Content-Type字段有关系，该头部字段确定了body里的数据格式，常见的有这几种。

* multipart/form-data ： 需要在表单中进行文件上传时，就需要使用该格式
* application/json： JSON数据格式
* application/x-www-form-urlencoded ：前端表单中默认的encType，form表单数据被编码为key/value格式发送到服务器（表单默认的提交数据的格式）
* text/html ： HTML格式
* text/plain ：纯文本格式

更详细的可以参阅网上的详细文档教程。

接下来我们使用ApiFox模拟HTTP请求发送，来看一看各种格式的body数据会是什么样的HTTP报文

![image-20220505144543172](http://img.jjjzzzqqq.top/image-20220505144543172.png)

* form-data格式：每一个数据都在一个单独分隔中，boundary代表分隔符，数据中的name代表该数据的name，之后在单独的一行中表示值。

  ![image-20220505142124979](http://img.jjjzzzqqq.top/image-20220505142124979.png)

* x-www-form-urlencoded格式：前端表单默认的数据格式，body中用key=value&key=value的格式来传递所有数据，表现形式与url编码一样

  ![image-20220505142250162](http://img.jjjzzzqqq.top/image-20220505142250162.png)

* json格式：body中就是一个JSON字符串，content-type为application/json，但是其body中的表现形式还是纯文本，这也意味着其实Json格式的数据只不过是一个约定，数据传输都按照JSON约定去编码解码，本质上JSON其实就是一种纯文本协议

  ![image-20220505142353646](http://img.jjjzzzqqq.top/image-20220505142353646.png)

* raw格式：纯文本格式，body里填入啥数据就是啥数据。

  ![image-20220505142445934](http://img.jjjzzzqqq.top/image-20220505142445934.png)

## SpringMVC接收参数

### @RequestParm

该注解一旦使用，那么就能从HTTP报文中获取参数，它可以从以下几个地方获取参数

1. 可以接收url参数
2. 可以接收格式为form-data的body参数

而如果不使用@RequestParm注解的话，能接收到的参数与加了该注解一致，等于@RequestParm(required = false)的情况，不推荐使用。

注意：当接收对象的时候，使用@RequestParm注解和不使用有不同的表现。

![image-20220505153159915](http://img.jjjzzzqqq.top/image-20220505153159915.png)

1. 使用@RequestParm注解：无论如何发送都会报错
2. 不使用@RequestParm：能接收通过form-data格式传递的body数据，解析成一个对象



### @RequestBody

一般用来接收对象，且content-type格式只能为application/json，其他都会报错。



### 最佳实践

1. 对于key/value类型的参数，发送时可以用url参数时传输，也可以放在form-data里传递，后端接收时一律使用@RequestParm注解接收。
2. 对于对象类型的参数，有两种方法
    1. 将属性以key/value的形式以form-data的格式放入body中，后端直接用对象接收，不加任何注解。
    2. 将对象以Json格式放入body中，后端用@RequestBody接收。

## Go http库接收参数

Todo