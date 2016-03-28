---
layout: post
title: "Getting Started with Spring Roo"
category: Spring
tags: [spring, enterprise application, spring roo]
---

## Getting Started

### 前置条件

* Java
* Maven
* Spring Roo
* 以上环境和工具需要配置Path环境变量

### 初始化一个Roo工程

创建一个pizza的文件夹
{% highlight bash %}

> mkdir pizza
> cd pizza
pizza>

{% endhighlight %}

执行roo命令

{% highlight bash %}

pizza> roo
    ____  ____  ____
   / __ \/ __ \/ __ \
  / /_/ / / / / / / /
 / _, _/ /_/ / /_/ /
/_/ |_|\____/\____/    1.2.1.RELEASE [rev 6eae723]


Welcome to Spring Roo. For assistance press TAB or type "hint" then hit ENTER.
roo>
{% endhighlight %}

执行hint命令可以获取当前上下文可以执行的命令和提示, 对于当前情况, roo提示可以通过project命令创建一个工程

{% highlight bash %}
roo> hint
Welcome to Roo! We hope you enjoy your stay!

Before you can use many features of Roo, you need to start a new project.

To do this, type 'project' (without the quotes) and then hit TAB.

Enter a --topLevelPackage like 'com.mycompany.projectname' (no quotes).
When you've finished completing your --topLevelPackage, press ENTER.
Your new project will then be created in the current working directory.

Note that Roo frequently allows the use of TAB, so press TAB regularly.
Once your project is created, type 'hint' and ENTER for the next suggestion.
You're also welcome to visit http://stackoverflow.com/questions/tagged/spring-roo
for Roo help.
roo>
{% endhighlight %}

初始化一个ROO工程, 并设置根包名为`com.springsource.roo.pizzashop`

{% highlight bash %}
roo> project setup --topLevelPackage com.springsource.roo.pizzashop
Created ROOT/pom.xml
Created SRC_MAIN_RESOURCES
Created SRC_MAIN_RESOURCES/log4j.properties
Created SPRING_CONFIG_ROOT
Created SPRING_CONFIG_ROOT/applicationContext.xml
roo>
{% endhighlight %}

### 创建实体类和字段

获取提示通过键入`ent`(entity)+TAB*2获取关于创建实体类的更多信息

{% highlight bash %}

com.springsource.roo.pizzashop roo> hint
You can create entities either via Roo or your IDE.
Using the Roo shell is fast and easy, especially thanks to the TAB completion.

Start by typing 'ent' and then hitting TAB twice.
Enter the --class in the form '~.domain.MyEntityClassName'
In Roo, '~' means the --topLevelPackage you specified via 'create project'.

After specify a --class argument, press SPACE then TAB. Note nothing appears.
Because nothing appears, it means you've entered all mandatory arguments.
However, optional arguments do exist for this command (and most others in Roo).
To see the optional arguments, type '--' and then hit TAB. Mostly you won't
need any optional arguments, but let's select the --testAutomatically option
and hit ENTER. You can always use this approach to view optional arguments.

After creating an entity, use 'hint' for the next suggestion.
com.springsource.roo.pizzashop roo>

{% endhighlight %}

创建一个名为`Topping`的实体类, 并创建关于它的一系列测试

{% highlight bash %}

com.springsource.roo.pizzashop roo> entity jpa --class ~.domain.Topping --testAutomatically
Created SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain
Created SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain/Topping.java
Created SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain
Created SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain/ToppingDataOnDemand.java
Created SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain/ToppingIntegrationTest.java
Created SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain/Topping_Roo_Configurable.aj
Created SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain/Topping_Roo_ToString.aj
Created SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain/Topping_Roo_Jpa_Entity.aj
Created SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain/Topping_Roo_Jpa_ActiveRecord.aj
Created SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain/ToppingDataOnDemand_Roo_Configurable.aj
Created SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain/ToppingDataOnDemand_Roo_DataOnDemand.aj
Created SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain/ToppingIntegrationTest_Roo_Configurable.aj
Created SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain/ToppingIntegrationTest_Roo_IntegrationTest.aj

{% endhighlight %}

获取提示: 通过键入`field`命令来为`Topping`添加一个字段, 注意当前命令行环境在`~.domain.Topping`下

