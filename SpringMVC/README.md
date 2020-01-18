##### 1.SpringMVC执行流程？
		初始化阶段，DispatcherServlet父类HttpServletBean调用initServletBean()方法，加载web.xml文件，
	实际上是加载init-param里面的参数，获得springMVC配置文件的路径，然后把内容加载道PathVriable中，而后
	DispatcherServlet类中的静态代码块完成HandlerMappings初始化（HandlerMappings就是一个请求路径与对应
	HandlerExcutionChain执行链的映射关系，而HandlerExcutionChain是什么呢？它是一个Handler的封装类，
	其中封装了一个Handler，这个Handler可以是一个类，也可以是一个方法，取决于这个Handler的注册方式。再加上
	该Handler对应的一个拦截器数组，组成了HandlerExcutionChain的属性）。这样，初始化工作就基本完成了。
		客户端发送请求道前端控制器DispatcherServlet。
		DispatcherServle收到请求后，最终会执行doDispatcher(),首先判断该请求是否带有二进制参数（是否
	有文件传输）然后再遍历初始化好的HandlerMappings，以该次请求的HttpServletRequest为参数，获得与该请求
	对应HandlerExcutionChain。然后再获得对应的HandlerAdapter（适配器，用来执行Handler）。然后遍历
	HandlerExcutionChain里面的拦截器数组，执行其中的前置拦截器，然后再调用适配器的handle方法在执行对应
	的handler，而这也就是我们写的业务逻辑，这个方法执行完后，返回ModelAndView对象给DispatcherServlet，
	然后DispatcherServle再把ModelAndView传给ViewResolver视图解析器进行解析。
		ViewResolver解析后返回具体的View。
		DispatcherServle对View进行渲染，然后响应用户。


##### 2.SpringMVC的主要组件。
		无需开发人员开发：
		前端控制器DispatcherServle：接受请求，响应结果，视图渲染。DispatcherServle相当于一个转发器，
	并减少了其他组件之间的耦合度。
		映射器HandlerMappings：根据请求的URL来查找对应的Handler（准确来说是HandlerExcutionChain）。
		处理器适配器HandlerAdapter：适配器就是真正用来执行Handler的组件，它的handler方法就是通过反射来
	执行我们书写的业务逻辑，并且会返回一个ModelAndView对象。
		视图解析器ViewResolver：接受从DispatcherServle传过来的ModelAndView对象，进行视图解析后得到
	View，再返回给DispatcherServlet进行渲染。、
	
		需要开发人员开发：
		Handler：其实就是Controller，没啥好说的。
		View：它是一个借口，实现类支持比如说像jsp,freemarker这样的类型。

##### 3.如何解决POST请求中文乱码问题？GET请求呢？
		POST请求中文乱码可以在web.xml中配置编码过滤器CharacterEncodingFilter，强制将参数转化为utf-8，
	就不会出现乱码了。
		而GET请求乱码用两种解决方式：1.修改tomcat配置文件，使其按utf-8来编码  2.在后端进行重新编码处理：
	String userName=new String(request.getParamter("userName").getBytes("ISO8859-1"),"utf-8")

##### 4.SpringMVC异常处理
	常用的有三种：
		1.在配置文件中进行配置。注册SimpleMappingExceptionResolver类，可以对Controller中出现的各种异常
	都可以进行一个错误界面的映射配置。但是这种方法比较冗长，需要写不少xml代码。
		2.在要进行异常处理的方法上加上@ExceptionHandler注解，注解参数写的是要进行处理的异常的class，如：
	@ExceptionHandler(MyException.class)。这样当前类抛出该异常时，该方法就会进行捕获。但缺点是在该类之外
	的方法如果抛出了此异常，就无法进行处理。当然也有解决方法，就是把所有异常处理的方写在一个类上，然后让所有的
	Controller都继承该类，那么Controller中所抛出的异常就可以被处理了。但是这样使代码耦合度变高，在代码结构
	上来看，并不是好的方法。
		3.写一个异常处理类，实现HandlerExceptionResolver接口，实现其resolveException方法，然后在
	springmvc中的配置文件中注册该类。然后抛出异常时，该方法就能捕获到并进行对应的逻辑处理。
	
参考: [spring mvc异常处理的三种方式](http://www.monkey1024.com/framework/1291).


##### 5.SpringMVC返回类型
	1.String：如果使用String，则返回的是视图的名字。
	2.ModelAndView：返回视图对象，同时还可以使用它的setAttribute方法来发送一些对象。
	3.Object:：前后端进行数据交互的时候，一般使用Object返回类型，并配合@Response注解可以返回json格式。
	4.Map：用Map作为返回类型在前后端使用ajax进行交互的场景下比较常见。
	
	
		
	
