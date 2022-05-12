# Maven进阶入门详解

在一次需求迭代中，同事要求我把写好的RPC接口打好包上传到公司私服上，我人直接当场懵逼住了。

![](http://img.jjjzzzqqq.top/tempImage1652272886269.gif)

突然发现自己对于Maven仅仅是处于最基础的使用阶段，不仅不知道背后的一些原理，甚至连一些常见的概念都不是很清晰，仅仅会使用Maven构建项目，引入依赖，打包等最基础的操作，所以连忙补补课，成功完成了需求，并且在此处总结一下Maven中稍微进阶一点的知识。

## 依赖

依赖是我们在使用Maven构建项目时最常使用的功能，通过依赖标签，我们可以直接从Maven仓库中引入对应的Jar包，无需手动再将Jar添加到目录下了，可谓是十分方便，不过我们除了使用，还需要考虑多模块下依赖之间的关系。

### 依赖配置

这个大家应该都很熟悉了，通过`<dependency>`标签引入Maven依赖

```xml
<dependencies>	
	<!-- servlet包 -->
     <dependency>
          <groupId>javax.servlet</groupId>
          <artifactId>javax.servlet-api</artifactId>
     </dependency>
</dependencies>
```

引入依赖之后，刷新一下Maven依赖就可以引入相关的Jar包了。

### 依赖传递

依赖具有传递性，当我们引入了一个依赖的时候，就会自动引入该依赖引入的所有依赖，依次往下引入所有依赖。

比如我们引入了Druid数据库连接池的SpringBoot-Starter，那么就会自动引入一些依赖

![image-20220511212623069](http://img.jjjzzzqqq.top/image-20220511212623069.png)

如图，我们仅仅引入了druid-spring-boot-starter依赖，就自动引入了该依赖依赖的依赖。总而言之就是套娃就完事了。

我们将这三个依赖称为间接引入的依赖，而我们在`<dependency>`标签中引入的依赖称为直接依赖，那么如果这两个重复了并且版本不一样的话会怎么办呢，最后引入的到底是哪个版本呢，还是说都会引入呢？

如果重复了，遵从以下规则

![image-20220511213047369](http://img.jjjzzzqqq.top/image-20220511213047369.png)

简单来说，就是越在外层的优先级越高，如果同级的就按照配置顺序，配置顺序靠前的覆盖配置顺序靠后的。

### 可选依赖

可选依赖指对外隐藏当前所依赖的资源

```xml
<dependency>
	<groupId>junit</groupId>
	<artifactId>junit</artifactId>
	<version>4.12</version>
	<optional>true</optional>
</dependency>

```

配置了该选项之后，间接依赖就失效了。

### 排除依赖

排除依赖指主动断开间接依赖的资源

```xml
<dependency>
	<groupId>junit</groupId>
	<artifactId>junit</artifactId>
	<version>4.12</version>
	<exclusions>
		<exclusion>
			<groupId>org.hamcrest</groupId>
			<artifactId>hamcrest-core</artifactId>
		</exclusion>
	</exclusions>
</dependency>

```

配置了该选项之后，间接依赖也会失效。

排除依赖和可选依赖的区别：

可选依赖是依赖提供者设置的，比如我们引入了Durid，那么该选项由Durid开发者设置

排除依赖由依赖引入者设置，比如我们引入了Durid，那么我们可以设置该选项

### 依赖范围

依赖的jar默认情况可以在任何地方使用，可以通过scope标签来改变依赖的作用范围。

![image-20220511215229028](http://img.jjjzzzqqq.top/image-20220511215229028.png)

主代码指的是main文件夹下的代码，测试代码指的是test文件夹下的代码（就那个绿色的玩意），打包指的是maven package指令执行时是否将Jar包打包。

其实如果我们偷懒的话，全部都默认也不是不可能，不过为了我们程序代码的可读性与简洁性，还是按照规范来比较好。

## 生命周期与插件

### 项目构建生命周期

Maven项目构建生命周期描述的是一次构建过程经历了多少个事件，我们可以把生命周期当成一个人的年龄。

Maven将生命周期划分为三个大阶段，类似于人类的婴儿，青年，入土

* clean：清理工作
* default：核心工作，例如编译，测试，打包，部署
* site：产生报告，发布站点

第一个和第三个周期比较简单，我们重点介绍一下default阶段

先上一张劝退图

![image-20220511220127441](http://img.jjjzzzqqq.top/image-20220511220127441.png)

以上就是defalut阶段完整的生命周期，其中标红的地方，是几个比较重要的周期，在Idea的Maven工具中也能体现出来

![image-20220511220227426](http://img.jjjzzzqqq.top/image-20220511220227426.png)

当我们在Idea中点击这几个生命周期时，Maven会自动将之前所有的生命周期都执行到，就类似于如果我18岁了，那么我肯定经历过8岁。

### 插件

插件就是Idea中Maven工具的Plugins部分

![image-20220511220919025](http://img.jjjzzzqqq.top/image-20220511220919025.png)

通过pom文件中的`<build></build>`标签引入新的插件

```xml
<build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.1</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
</build>
```

那么什么是插件呢？

* **插件与生命周期内的阶段绑定**，在**执行到对应生命周期时执行对应的插件功能**

* 默认maven在各个生命周期上绑定有预设的功能

* 通过插件可以自定义其他功能

  ```xml
  <build>
  	<plugins>
  		<plugin>
  			<groupId>org.apache.maven.plugins</groupId>
  			<artifactId>maven-source-plugin</artifactId>
  			<version>2.2.1</version>
  			<executions>
  				<execution>
  					<goals>
  						<goal>jar</goal>
  					</goals>
  					<phase>generate-test-resources</phase>
  				</execution>
  			</executions>
  		</plugin>
  	</plugins>
  </build>
  ```

  上述自定义插件的作用指的是在generate-test-resources生命周期执行打jar包的操作。

其实简单的说，生命周期就是一个人的年龄阶段，而插件就是每个人在每个年龄需要做的事情

总结：

<img src="http://img.jjjzzzqqq.top/image-20220511222130111.png" alt="image-20220511222130111" style="zoom:67%;" />

Maven将一个项目构建的过程分为一长串连续的生命周期，在对应的生命周期会通过插件完成对应的事件，通过使用Maven的生命周期，我们可以获得我们需要的功能，可能是打jar包，可能是安装到本地仓库，可能是部署到私服。

## 模块聚合

当使用Maven进行多模块开发的时候，有可能出现A模块依赖B模块，B模块依赖C模块，那么我们如果想对A模块打包，那么就要先打包C模块，再打包B模块，最后打包A模块才能成功，否则会报错，并且，如果C模块更新了，我们也要手动更新所有依赖C模块的模块，这样是及不方便的，Maven为了更好的进行多模块开发，提供了模块聚合的功能。

作用：**聚合用于快速构建Maven工程，一次性构建多个项目/模块**

* 使用步骤，我们用开源项目ruoyi的项目结构来看一下聚合在ruoyi中的使用

  1. ![image-20220512093658401](http://img.jjjzzzqqq.top/image-20220512093658401.png)

  2. RuoYi-Vue父模块的pom文件

     ```xml
     <!--聚合的所有模块-->
     <modules>
             <module>ruoyi-admin</module>
             <module>ruoyi-framework</module>
             <module>ruoyi-system</module>
             <module>ruoyi-quartz</module>
             <module>ruoyi-generator</module>
             <module>ruoyi-common</module>
     </modules>
     <!--打包类型定义为pom-->
     <packaging>pom</packaging>
     ```

  3. 直接对打包类型为pom的模块进行生命周期的管理，Maven会自动帮我们管理聚合的所有模块的生命周期，操作顺序跟依赖顺序有关系。



## 模块继承

还是在多模块项目开发中，多个子模块可能会引入相同的依赖，但是他们有可能会各自使用不同的版本，版本问题，有可能会导致最后构建的项目出问题，所以我们需要一种机制，来约定子模块的相关配置，于是就有了模块继承

作用：通过继承可以实现在子工程中沿用父工程中的配置

实现步骤：还是以ruoyi为例

1. 在子工程中声明其父工程坐标与对应的位置

   ```xml
    <parent>
           <artifactId>ruoyi</artifactId>
           <groupId>com.ruoyi</groupId>
           <version>3.8.1</version>
    </parent>
   ```

2. 在父工程中定义依赖管理

   ```xml
   <dependencyManagement>
           <dependencies>
               <!-- SpringBoot的依赖配置-->
               <dependency>
                   <groupId>org.springframework.boot</groupId>
                   <artifactId>spring-boot-dependencies</artifactId>
                   <version>2.5.8</version>
                   <type>pom</type>
                   <scope>import</scope>
               </dependency>
               <!-- 阿里数据库连接池 -->
               <dependency>
                   <groupId>com.alibaba</groupId>
                   <artifactId>druid-spring-boot-starter</artifactId>
                   <version>${druid.version}</version>
               </dependency>
               <!-- SpringBoot集成mybatis框架 -->
               <dependency>
                   <groupId>org.mybatis.spring.boot</groupId>
                   <artifactId>mybatis-spring-boot-starter</artifactId>
                   <version>${mybatis-spring-boot.version}</version>
               </dependency>
               <!-- pagehelper 分页插件 -->
               <dependency>
                   <groupId>com.github.pagehelper</groupId>
                   <artifactId>pagehelper-spring-boot-starter</artifactId>
                   <version>${pagehelper.boot.version}</version>
               </dependency>
       </dependencies>
   </dependencyManagement>
   ```

3. 定义完成之后，子工程相关的依赖就无需定义版本号，会直接使用父工程的版本号

   ```xml
   <dependency>
               <groupId>com.github.pagehelper</groupId>
               <artifactId>pagehelper-spring-boot-starter</artifactId>
   </dependency>
   ```

继承除了依赖版本号之外，还会继承一些资源，如下图

![image-20220512095525614](http://img.jjjzzzqqq.top/image-20220512095525614.png)

## 属性

在Maven中，对于有些依赖可能需要保证相同的版本，比如Spring相关依赖，那么我们就需要一个机制来保证这些依赖的版本都相同，我们可以使用Maven中的属性，类似编程语言的全局变量。

Maven中有很多属性：

1. 自定义属性
2. 内置属性
3. Setting属性
4. Java系统属性
5. 环境变量属性

此处我们重点讲解一下

### 自定义属性

作用：将一些字符串定义为变量，方便统一维护

使用步骤：还是以ruoyi为例

1. 定义自定义属性

   ```xml
   <properties>
           <ruoyi.version>3.8.1</ruoyi.version>
   </properties>
   ```

2. 调用：${xxx.yyy}

   ```xml
   <groupId>com.ruoyi</groupId>
   <artifactId>ruoyi</artifactId>
   <version>${ruoyi.version}</version>
   ```

### 内置属性

作用：使用Maven内置属性，快速配置一些文件

```xml
${basedir}
${version}
```

### Setting属性

作用：使用Maven配置文件setting.xml中的标签属性，用于动态配置

```xml
${settings.localRepository}
```

### Java系统属性

作用：读取Java系统属性

调用格式

```
${user.home}
```

系统属性查询方式

```
mvn help:system
```

### 环境变量属性

作用：使用Maven环境变量

```
${env.JAVA_HOME}
```

## 版本管理

对于我们的项目来说，如果我们将其放到一些Maven仓库中，那么就需要对其进行版本控制，我们可以看一下一些开源项目的Maven官网上的版本。

![image-20220512102131360](http://img.jjjzzzqqq.top/image-20220512102131360.png)

pom文件配置

```xml
<version>1.0.0.RELEASE</version>
```

工程版本号约定

![image-20220512104407356](http://img.jjjzzzqqq.top/image-20220512104407356.png)

工程版本

![image-20220512104610021](http://img.jjjzzzqqq.top/image-20220512104610021.png)



## 环境配置

一个项目，开发环境、测试环境、生产环境的配置文件必然不同，那么Maven就需要进行多环境配置管理

Maven多环境对应Idea中Maven工具的Profiles

![image-20220512104922780](http://img.jjjzzzqqq.top/image-20220512104922780.png)

配置文件：通过`<profiles>`配置文件配置，一个profile代表一个可选项

```xml
<profiles>
        <profile>
            <id>local</id>
            <properties>
                <!-- 环境标识，需要与配置文件的名称相对应 -->
                <profiles.active>local</profiles.active>
                <logging.level>debug</logging.level>
            </properties>
        </profile>
        <profile>
            <id>dev</id>
            <properties>
                <!-- 环境标识，需要与配置文件的名称相对应 -->
                <profiles.active>dev</profiles.active>
                <logging.level>debug</logging.level>
            </properties>
            <activation>
                <!-- 默认环境 -->
                <activeByDefault>true</activeByDefault>
            </activation>
        </profile>
        <profile>
            <id>test</id>
            <properties>
                <profiles.active>test</profiles.active>
                <logging.level>debug</logging.level>
            </properties>
        </profile>
        <profile>
            <id>prod</id>
            <properties>
                <profiles.active>prod</profiles.active>
                <logging.level>warn</logging.level>
            </properties>
        </profile>
</profiles>
```

然后我们在application.yml配置文件中设置即可，之后通过设置maven的profiles，就可以动态调整环境了。

![image-20220512110010056](http://img.jjjzzzqqq.top/image-20220512110010056.png)

## 私服

Maven私服指的是企业自己搭建的Maven仓库，通过Maven私服，第三方组织可以把自己组织内部的Maven依赖安装到私服上，提供给组织内部使用，搭建完私服之后，通过配置Maven，我们不止可以从中央仓库中获取Maven依赖，还可以从私服中获取Maven依赖。

下图是获取资源的过程，中央仓库的资源会从中央仓库获取，其他资源会从私服仓库获取

![](http://img.jjjzzzqqq.top/image-20220512111640527.png)



### 私服搭建

通过Nexus搭建私服

Nexus是Sonatype公司的一款Maven私服产品

下载地址：[Download (sonatype.com)](https://help.sonatype.com/repomanager3/product-information/download)

![image-20220512112052630](http://img.jjjzzzqqq.top/image-20220512112052630.png)

### 私服仓库介绍

安装好之后我们来看一下私服默认的仓库列表

![image-20220512112134176](http://img.jjjzzzqqq.top/image-20220512112134176.png)

可以将这些仓库分为三大类

* **宿主仓库hosted**：保存无法从中央仓库获取的资源
  * 自主研发
  * 第三方非开源项目
* **代理仓库proxy**
  * 代理远程仓库，通过nexus访问其他公共仓库
* **仓库组**：将若干个仓库组成一个群组，简化配置，它仅仅是一种配置，不是真实的仓库
  * 比如我们可以将二课项目相关的依赖放到一个仓库组中，将抽奖项目的依赖放到一个仓库组中

创建私服仓库

点击create repository

<img src="http://img.jjjzzzqqq.top/image-20220512124904034.png" alt="image-20220512124904034" style="zoom:67%;" />

选择maven2（hosted）

![image-20220512124926302](http://img.jjjzzzqqq.top/image-20220512124926302.png)

填入仓库名称

<img src="http://img.jjjzzzqqq.top/image-20220512124948968.png" alt="image-20220512124948968" style="zoom:67%;" />



创建完之后在仓库列表可见，将新建的仓库加入maven-public仓库组，之后通过该仓库组的url访问

![image-20220512125248166](http://img.jjjzzzqqq.top/image-20220512125248166.png)

点击maven-public仓库组

<img src="http://img.jjjzzzqqq.top/image-20220512125108385.png" alt="image-20220512125108385" style="zoom: 67%;" />



### 本地仓库访问私服配置

配置本地仓库访问私服的权限（setting.xml文件），如果你想从这个仓库中获取或者部署资源，那么就需要server配置来验证权限，此处可以是不同的账号密码，不同的用户对于仓库的权限也不同。

**配置Servers**

```xml
<servers>
	<server>
		<id>ticknet-release</id>
		<username>admin</username>
		<password>admin</password>
	</server>
	<server>
		<id>ticknet-snapshots</id>
		<username>admin</username>
		<password>admin</password>
	</server>
</servers>
```

**配置setting.xml的Profiles**

```xml
    <profiles>
        <profile>
            <id>artifactory</id>
            <repositories>
                <repository>
                    <snapshots>
                        <enabled>false</enabled>
                    </snapshots>
                    <id>repo</id>
                    <name>repo</name>
                    <url>xxxx</url>
                </repository>
                <repository>
                    <snapshots/>
                    <id>snapshots</id>
                    <name>snapshots-only</name>
                    <url>xxxx</url>
                </repository>
            </repositories>
        </profile>
    </profiles>
```

此处的URL通过

![image-20220512151244007](http://img.jjjzzzqqq.top/image-20220512151244007.png)

这个copy按钮获取。

**配置激活profiles**  

```xml
 <activeProfiles>
        <activeProfile>artifactory</activeProfile>
 </activeProfiles>
```

之后就可以从私服获取资源了

### 上传资源到私服

配置项目pom文件

```xml
<distributionManagement>
	<repository>
		<id>ticknet-release</id>
		<url>http://localhost:8081/repository/ticknet-release/</url>
	</repository>
	<snapshotRepository>
		<id>ticknet-snapshots</id>
		<url>http://localhost:8081/repository/ticknet-release/</url>
		</snapshotRepository>
</distributionManagement>
```

配置完执行生命周期的deploy即可

OK，大功告成。
