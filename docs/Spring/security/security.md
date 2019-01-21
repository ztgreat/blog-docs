## 前言

这是一篇关于权限方面的文章，这是去年的时候结合一些简单的项目，自己抽空瞎折腾的学习demo，进行了一定的整理，涉及的内容包括

>spring boot 2.x，shiro，spring security，oauth2

公布出来的目的？？，方便爱学习的你（**骗赞**）。

本文也不得介绍这些项目流程，也不会讲解怎么实现的，毕竟都是套路，难度不大，自己debug 调试，也算是能力的提高。

每个项目中很多都还没有完善（**各种优化都没加**），**只能学习参考**，至于有些自己是如何整合的？ 看源码

没错，权限这个东西，要验证很多东西，不明白的时候，只有跟踪流程，了解源码，然后再搜索相关知识，这样也算是自己的能力的一种提高吧，捡现成固然容易，但是没有解决问题的能力，这就很麻烦了。

## springboot-shiro

基于spring boot,**shiro**,mybatis(使用mybatis-plus)的简易的一个后台权限模块,只包含权限部分

前端展示配合 [react-authority](https://github.com/ztgreat/react-authority)

此外 另一个权限[springboot-security](https://github.com/ztgreat/springboot-security) 项目使用spring security 重写了权限部分，从功能上来说是一致的

### shiro

权限部分是通过shiro 来实现的，基于资源的访问控制（实质也是角色访问控制），根据请求资源，判断用户是否拥有该资源，进而判断是否允许用户访问，前端使用ant-design-pro来简单的实现，**并没有很完善化**，可以具体到某个action,展示界面**未到按钮级**别进行控制。

#### 菜单管理

![菜单管理](http://img.blog.ztgreat.cn/document/security/20190112182753.png)

#### 资源权限

![资源权限](http://img.blog.ztgreat.cn/document/security/20190112182754.png)



#### 角色管理

![角色管理](http://img.blog.ztgreat.cn/document/security/20190112182755.png)

可以给每个角色分配菜单和资源

#### 角色分配

![角色分配](http://img.blog.ztgreat.cn/document/security/20190112182756.png)



![角色分配2](http://img.blog.ztgreat.cn/document/security/20190112182757.png)

### session

将用户session 已json字符串的方式 放入redis 中，**shiro session 没法直接json序列化到redis**，项目中做了一些调整和修改

![session](http://img.blog.ztgreat.cn/document/security/session.png)



## springboot-security

基于spring boot,**spring-security**,mybatis(使用mybatis-plus)的简易的一个后台权限模块,只包含权限部分

前端展示配合 [react-authority](https://github.com/ztgreat/react-authority)

此外 另一个权限[springboot-shiro](https://github.com/ztgreat/springboot-shiro) 项目使用shiro 重写了权限部分，从功能上来说是一致的

### spring security

基于资源的访问控制（实质也是角色访问控制），根据请求资源，判断用户是否拥有该资源，进而判断是否允许用户访问，前端使用ant-design-pro来简单的实现，并没有很完善化，可以具体到某个action,展示界面未到按钮级别进行控制。
url 资源判断**复用了shiro 中的模块**（WildcardPermission）,具体代码中有注释。

## springboot-security-oauth2

前端展示请配合 [react-authority](https://github.com/ztgreat/react-authority)

在[springboot-security](https://github.com/ztgreat/springboot-security) 项目基础上,融合了部分**oauth2**的部分功能。

### Oauth2-authorizedGrantType-client

在Spring security的基础上集成**Oauth2的客户端认证模式**,原权限部分不受影响。

基础权限 功能请访问[springboot-security](https://github.com/ztgreat/springboot-security)

#### 未登录访问

![未登陆访问](http://img.blog.ztgreat.cn/document/security/20190112182758.png)

#### 获取Access_Token

http://localhost:8080/oauth/token?grant_type=client_credentials&client_id=client_1&client_secret=123456

![获取Access_Token](http://img.blog.ztgreat.cn/document/security/20190112182759.png)

#### 通过Access_Token 访问

![通过Access_Token访问](http://img.blog.ztgreat.cn/document/security/20190112182760.png)

#### 登录

![登录](http://img.blog.ztgreat.cn/document/security/20190112182761.png)



#### 登录后访问

![登录后通过cookie访问](http://img.blog.ztgreat.cn/document/security/20190112182762.png)

#### 注意

不要在同一台电脑上同时测试登录授权和通过Access_Token授权访问，否则授权信息会被覆盖。

### Oauth2-authorizedGrantType-code

在Spring security的基础上集成**Oauth2的授权码认证模式**,原权限部分不受影响 

基础权限 项目：[springboot-security](https://github.com/ztgreat/springboot-security)

配合前端展示：[react-authority](https://github.com/ztgreat/react-authority)

#### 未登录获取授权码

http://localhost:8080/oauth/authorize?response_type=code&client_id=client_2&client_secret=123456&redirect_uri=http://baidu.com

![未登录获取授权码](http://img.blog.ztgreat.cn/document/security/20190112182763.png)



#### 前端登录

账号：admin

密码：000000

![前端登录](http://img.blog.ztgreat.cn/document/security/20190112182764.png)



注意：这是前后端分离的前端页面

#### 授权页面

![授权页面](http://img.blog.ztgreat.cn/document/security/20190112182765.png)



#### 获取授权码

![获取授权码](http://img.blog.ztgreat.cn/document/security/20190112182766.png)



地址栏 code 便是授权码

#### 通过授权码获取token

http://localhost:8080/oauth/token?client_id=client_2&client_secret=123456&grant_type=authorization_code&redirect_uri=http://baidu.com&code=A3dv5E

![获取token](http://img.blog.ztgreat.cn/document/security/20190112182767.png)

#### 通过token 获取数据

![获取数据](http://img.blog.ztgreat.cn/document/security/20190112182768.png)



> error 页面，以及授权页面都没有重写（实在不想写前端页面了），只是简单的把功能过了一遍，仅供参考，还有很多没有完善，基本上弄懂这些后，扩展我觉得问题都不大，有了渔害怕没鱼嘛。
>
> 项目中都有相应的sql，方便测试
>
> 有问题，欢迎提出来，有些我也就简单的进行了测试。