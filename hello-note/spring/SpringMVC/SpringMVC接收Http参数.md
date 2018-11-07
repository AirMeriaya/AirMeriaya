### HttpRequest

> 想知道SpringMVC如何做到自动绑定，就要先知道Http请求是什么样的

#### 1. HttpRequest类别

* **按Http协议格式分**
  * Http Header
  * Http Body

* **按HttpMethod分**
  * 可以带有body **（POST, DELETE, UPDATE）**
  * 不可以带有body **（GET）**
* **按Content-Type分**
  * application/x-www-form-urlencoded **（default）**
  * application/json、application/xml
  * multipart/form-data
  * application/octet-stream
  * other

#### 2. 参数形式

根据以上分类，并结合Postman提供的参数设置详细分析下各种`HttpRequest`的形式

通过Postman的界面，我们可以得知，参数的设置一共可以有6种：

##### 1. Http Header

​	写在Http Header中的参数，K: V结构，注意要避免使用协议中的常用字段

##### 2. Query String (Params)

​	在URL中问号“?”后面的参数，K=V结构，参数之间使用“&”连接

##### 3. form-data

​	`Content-Type: multipart/form-data`，使用boundary将表单中各个字段分割成独立的part，每个part都有一个`Content-Disposition`记录该字段的信息，对于文件还有`Content-Type`表示该字段的类型

​	form-data算是以boundary为分隔符的`List<Part>`结构，每个part算是一个K=V结构。可以上传多文件，或文件键值混合等等

> 此时Content-Type值为：multipart/form-data;boundary={随机字符串}
>
> 在body中，以--{boundary}\r\n为分隔符

##### 4. x-www-form-urlencoded

​	普通form表单，可以理解为是在body中的Query String，K-V结构，参数之间使用“&”连接。在Java servlet中，与Query String一起组成了`HttpRequest.parameters`部分

##### 5. raw (Plain Text、JSON、XML)

​	直译：裸类型；可以理解为纯文本类型。随着`Content-Type`的不同，文本内容可以是JSON、XML等

##### 6. binary

​	`Content-Type: application/octet-stream`二进制/流，一般用于上传文件。由于不是K-V结构，因此该形式只能上传单文件。



### RequestMapping中的参数

在SpringMVC中，提供了很多注解帮助开发者指定请求与方法间参数的对应关系。而在Java方法中，参数可以是基本类型（Primitive），可以是对象（Object）。



##### 对应关系如下表

| 注解类型       | Primitive Type                                               | Object                                                       |
| -------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 无注解         | Query String<br/>x-www-form-urlencoded<br/>请求参数的Key与方法中参数名匹配 | Query String<br/>x-www-form-urlencoded<br/>请求参数的Key与对象成员变量名匹配 |
| @RequestHeader | Http Header<br />若在注解中指定变量名则匹配变量名值，反之匹配方法中参数名 | 无效                                                         |
| @PathVariable  | 在URL中指定变量名<br />若在注解中指定变量名则匹配变量名值，反之匹配方法中参数名 | 无效                                                         |
| @RequestParam  | Query String<br/>x-www-form-urlencoded<br/>若在注解中指定变量名则匹配变量名值，反之匹配方法中参数名 | Query String<br/>x-www-form-urlencoded<br/>若在注解中指定变量名则匹配变量名值，反之匹配方法中参数名 |
| @RequestBody   | 若为x-www-form-urlencoded，则将其视为字符串，反之无效<br/>该注解在参数中有且只有一个 | application/json<br/>application/xml<br/>反序列化为对象      |
| @RequestPart   | multipart/form-data<br/>提取part，并按@RequestParam匹配      | multipart/form-data<br/>提取part，并按@RequestParam匹配      |

*补充 1*：`@RequestParam`也可以解析`multipart/form-data`，但需要额外配置`HttpMessageConverter`，有两种方式：

* `StandardServletMultipartResolver` + `Servlet 3.0+`
* `CommonsMultipartResolver` + `commons-fileupload`

*补充 2*：强烈建议在能指定变量名的注解中指定值。当注解中未指定值时，SpringMVC会根据参数名进行匹配，虽然便捷了开发，但在第一次调用时，SpringMVC必须利用`asm、javassist`等第三方jar包通过分析字节码中本地变量表获取参数名，并缓存到内存中，既降低效率又浪费资源