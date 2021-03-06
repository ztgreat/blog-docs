## 前言

Netty是一个高性能、异步事件驱动的NIO框架，提供了对TCP、UDP和文件传输的支持，对于目前而言，只要互联网存在，那么网络IO 也将存在，就目前的形式而言，硬件越来越好，带宽也越来越大，这个时候IO的瓶颈就凸显了出来，个人觉得，掌握一门IO框架是很重要的，不能仅仅的学会做点应用层，对于中下层还是需要熟悉的，这样才能健全整个知识体系，越是上层的东西，更新换代也快，但是很多时候思想都是差不多的，万变不离其宗，稳扎稳大的弄好基础，这才是关键。

这个proxy 是LZ 基于Netty 闲暇时间写的，通过名字也知道这个是一个代理程序，其实网上有很多类似这样的程序，当然不仅仅局限于Java 语言，LZ是网络专业出身的，因此比较喜欢研究一下网络工具，有的时候很想让自己的内网主机暴露出去，可以通过外部访问，这样不仅方便调试程序，而且很容易控制。

这款 proxy 呢，可以通过公网服务器访问内网主机，目前仅支持tcp流量转发，是的，需要借助公网服务器，但是不需要把本机的服务部署在公网，这个还是会减少很多工作量的，而且大家也知道，云服务器的资源那是大大的宝贵，当然网上流行的内网穿透，不借助公网服务器，直接穿透内网，以LZ的网络知识，我只想说很难，而且这个在不同的网络环境下影响很大，极大可能性就是不稳定。

proxy 最开始的版本是通过端口转发的，但是这样就需要不同的服务绑定不同的端口，这无疑是一种资源浪费，同时需要开放公网服务器的端口，这样也不好，现在呢可以通过域名转发，同时也可以配合nginx来使用。

## 工作流程

![proxy](http://img.blog.ztgreat.cn/document/netty/proxy.png)



上面呢就是proxy的工作模式，很多的代理软件模式都差不多是这样的，代理服务器和代理客户端呢通过私有的协议进行通信。

代理客户端先和代理服务器建立连接，代理服务器通过不同的域名（或端口)来区分具体的代理服务，用户通过访问代理服务器的指定域名（或端口），然后代理服务器将数据转发给代理客户端，客户端再转发数据给真实服务器，当客户端接收到真实服务器响应后，再传输给代理服务器，代理服务器再将数据传送给用户，完成一次请求。

## example

（1）通过外网访问本机的mysql

```
- serverport: 3307
              proxyType: tcp
              realhost: 127.0.0.1
              realhostport: 3306
              description: mysql 代理
```

配置服务端，将服务端程序运行在公网服务器，同时开放3307端口，本地运行客户端（需要简要配置），这样通过外网ip或者域名访问其3307端口，便是访问**内网本机**的3306端口了。



![20180911152812](http://img.blog.ztgreat.cn/document/netty/20180911152812.png)



（2）ssh 服务

```
- serverport: 2222
              proxyType: tcp
              realhost: 172.16.254.63
              realhostport: 22
              description: ssh 代理
```

配置服务端，将服务端程序运行在公网服务器，同时开放2222端口，本地运行客户端（需要简要配置），这样通过外网ip或者域名访问其2222端口，便是访问**内网本机(172.16.254.63)**的22端口了。



![TIM截图20180911153744](http://img.blog.ztgreat.cn/document/netty/20180911153744.png)



(3)域名转发

```
- domain:  proxy.ztgreat.cn
           proxyType: http
           realhost: 127.0.0.1
           realhostport: 8080
           description: http代理
```

访问proxy.ztgreat.cn  实际访问的便是本机的8080端口

>通过域名转发，是共享的同一个端口（非80），这样需要入口配合nginx来使用

## 项目情况

proxy 整体很简单，只是可能有点繁琐，鉴于LZ水平还很菜，代码质量也就一般般，还有很多待完善，后面根据需求，不断完善吧，慢慢来。