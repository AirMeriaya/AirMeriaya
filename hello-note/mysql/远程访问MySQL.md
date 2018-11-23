### 1. 如何配置

使用管理员账号登录MySQL，修改mysql.user表

```mysql
UPDATE mysql.user SET host='%' WHERE user='root';
```

此时就可以在其他机器上使用root账号登录了

如果想添加其他可以远程登录的用户，可以在创建用户的时候指定其host为%

``` mysql
CREATE USER 'username'@'%' IDENTIFIED BY 'password';
```

---

### 2. 遇到的问题

  - A机器装有MySQL服务且添加可远程访问的用户
  - B机器装有MySQL客户端且测试可以链接成功
  - 在B机器上编写Java代码用jdbc访问A机器的MySQL时，报如下错（略掉调用栈）
```
Exception in thread "main" com.mysql.jdbc.exceptions.jdbc4.CommunicationsException: Communications link failure

The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
	......
Caused by: java.net.SocketException: Permission denied: connect
	at java.net.DualStackPlainSocketImpl.connect0(Native Method)
	......
```
google一番，N多种办法，但都不能解决
最后试了试添加JVM参数【-Djava.net.preferIPv4Stack=true】，访问成功！
`当时最不看好的一个解决方案，原来就是我要找的=。=`

**PS:** 该参数告诉JVM对外连接时，优先使用IPv4地址

---

### 3. 总结

能否远程访问MySQL，不考虑密码问题的话，取决于两个因素

* MySQL服务启动时端绑定的地址
* 客户端所在机器的地址

对于绑定地址，MySQL配置文件或启动命令行中有这样一个参数`bind-address`，服务端启动后会监听配置中地址，例如：

​	`127.0.0.1，服务端监听本地回环地址，因此客户端只能是本地`

​	`0.0.0.0，IPv4地址，只要客户端通过IPv4地址就可以连接成功，注意，并没有说登录成功，因为还要看第二个因素`

​	`::，IPv6地址，同IPv4要求`

​	`通配符*，IPv4和IPv6`

此外，还支持以逗号“,”分割的多地址配置

在明确了服务端绑定地址后，再来看客户端地址。

在文章一开始，我们修改root用户的host字段为%来实现远程访问，这里的host就代表了客户端所使用的地址，而%代表了所有，也就是说，root用户可以用任意的IP来进行访问

那么既然是任意IP了，为什么还会遇到上面那个问题呢？因为我服务端绑定的地址是0.0.0.0，而我用Java写的程序连接时使用了IPv6