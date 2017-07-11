__Spring加载properties可以大致分为两种方式：环境变量，包装类__

## 一、环境变量
加载方式:
``` xml
<context:property-placeholder location="classpath:xxx.properties"/>
```
或
``` xml
<bean id="configProperties" class="org.springframework.beans.factory.config.PropertyPlaceholderConfigurer">
    <property name="ignoreResourceNotFound" value="true"/>
    <property name="locations">
    <list>
        <value>classpath:xxx.properties</value>
    </list>
    </property>
</bean>
```
这种方式下，我们可以用__${key}__的形式在xml配置文件中或者在字段上使用__@Value__读取properties文件中的内容。
<br>
## 二、包装类
加载方式：
``` xml
<util:properties id="prop" location="classpath:xxx.properties"/>
<!-- 需要添加util的声明 -->
xmlns:util="http://www.springframework.org/schema/util"
xsi:schemaLocation="http://www.springframework.org/schema/util http://www.springframework.org/schema/util/spring-util.xsd"
```
或
``` xml
<bean id="prop" class="org.springframework.beans.factory.config.PropertiesFactoryBean">
    <property name="ignoreResourceNotFound" value="true"/>
    <property name="locations">
    <list>
        <value>classpath:/xxx.properties</value>
    </list>
    </property>
</bean>
```
这种方式下，我们可以用#{prop['key']}读取在xml配置文件中读取properties文件中的内容。
