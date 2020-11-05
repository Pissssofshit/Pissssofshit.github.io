---
title: go-blog系列(1) restful风格接口设计
date: 2020-11-04 23:25:19
tags:
---

1.  [restful 风格的几大准则](#orgb544a95)
2.  [给我们的博客设计接口](#org0aacddb)
3.  [代码](#orgf12b47a)

## restful 风格的几大准则

1.  以json格式接受和返回请求
2.  使用名词结尾
    解释:http请求方法已经有POST、DELETE、PUT、GET能分别对应增、删、改、查
3.  名词用复数形式
    解释:请求对应的是服务器的一种资源,资源往往都是复数的。如我们的博客文章表,也是命名为articles,而不是article
4.  使用层级区分资源
    解释:如我们想要获取评论,应当使用下面这样的形式

> '*articles*:articleId/comments'

​	因为comments是articles的子级资源

5. 接口应当接受筛选参数、排序和分页

6. 标准化响应请求
   常用的返回码如下:

* 400 Bad Request – This means that client-side input fails validation.
* 401 Unauthorized – This means the user isn’t not authorized to access a resource. It usually returns when the user isn’t authenticated.
* 403 Forbidden – This means the user is authenticated, but it’s not allowed to access a resource.
* 404 Not Found – This indicates that a resource is not found.
* 500 Internal server error – This is a generic server error. It probably shouldn’t be thrown explicitly.
* 502 Bad Gateway – This indicates an invalid response from an upstream server.
* 503 Service Unavailable – This indicates that something unexpected happened on server side (It can be anything like server overload, some parts of the system failed, etc.).

7. 给接口加上版本号
   解释:如下形式

> /v1/employees

资料:
[rest api 博客](https://stackoverflow.blog/2020/03/02/best-practices-for-rest-api-design/)




## 给我们的博客设计接口

<table border="2" cellspacing="0" cellpadding="6" rules="groups" frame="hsides">



<colgroup>
<col  class="org-left" />

<col  class="org-left" />

<col  class="org-left" />
</colgroup>
<tbody>
<tr>
<td class="org-left">功能</td>
<td class="org-left">HTTP方法</td>
<td class="org-left">路径</td>
</tr>


<tr>
<td class="org-left">用户注册</td>
<td class="org-left">POST</td>
<td class="org-left">/users</td>
</tr>


<tr>
<td class="org-left">用户登录</td>
<td class="org-left">GET</td>
<td class="org-left">/tokens</td>
</tr>


<tr>
<td class="org-left">用户更新</td>
<td class="org-left">PUT</td>
<td class="org-left">*users*:user<sub>id</sub></td>
</tr>


<tr>
<td class="org-left">用户删除</td>
<td class="org-left">DELETE</td>
<td class="org-left">*users*:user<sub>id</sub></td>
</tr>


<tr>
<td class="org-left">发表文章</td>
<td class="org-left">POST</td>
<td class="org-left">*users*:user<sub>id</sub>/articles</td>
</tr>


<tr>
<td class="org-left">编辑文章</td>
<td class="org-left">PUT</td>
<td class="org-left">*users*:user<sub>id</sub>/articles/:article<sub>id</sub></td>
</tr>


<tr>
<td class="org-left">删除文章</td>
<td class="org-left">DELETE</td>
<td class="org-left">*users*:user<sub>id</sub>/articles/:article<sub>id</sub></td>
</tr>
</tbody>
</table>

解释一下用户登录为什么是/tokens;restful风格的接口将请求认为是对服务器资源的访问，而登录实质上是获取token资源的一个过程


<a id="orgf12b47a"></a>

## 代码

在项目中新建api/v1目录并新建文件UserService.go,然后写入如下代码

    type UserService struct {
    }
    
    func NewUserService() UserService {
    	return UserService{}
    }
    
    func (userService *UserService) Create(c *gin.Context) {
    
    }
    func (userService *UserService) Login(c *gin.Context) {
    
    }
    func (userService *UserService) Edit(c *gin.Context) {
    
    }
    
    func (userService *UserService) Delete(c *gin.Context) {
    
    }
    func (userService *UserService) PostArticles(c *gin.Context) {
    
    }
    
    func (userService *UserService) EditArticles(c *gin.Context) {
    
    }
    func (userService *UserService) DeleteArticles(c *gin.Context) {
    
    }

在项目根目录新建main.go,
在main.go中写入如下代码

    router := gin.Default()
    
    //版本号v1
    v1 := router.Group("/v1")
    {
    	userService := service.NewUserService()
    	v1.POST("/users", userService.Create)
    	v1.GET("/tokens", userService.Login)
    	v1.PUT("/users/:user_id", userService.Edit)
    	v1.DELETE("/users/:user_id", userService.Delete)
    	v1.POST("/users/:user_id/articles", userService.PostArticles)
    	v1.PUT("/users/:user_id/articles/:article_id", userService.EditArticles)
    	v1.DELETE("/users/:user_id/articles/:article_id", userService.DeleteArticles)
    }
    
    router.Run(":8080")

完整代码查看我的github项目

    git clone git clone https://github.com/Pissssofshit/go-blog

,并检出分支

    git checkout restfulapi设计

在项目下运行

    go run main.go

项目就会运行在8080端口下

