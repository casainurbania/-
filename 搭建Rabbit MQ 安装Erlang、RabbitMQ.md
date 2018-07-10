# [搭建Rabbit MQ 安装Erlang、RabbitMQ](https://www.cnblogs.com/wangjiesheng/p/7728268.html)

我这边是在Winddows 下面安装的。
1、安装RabbitMQ需要先安装Erlang语言开发包。下载地址 http://www.erlang.org/download.html 我的百度云：链接：http://pan.baidu.com/s/1bZask2 密码：a7qy
配置环境变量
ERLANG_HOME
D:\Program Files\erl7.3
添加到PATH %ERLANG_HOME%\bin;

2、安装RabbitMQ 下载地址 http://www.rabbitmq.com/download.html 我的百度云：链接：http://pan.baidu.com/s/1qXFAT4o 密码：bu6s
配置环境变量
RABBITMQ_SERVER
D:\Program Files\RabbitMQ Server\rabbitmq_server-3.6.12
添加到PATH %RABBITMQ_SERVER%\sbin;

3、安装好上面了两个步骤下面进行配置
1、激活 RabbitMQ's Management Plugin
"D:\Program Files\RabbitMQ Server\rabbitmq_server-3.6.12\sbin\rabbitmq-plugins.bat" enable rabbitmq_management
2、这样，就安装好插件了，是不是能使用了呢？别急，需要重启服务才行，使用命令：
net stop RabbitMQ && net start RabbitMQ

![img](https://images2017.cnblogs.com/blog/931271/201710/931271-20171025125047660-2094819266.png)

安装参照文档：http://www.cnblogs.com/ericli-ericli/p/5902270.html. 里面会介绍创建用户权限等。

使用浏览器打开 http://localhost:15672 访问Rabbit Mq的管理控制台。默认账户有guest guest。

一个RabbitMq 算是搭建好了。