## RESTful概念与最佳实践

作者：[EaconTang](https://github.com/EaconTang) | 文章出处：[博客链接](http://blog.tangyingkang.com/post/2017/02/20/restful-theory-n-best-practise/)  

----


### RESTful概念
__REST(Representational State Transfer)__，意即__资源的表现层状态转换__，起源于Roy Felding在他的论文[《基于网络的软件架构》](http://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)。具体来讲：

- __资源（Resource）__
    + 资源代表网络上的一个实体，比如文本、图片、视频和音乐等，每个资源可以通过特定的URI来访问
    + 一般用于HTTP的请求地址
- __表现层（Representation）__
    + 上面的资源，可以有不同的表现形式，例如同一个文本，可以是TXT、XML、JSON、HTML和二进制格式
    + 一般在HTTP请求头中通过字段Accept和Content-Type来指定
- __状态转换（State Transfer）__
    + 客户端使服务器资源发生变化，这个就是对应HTTP协议的操作动词了：GET/POST/PUT/DELETE/PATCH...

其核心理念是将后端功能拆分为资源提供服务，通过HTTP行为(GET/POST/PUT/DELETE)进行CRUD操作  
符合RESTful风格的API就好比是后端服务的UI，应该友好、简洁、优雅。  

下面总结一些RESTful架构的最佳实践：

### API设计风格
1. __名词优于动词__
    - URI中不应该出现动词，如修改用户信息，不是设计为```/modify/user?id=01```，而应该是通过POST操作访问```/users```，并将修改信息放到请求body中

2. __为了避免混淆使用名词单数和复数，建议统一为复数__

3. __统一命名风格__
    - Java驼峰风格
    - Python蛇形（下划线）风格

4. __GET操作用于只读资源__
    - 任何状态改变应该使用POST/PATCH/PUT/DELETE这些操作

5. __声明序列化格式__
    - 在HTTP头部通过Accept和Content-type字段来指定通讯内容格式
    - 建议使用JSON格式，比XML更加简单易读

6. __通过URL参数提供过滤／排序／分页／选择字段等功能，例如：__
    - 过滤：```GET /users?sex=male```，筛选男性用户
    - 排序：```GET /users?sort=age```，按照年龄进行排序
    - 分页：```GET /users?offset=5&limit=-1```，通过offset和limit来截取、分页，offset=5表示从第5个用户起，limit=-1一般表示获取所有用户，不限制数量
    - 字段选择：GET /users?fields=name,sex,age，返回用户的name,sex,age这几个字段

7. __版本控制__
    - 版本究竟应该放倒URL还是Header，这个业界尚有争论（[《版本信息应该放进URL中还是Header？》](http://stackoverflow.com/questions/389169/best-practices-for-API-versioning)），我个人认为放URL也无妨
    - 例如：```GET /api/v1/users```，```GET /api/v2/users```

8. __使用SSL__
    - 尽量使用SSL，如OAuth协议；但记住，不要让非SSL的url访问重定向到SSL的url

9. __准确的HTTP状态码和详细信息__
    - API返回码应该尽可能准确，常用的状态码有如下：
        + 100 Continue：服务器仅接收到部分请求，但是一旦服务器并没有拒绝该请求，客户端应该继续发送其余的请求
        + 200 OK：请求成功
        + 201 Created：请求被创建完成，同时新的资源被创建
        + 202 Accepted：请求已被接受，但是处理未完成
        + 206 Partial Content：客户端发送了一个带有Range头的GET请求，服务器完成了它
        + 301 Moved Permanently：所请求的页面已经转移至新的url
        + 304 Not Modified：服务器告诉客户，原来缓冲的文档还可以继续使用。
        + 400 Bad Requests：无效请求，服务器不能理解
        + 401 Unauthorized：用户未认证
        + 403 Forbidden：禁止访问
        + 404 Not Found：无法找到被请求的页面
        + 405 Method Not Allowed：请求中指定的方法不被允许
        + 500 Internal Server Error：服务器内部错误
        + 501 Not Implemented：不支持所请求的功能
        + 502 Bad Gateway：服务器从上游服务器收到一个无效的响应
        + 503 Service Unavailable：服务器不可用
        + 505 HTTP Version Not Supported：不支持请求中指明的HTTP协议版本

10. __创建或更新操作应该返回新资源url__
    - 防止用户多次调用API

11. __默认使用pretty print__
    - 增加返回结果的可读性

12. __使用gzip压缩请求结果__

13. __完善API文档__
