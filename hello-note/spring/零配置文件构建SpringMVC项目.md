## 零配置文件构建Spring Web项目
<br>

当我们使用Spring MVC来构建一个web应用时，有两个基础的配置文件必不可少 —— **dispatcher-servlet.xml**和**web.xml**。后续为满足系统业务需要，还要编写大量的配置关系到xml文件中去，而大量的配置文件和配置文件的命名空间的管理已经越来越成为Spring的累赘。

今天，我们来看看如何使用Spring的注解并以代码的方式去除这些繁琐的配置文件。

### 1. 去除web.xml文件
下面是一个很普通的web.xml文件，这里面定义了一个SpringMVC的servlet并委托其初始化Spring配置文件
```xml
<?xml version="1.0" encoding="UTF-8"?>
<web-app xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns="http://java.sun.com/xml/ns/javaee"
	xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_3_0.xsd"
	version="3.0">

	<servlet>
		<servlet-name>springmvc</servlet-name>
		<servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
		<init-param>
			<param-name>contextConfigLocation</param-name>
			<param-value>/WEB-INF/context.xml</param-value>
		</init-param>
        <load-on-startup>1</load-on-startup>
	</servlet>

	<servlet-mapping>
		<servlet-name>springmvc</servlet-name>
		<url-pattern>/</url-pattern>
	</servlet-mapping>

</web-app>
```
接下来我们将结合servlet 3+的新特性以及spring-web模块来代替上面的配置。先上代码：
```java
public class MyWebApplicationInitializer implements WebApplicationInitializer {

	@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		ServletRegistration.Dynamic dispatcher = servletContext.addServlet("dispatcher", new DispatcherServlet());
		dispatcher.setInitParameter("contextConfigLocation", "/WEB-INF/dispatcher-servlet.xml");
		dispatcher.setLoadOnStartup(1);
		dispatcher.addMapping("/");
	}
}
```
新建一个类实现spring-web包中的`WebApplicationInitializer`接口，里面只有一个方法`onStartup`，方法中的实现相信大家通过api的语义也能明白，就是将之前web.xml中配置的servlet用代码的方式呈现出来。

好，那么问题来了，这个类为什么就可以替代web.xml？它又是如何工作的呢？

在servlet 3以后，提供了可插拔的web组件（servlet、filter）功能，其中对于web容器的配置加载采用SPI的方式，通过实现`ServletContainerInitializer`接口，并在包中声明其实现类的全类名，便可由web容器启动后自动加载。

回到我们的代码中，在spring-web.jar中，可以找到`/META-INF/services/javax.servlet.ServletContainerInitializer`文件，里面的内容为`org.springframework.web.SpringServletContainerInitializer`，我们查看这个类发现，它实现了`ServletContainerInitializer`，并且在类声明处有注解`@HandlesTypes(WebApplicationInitializer.class)`，这个注解的意思是在classpath下扫描某个接口的所有实现类并传给使用它的类。

到这里我们就知道了web容器启动时会加载`SpringServletContainerInitializer`，而它里面干的事就是执行所有实现`WebApplicationInitializer`接口的类，那么我们上面的代码就会被执行了。去除web.xml目标完成。

### 2. 去除dispatcher-servlet.xml
在1的代码中可以发现，我们仍需要编写`dispatcher-servlet.xml`配置文件，下面是一份简单的该配置文件内容：
```xml
<beans xmlns="http://www.springframework.org/schema/beans"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xmlns:context="http://www.springframework.org/schema/context"
	xmlns:mvc="http://www.springframework.org/schema/mvc"
	xsi:schemaLocation="http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd
         http://www.springframework.org/schema/context http://www.springframework.org/schema/context/spring-context.xsd
         http://www.springframework.org/schema/mvc http://www.springframework.org/schema/mvc/spring-mvc.xsdd">

    <context:property-placeholder location="classpath:jdbc.properties"/>
    <context:component-scan base-package="com.xxx"/>
    <mvc:annotation-driven/>

    <bean id="dataSource" class="org.apache.commons.dbcp2.BasicDataSource" destroy-method="close">
		<property name="driverClassName" value="${db.mysql.driver}"/>
		<property name="url" value="${db.mysql.url}"/>
		<property name="username" value="${db.mysql.username}"/>
		<property name="password" value="${db.mysql.password}"/>
	</bean>
```
`jdbc.properties`内容为：
```
db.mysql.driver=com.mysql.jdbc.Driver
db.mysql.url=jdbc:mysql://localhost:3306/db
db.mysql.username=username
db.mysql.password=password
```
下面我们用代码的方式来实现上述配置文件：
```java
@Configuration
@PropertySource("classpath:jdbc.properties")
@ComponentScan(basePackages="com.xxx")
public class MySpringConfig {

	@Value("${db.mysql.url}")
	private String url;

	@Value("${db.mysql.driver}")
	private String driver;

	@Value("${db.mysql.username}")
	private String username;

	@Value("${db.mysql.password}")
	private String password;

	@Bean(destroyMethod="close")
	public DataSource getDataSource() {
		BasicDataSource dataSource = new BasicDataSource();
		dataSource.setUrl(url);
		dataSource.setDriverClassName(driver);
		dataSource.setUsername(username);
		dataSource.setPassword(password);
		return dataSource;
	}
}
```
代码中涉及到了Spring framework部分注解，若想详细了解请大家自行baidu或google之。那么到这里我们完成了用代码表示`dispatcher-servlet.xml`文件，但是要如何告诉Spring容器呢？

此时，我们需要修改之前的`MyWebApplicationInitializer`类中的`onStartup`方法，如下：
```java
@Override
	public void onStartup(ServletContext servletContext) throws ServletException {
		AnnotationConfigWebApplicationContext wc = new AnnotationConfigWebApplicationContext();
		wc.register(MySpringConfig.class);
		ServletRegistration.Dynamic dispatcher = servletContext.addServlet("dispatcher", new DispatcherServlet(wc));
		dispatcher.setLoadOnStartup(1);
		dispatcher.addMapping("/");
	}
```
新建`AnnotationConfigWebApplicationContext`并注册我们的`MySpringConfig`类，再将该context传给`DispatcherServlet`。

至此，我们已经成功的去除了`web.xml`和`dispatcher-servlet.xml`配置文件。
