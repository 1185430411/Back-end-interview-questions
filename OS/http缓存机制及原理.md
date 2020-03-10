@[toc]

# HTTP报文

HTTP报文分为两个部分

- Header：报头，填写着相关一些字段属性，与HTTP缓存相关的规则就保存在Header中
- Body：用来存放HTTP传输的真正数据

# HTTP缓存

HTTP缓存涉及到三个主主体：客户端浏览器，缓存数据库和服务端。而HTTP缓存又分为两种：

- **强缓存**
- **协商缓存**

## 强缓存

其基本思想就是：当数据不存在于缓存数据库时，此时会直接去请求服务器，并把得到的结果写入缓存数据库中；若数据存在缓存数据库且未过时，则直接去缓存数据库中查询，完成这次请求。

流程如下：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200310220013712.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)

![在这里插入图片描述](https://img-blog.csdnimg.cn/20200310220037340.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)
还是比较容易理解。另外需要注意的是：请求服务器资源成功的响应码是200，而从缓存数据库中请求成功的响应码是304。
和强缓存相关的首部字段有三个：
1.Expire：规定缓存过期的绝对时间，由于客户端时间和服务端时间存在偏差，所以在HTTP1.1之后的版本一般来说都不会使用这个字段了
2.Cache-Control：与缓存相关字段，常用的几个取值有：

- private：客户端可以缓存
- public：客户端和代理服务器都可以缓存
- max-age：缓存失效的一个相对时间，单位为秒
- must-revalidate：规定了缓存在使用之前必须验证旧资源的状态，并且不可以使用过期数据

这是强缓存的时候，那当缓存失效之后怎么办？此时就要进行协商缓存了

## 协商缓存

当请求的值在缓存数据库中失效，或者首部字段中明确要求每次读取缓存后都要去服务端进行一次验证，并且不能使用过期数据时，使用协商缓存
协商缓存的机制其实也很简单：查看缓存数据库中对应资源缓存是否已经过期，如果是的话，在使用缓存数据之前还要向服务端发送一次HEAD请求，主要用来检验对应数据是否已经被更新过，如果没有的话，则可以直接使用缓存数据库中的过期值，如果已经被更新过，那么就要从服务端中重新请求新值。
检验是否过期的方法很简单：
在第一次从服务端获取资源时，响应报文首部有一个last-modified字段，用于记录这个资源的最近一次更新时间，那在发送HEAD请求确认的时候，会写入一个if-modified-since字段，这个字段的值就是接受到的last-modified。在服务端中检查这个if-modified-since值和last-modified值是否相等，如果相等说明没有会被更新，可以放心使用缓存数据库的值，如果不相等，说明已经被更新过了，此时从服务端请求新值。

总结就是：
![在这里插入图片描述](https://img-blog.csdnimg.cn/20200310222400209.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L1NDVVRKQVk=,size_16,color_FFFFFF,t_70)
