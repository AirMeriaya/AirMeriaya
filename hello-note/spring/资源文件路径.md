# 关于Spring框架中资源路径问题
## 1. classpath和classpath*的区别：
    classpath：只搜索当前工程classpath下的配置文件
    classpath*：不仅搜索当前工程classpath，还会搜索其引用的jar下符合条件的配置文件。
    
## 2. 斜杠“/”：
    斜杠“/”在spring中代表工程根目录，相当于System.getProperty("user.dir")
