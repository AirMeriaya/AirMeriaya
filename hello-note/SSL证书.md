## 写在前面

最近有朋友让我帮忙把他的web服务以HTTPS发布，第一想法就是去相应的web容器官网搜索如何配置。但被很多的概念和名词弄得云里雾里，比如X.509，PKCS#12，还有各种扩展名

所以决定花一些时间探究下这些东西到底是个啥



## SSL证书的原理和目的

> 这里不会展开讲解非对称加密，也没有涉及到证书颁发机构（CA），有兴趣的话可以百度或谷歌之

我个人理解，SSL证书原理是基于非对称加密算法的一种证明文件，目的是为验证当前通信双方的身份是否可信。非对称加密算法保证了传输内容的可靠性，而SSL证书包含了一些身份信息，用以让对方验证

RSA是使用最广泛的非对称加密算法，其原理就是利用大数质因数分解困难的特性。因此在使用RSA加密时，我们要产生两个不相等的大质数，用来生成密钥对，一个称作私钥，不可外传，一个称作公钥，可以分发给任何人。公钥可以解密用私钥加密的密文，私钥可以解密用公钥加密的密文而且还可以根据私钥得到公钥

当两个端点需要建立连接进行通信的时候，都要将各自的公钥分发给对方，并在传输过程中使用各自的私钥和对方的公钥加解密，保障内容不被劫持篡改

这里面会涉及到三个概念，

- 摘要：用散列算法对部分信息进行操作得到的内容，散列后的内容不可逆，常用算法有MD5、SHA256等等
- 加密：使用对方的公钥对信息进行加密
- 加签：也叫签名，使用自己的私钥对信息（一般为摘要）进行加密

拿A发送信息给B举例来说

1. A使用散列算法对身份信息操作得到摘要
2. A使用私钥对1中的摘要进行加密（即加签）
3. A用B的公钥对传输内容和加签后的内容进行加密
4. B收到请求信息
5. B使用私钥对请求信息进行解密，解密出明文内容和加签信息
6. B使用相同散列算法对明文中身份信息操作得到摘要
7. B使用A的私钥对加签信息进行解密得到摘要
8. B对比6、7中的摘要是否相等
9. 验签完成

说了这么多SSL的基础概念，接下来进入到这次的主题，证书中经常出现的概念和名词都有什么含义



## 证书标准

证书标准规定了证书中包含哪些内容，大致分为两种

- X.509：最基础的标准，可以认为是所有证书标准的规范，内容包含公钥、身份信息和签名信息
- PKCS系列：可以认为是X.509的实现。常见的有PKCS#7和PKCS#12，其中7包含的信息与X.509大致相同，主要用于数字签名。12定义了私钥，公钥，证书的打包格式，还会额外有密码进行保护其内容，主要用于证书分发（如U盾等）



## 证书编码格式

通过证书标准，可以确定一个文件中都包含哪些东西以及以什么样的形式打包，那我们要以什么样的形式编码到文件中呢。基本分为两种

- PEM：BASE64编码的纯文本格式，会以`-----BEGIN xxxx-----`开头，以`-----END xxx-----`结尾，中间为BASE64编码内容
- DER：二进制编码



## 常见扩展名

- .der：使用DER格式编码的证书，Java和Windows服务器偏向于使用这种编码格式
- .pem：使用PEM格式编码的证书，Apache和Unix/Linux服务器偏向于使用这种编码格式
- .cer：数字证书，常见于Windows系统,可能是PEM编码，也可能是DER编码,大多数应该是DER编码
- .crt：数字证书，常见于Unix/Linux系统,有可能是PEM编码，也有可能是DER编码,大多数应该是PEM编码
- .pfx：PKCS#12标准的证书文件，常见于Windows系统，DER编码并使用提取码进行加密，jetty和tomcat等Java web容器可直接使用
- .csr：非证书，证书签名请求文件，核心内容是发起请求一方的公钥
- .key：非证书，一般是公钥或私钥文件，可能是PEM编码，也可能是DER编码
- .jks：即Java Key Storage，Java专利证书格式，JDK中keytool默认生成的证书为此格式，与.pfx包含的内容大致相同，也有提取码进行加密处理，jetty和tomcat等Java web容器可直接使用

扩展名一方面是为了在文件名的可读性上做一个直观区分，另一方面是为了关联应用程序。因此理论上一个证书或者秘钥是可以使用自定义扩展名甚至是没有扩展名，比如使用`ssh-keygen -t rsa`默认生成的私钥/公钥，就是id_rsa/id_rsa.pub



## 证书生成和格式转换

当我们手上某一种证书格式不满足于应用需要的时候，是可以通过工具进行转换的。工具有很多，甚至还可以在线转换，但为了安全考虑，掌握一些转换工具是很有必要的。常用的具有OpenSSL和JDK中的keytool。关于工具的安装方法网上有很多，这里不再赘述。

