Spring Boot有一个怎样使用Spring构建应用的[opinionated view](http://stackoverflow.com/questions/802050/what-is-opinionated-software)：例如它有存放常用配置文件的习惯位置,常用管理和监控任务的端点。Spring Cloud构建于此之上,并且添加了一些*可能包含了一个系统需要或者偶尔需要的所有组件*的特性。  
   
   ---

**应用引导上下文(The Bootstrap Application Context)**  

Spring Cloud应用通过为main application创建一个"bootstrap"父级上下文来运行。开箱即用对加载外部来源的配置属性是可靠的，并且解密本地外部配置文件的属性。这两个上下文为Spring应用共享同一个外部属性来源的环境。引导属性通过搞优先级来添加,这样就不能通过本地配置或默认配置来重写。

The bootstrap context uses a different convention for locating external configuration than the main application context, so instead of application.yml (or .properties) you use bootstrap.yml, keeping the external configuration for bootstrap and main context nicely separate. Example:  
引导上下文与main application上下文使用一个不同习惯来定位外部配置,因此使用`bootstrap.yml`来替换`application.yml`(或`bootstrap.properties`),保持两个上下文的外部配置更好的可读性。例如:  

bootstrap.yml  

    spring: 
      application: 
        name: foo
      cloud: 
        config: 
          uri: ${SPRING_CONFIG_URI:http://localhost:8888}  





如果你的应用需要任何来自服务器的面向应用的配置,在`bootstrap.yml`或`application.yml`中设置将会是个好主意。  

你完全可以通过设置`spring.cloud.bootstrap.enabled=false`来禁用bootstrap进程。

---
**应用上下文层级(Application Context Hierarchies)**  

如果你通过SpringApplication或SpringApplicationBuilder构建应用上下文,‘bootstrap‘上下文将会以一个父级上下文添加到应用中。与不通过Spring Cloud Config来构建相同的上下文相比,子级(继承方)上下文继承属性来源并描述父级上下文,这样’main‘上下文将包含额外的属性来源,这是Spring的一个特性。额外添加的属性来源是:  
- "bootstrap": 一个可选的`CompositePropertySource`  
如果任何`PropertySourceLocators`出现在引导上下文,并且含有非空的属性,将会呈现出一个可选的高优先权的CompositePropertySource。一个例子,从Spring Cloud Config Server来的属性。[怎样定制属性来源内容的说明](http://cloud.spring.io/spring-cloud-static/Brixton.SR5/#customizing-bootstrap-property-sources)。
- "applicationConfig:[classpath:bootstrap.yml]"(或类似的Spring激活的配置文件)。如果你有一个`bootstrap.yml`(或properties),那么这些属性将会用于配置引导上下文(++Bootstrap Context++),并且将会被添加到子级的上下文中(++child context++)。他们拥有比`application.yml`更低的权限,并且其他属性来源将会像创建Spring Boot application的进程正常部分添加到子类中。[怎样定制属性来源内容的说明](http://cloud.spring.io/spring-cloud-static/Brixton.SR5/#customizing-bootstrap-properties)。  
  

由于属性来源的排序规则,‘bootstrap’获得优先权,但是请注意,拥有非常第优先权但是可以用来设置默认值的`bootstrap.yml`它们不包含任何数据。  

你可以通过设置任何来自于你创建的ApplicationContext的父级上下文来集成上下文层级,或者通过`SpringApplicationBuilder`的便利方法(`parent()`,`child()`和`sibling()`)。引导上下文(++bootstrap context++)将会成为你的应用中最高级祖先的父级。层级中的每一个上下文都会有自己的"bootstrap"属性来源(可能为空),来避免不小心从父级推广到子级的值。层级中的每一个上下文都可以有一个不同的`spring.application.name`和不同的远程属性来源,如果有Config Server的话。普通Spring应用上下文行为规则适用于属性解决方案:子级上下文根据name和属性来源name重写父级中的属性(如果子级与父级中有相同name的属性来源,父级中的将不会包含在子级中)。  
注意`SpringApplicationBuilder`允许你在全部层级中共享一个环境,但是这不是默认的。因此,同级上下文不需要有相同的配置文件或属性来源,即使他们共享来自于父级的一些通用的东西。

---

**修改引导属性的位置(Changing the location of Bootstrap Properties)**  
 
`bootstrap.yml`(or.properties)的位置可以通过`spring.cloud.bootstrap.name`(默认"bootstrap")或者`spring.cloud.bootstrap.location`(默认空)来修改。这些属性表现的像`spring.config.*`有相同名字的变式,实际上这些属性用来在环境中设置,从而建立`bootstrap ApplicationContext`。如果有激活的配置文件(来自`spring.profiles.active`或者通过你构建上下文的环境中的API),那么配置文件中的属性将会被加载,就像一个正规的Spring Boot应用,例如对"development"的配置文件`bootstrap-development.properties`。  

---  
**重载远程属性值(Overriding the Values of Remote Properties)**  