{% highlight bash %}

~.domain.Topping roo> hint
You can add fields to your entities using either Roo or your IDE.

To add a new field, type 'field' and then hit TAB. Be sure to select
your entity and provide a legal Java field name. Use TAB to find an entity
name, and '~' to refer to the top level package. Also remember to use TAB
to access each mandatory argument for the command.

After completing the mandatory arguments, press SPACE, type '--' and then TAB.
The optional arguments shown reflect official JSR 303 Validation constraints.
Feel free to use an optional argument, or delete '--' and hit ENTER.

If creating multiple fields, use the UP arrow to access command history.

After adding your fields, type 'hint' for the next suggestion.
To learn about setting up many-to-one fields, type 'hint relationships'.
~.domain.Topping roo>

{% endhighlight %}

为`Topping`添加一个类型为`java.lang.String`, 名称为`name`的字段

{% highlight bash %}

~.domain.Topping roo> field string --fieldName name --notNull --sizeMin 2
Updated SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain/Topping.java
Updated SRC_TEST_JAVA/com/springsource/roo/pizzashop/domain/ToppingDataOnDemand_Roo_DataOnDemand.aj
Created SRC_MAIN_JAVA/com/springsource/roo/pizzashop/domain/Topping_Roo_JavaBean.aj

{% endhighlight %}

通过`field set`新增一个many-to-many的关系

{% highlight bash %}
~.domain.Pizza roo> field set --fieldName toppings --type ~.domain.Topping
{% endhighlight %}

通过`field reference`添加一个many-to-one的关系

{% highlight bash %}
~.domain.Pizza roo> field reference --fieldName base --type ~.domain.Base
{% endhighlight %}

### 集成测试

{% highlight bash %}
~.domain.PizzaOrder roo> perform tests
...
-------------------------------------------------------
 T E S T S
-------------------------------------------------------

Tests run: 36, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 3.860s
[INFO] Finished at: Tue Feb 14 18:01:45 EST 2012
[INFO] Final Memory: 6M/81M
[INFO] ------------------------------------------------------------------------
{% endhighlight %}

### 生成web层

`web mvc setup`命令会将当前应用转变为一个web程序,  并为该应用添加web应用相应的依赖, 如Spring MVC, Tiles等

{% highlight bash %}
~.domain.PizzaOrder roo> web mvc setup
{% endhighlight %}

为当前所有实体生成web层代码, 并将所有生成的web层代码放到`com.springsource.roo.pizzashop.web`包下

{% highlight bash %}
~.domain.PizzaOrder roo> web mvc all --package ~.web
{% endhighlight %}

### 启动这个web项目

* 通过maven的命令`mvn tomcat:run`或`mvn jetty:run`启动
* 将项目引入IDE, 利用IDE的Run或Deploy功能启动

### 添加Spring Security来保护应用

{% highlight bash %}

~.web roo> security setup
Created SPRING_CONFIG_ROOT/applicationContext-security.xml
Created SRC_MAIN_WEBAPP/WEB-INF/views/login.jspx
Updated SRC_MAIN_WEBAPP/WEB-INF/views/views.xml
Updated ROOT/pom.xml [added property 'spring-security.version' = '3.1.0.RELEASE'; added dependencies
org.springframework.security:spring-security-core:${spring-security.version},
org.springframework.security:spring-security-config:${spring-security.version},
org.springframework.security:spring-security-web:${spring-security.version},
org.springframework.security:spring-security-taglibs:${spring-security.version}]
Updated SRC_MAIN_WEBAPP/WEB-INF/web.xml
Updated SRC_MAIN_WEBAPP/WEB-INF/spring/webmvc-config.xml
{% endhighlight %}

再次尝试启动应用, 发现访问需要登录

## ROO工程的架构

## 总结

Spring Roo的一些优缺点

优点:

* 既支持通过Entities到DB模型转换(默认), 也支持从DB模型到Entities的转换(DBRE)

缺点:

* 目前来说插件比较少
* 整个开发过程与IDE结合不是很好, Entity设计大部分情况需要在命令行下执行, 这点当然可以通过DBRE或其他方式来解决
* 不支持代码模板

特点:

* ROO工程大量的使用了AspectJ的ITD定义文件, 来将实体类的属性以及其方法动作分离