### 1. 证书生成

**keytool**

```shell
> keytool -genkey -alias demo_jks -keyalg RSA -validity 365 -keystore demo.jks
输入密钥库口令:
再次输入新口令:
您的名字与姓氏是什么?
  [Unknown]:  demo.com
您的组织单位名称是什么?
  [Unknown]:  a
您的组织名称是什么?
  [Unknown]:
您所在的城市或区域名称是什么?
  [Unknown]:
您所在的省/市/自治区名称是什么?
  [Unknown]:
该单位的双字母国家/地区代码是什么?
  [Unknown]:
CN=demo.com, OU=a, O=Unknown, L=Unknown, ST=Unknown, C=Unknown是否正确?
  [否]:  y

输入 <dome_jks> 的密钥口令
        (如果和密钥库口令相同, 按回车):
```

这里面需要注意两点

- 口令：需要设置的口令有两个，一个是密钥库的，也就是jks文件的口令；另一个是当前名为demo_jks的密钥口令。
- 域名：在生成证书的过程中，需要填写一些身份信息，其中**名字与姓氏**指的就是域名。

**openssl**

```shell
> openssl genrsa -out private.key
> openssl req -new -x509 -key ssl_pri.key -out demo.cer -days 365
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:
Email Address []:
```

使用openssl，需要先生成私钥，再基于私钥生成证书。

这里只展示了自签证书的生成，对于需要CA签发的证书，需要在生成私钥后生成证书请求csr，CA签名后会返回给你真正的证书

```shell
> openssl req -new -key prvtkey.pem -out cert.csr
```



### 2. 格式转换

**PEM to DER**

```shell
openssl x509 -outform der -in input.pem -out output.der
```

**PEM to PFX**

```shell
openssl pkcs12 -export -inkey private.key -in input.crt -out output.pfx
```

**DER to PEM**

```shell
openssl x509 -inform der -in input.cer -out output.pem
```

**PFX to PEM**

```shell
openssl pkcs12 -in input.pfx -out output.cer -nodes
```

**PFX to JKS**

```shell
keytool -importkeystore -srckeystore  input.pfx -srcstoretype pkcs12 -destkeystore output.jks -deststoretype JKS
```

**JKS to PFX**

```shell
keytool -importkeystore -srckeystore  input.jks -srcstoretype JKS -destkeystore output.pfx -deststoretype pkcs12
```

**Extract Certification from PFX**

```shell
openssl pkcs12 -in input.pfx -nokeys -out output.cer
```

**Extract Private Key from PFX**

```shell
openssl pkcs12 -in input.pfx -nocerts -nodes -out private.key
```

**Extract Certification from JKS**、**Extract Private Key from JKS**

```
先转成PFX，之后操作同PFX
```



## 应用

> 这里只列出了核心配置项，更详细的配置项请参考官网手册

### 0. 准备

- openssl
- 使用keytool生成的my.pfx证书



### 1. Jetty

- 证书准备：将my.pfx拷贝到jetty_home/etc/

- 修改配置：修改jetty_home/start.ini

  ```properties
  --module=ssl
  jetty.ssl.host=0.0.0.0
  jetty.ssl.port=8443
  jetty.sslContext.keyStorePath=etc/my.pfx
  jetty.sslContext.trustStorePath=etc/my.pfx
  jetty.sslContext.keyStorePassword=OBF:19iy19j019j219j419j619j8
  jetty.sslContext.keyManagerPassword=OBF:19iy19j019j219j419j619j8
  jetty.sslContext.trustStorePassword=OBF:19iy19j019j219j419j619j8
  
  --module=https
  ```

*注：密码是通过`java -cp jetty-util-xxx.jar org.eclipse.jetty.util.security.Password 123456`加密的*



### 2. Nginx

- 证书准备：由于nginx不能直接使用pfx证书，因此需要使用openssl提取证书和私钥

  证书`openssl pkcs12 -in my.pfx -nodes -out nginx.pem`

  私钥`openssl pkcs12 -in my.pfx -nodes -nocerts -out nginx.key`

  拷贝nginx.pem和nginx.key到nginx_home/conf下

- 修改配置：修改nginx_home/conf/nginx.conf

  ```
  server {
          listen       8443 ssl;
          server_name  localhost;
          ssl_certificate      nginx.pem;
          ssl_certificate_key  nginx.key;
          .....;
          location / {
              root   html;
              index  index.html index.htm;
          }
      }
  ```



在配置完服务端的https后，由于使用的是自签证书，并不在浏览器受信证书列表中，因此在访问时会有警告。此时只需将我们的证书导入到浏览器受信证书列表即可
