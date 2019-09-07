# JavaWeb



## Tomcat
<br>

`webapps ` 目录是用于存放 `web ` 项目

部署 `web` 项目的方式

1. 直接将项目放到 webapps 目录下即可
	* 访问方式 `localhost:8080/项目文件名/xxx`
	* 项目文件名就是虚拟目录
	* 可以将项目打成一个 `war` 包，然后放置到 `webapps` 目录下即可，`war` 会自动解压缩，如果要删除项目，只要删除 war包即可，项目会自动被移除
2. 修改 server.xml 文件
	* 在 `Host` 标签体中配置 `<Context docBase="项目路径" path="/虚拟目录名" />`
3. 	在 `Conf/Catalina/localhost` 目录下创建配置的 `虚拟目录名.xml` 文件
	* 在文件中添加 `<Context docBase="项目路径" />` 不需要 `path`

部署的项目的目录结构如下

```
-- 项目的根目录(web)
	-- WEB-INF 目录
		-- web.xml：web项目的核心配置文件
		-- classes目录：放置字节码文件的目录
		-- lib目录：放置依赖的jar包
	-- 其他资源 xx.html xx.css xx.js xx.xx
```


## Serlvet
<br>
> Servelt 就是一个接口，定义了 Java 类被浏览器访问(tomcat识别)到的规则，创建一个 Servlet 工程，需要在 project Structure 中的 Libraries 中，将 tomcat 的目录的 lib 添加进来，才能找到 Servlet 接口

<br>

在 `web.xml` 中配置，Servlet 和 url 的映射关系

```
<servlet>
    <servlet-name>别名</servlet-name>
    // 第一次匹配到后，tomcat会将其字节码文件加载进JVM，通过 Class.forName(..) 的方式
    // 然后在通过反射创建对象 => 在调用接口的方法
    <servlet-class>全限定性类名</servlet-class> 
</servlet>
<servlet-mapping>
    <servlet-name>别名</servlet-name>
    <url-pattern>匹配的url</url-pattern>
</servlet-mapping>
```
<br>

`Servlet3.0` 开始支持**注解配置**，可以不在 `web.xml` 中定义 `Servlet` 和 `url` 的映射关系，而是使用 `@WebServlet` 注解。

`WEB-INF` **目录下**的资源不能被浏览器直接访问

继承体系 `Servlet => GenericServlet => HttpServlet`



## Request 和 Response
<br>


### Request

主要的请求头  User-Agent、Connection: keep-alive、Referer 告诉服务器，当前的请求从哪里来，用于防止盗链和统计工作

**API**

* getContextPath 重要
	* 虚拟名
	* 例如当部署了一个 `test` 的项目在 `webapps` 中，访问的时候是 `localhost:8080/test/path` 获取的就是 `/test`，一般在返回给 `html` 的 `a` 标签设置的 `href` 的时候，动态获取链接字符串
		* `req.getContextPath() + "/getPage"` 这个 `/getPage ` 就是在 `servlet `中配置的映射 `@WebServlet("/getPage")`
* getRequestURI 重要
	* 虚拟名 + 路径 	
* getQueryString、getMethod、getServletPath、getRemoteAddr
* getHeader、getHeaderNames
* 重要的其他功能
	* 获取请求(GET和POST)参数的通用方式  
		* getParameter
		* getParameterValues
		* getParameterNames
		* getParameterMap
	* 请求转发
		* RequestDispatcher getRequestDispatcher(path)
			* dispatcher.forward(req, resp) 进行转发
		* 特点
		   * 浏览器的地址路径不发生变化
		   * 只能转发到服务器内部的资源中，而不能转发到别的服务器
		   * 只发送一次请求 
	* 共享数据
		* request域：代表一次请求的范围，用于转发中的资源共享
			* setAttribute、getAtrribute、removeAttribute 
	* 获取 ServletContext
		* getServletContext


处理 `POST` 请求的乱码问题解决方式是：在获取参数前设置 `req.setCharacterEncoding("utf-8");`


### Response
<br>

常用的响应头 

* Content-Type: Application/json;charset=UTF-8
* Content-disposition 告诉客户端以书面格式打开响应体数据
	* 默认是 `in-line` 表示在当前页面打开
	* `attachment` 表示以附件形式打开响应体，例如文件下载

**API**

* setStatus 设置相应状态码
* setHeader 设置响应头
* 设置响应体
	* getWriter
	* getOutputStream
	* 注意事项就是，在给输出流写入数据的时候，记得设置字符编码和 `Content-Type`，因为 `tomcat` 默认是 `ISO-8859-1` 编码
		* resp.setCharacterEncoding("utf-8")
		* resp.setHeader("Content-Type", "text/html;charset=utf-8")
		* resp.getWriter().write("xxx") 

Response 的重定向方式

```
resp.setStatus(302)
resp.setHeader("location", "/context/path")
=> 简单的重定向方法
resp.sendRedirect("/context/path")
```

在转发、或者 html 中路径的编写规则  

* 给客户端浏览器使用：需要添加虚拟目录，`a` 标签
* 给服务器使用：不需要虚拟目录，转发



## ServletContent

代表整个 `web` 的应用，可以和程序的容器(服务器)来通信，通过 `request.getServletContext` 方法获取

作用

* 获取 MIME 类型
* 是一个域对象
* 获取文件的真实路径(服务器上的全路径名)
	* getRealPath 重要的方法，获取到的是在 web 目录下的资源
		* System.out.println(req.getServletContext().getRealPath("/index.jsp")); 
		* /Users/xxxx/Documents/java4app/servlet/out/artifacts/servlet_war_exploded/index.jsp  

		
**文件下载操作功能**

* 设置响应头 `content-Type: attachment;filename=xxx`
* 将内容写入到输出流中
在文件下载中，如果响应头的 `filename` 设置为中文的话，浏览器可能出现乱码问题，所以需要针对不同的浏览器对 `filename` 的值进行编码

```
public static getFileName(String agent, String filename) throw UnsupportedEncodingException {
    if (agent.contains("MSIE")) {
        filename = URLEncoder.encode(filename, "utf-8");
        filename = filename.replace("+", " ");
    } else if (agent.contains("Firefox")) {
        BASE64Encoder base64Encoder = new BASE64Encoder();
        filename = "=?utf-8?B? + base64Encoder.encode(filename.getBytes("utf-8")) + "?=";
    } else {
        filename = URLEncoder.encode(filename, "utf-8");
    }
}
```



## Cookie 和 Session
<br>

### Cookie

* new Cookie
	* 对中文和特殊字符，可能报错，需要使用 `urlencoder` 进行编码
* resp.addCookie
* req.getCookies
* 其他 api
	* setMaxAge
		* 如果不设置 `setMaxAge`，那么在浏览器关闭后，`cookie` 就会被清除
		* 默认是负数则关闭浏览器，就会清除
		* 0表示删除 `cookie` 信息
	* `setPath` 用于设置 `cookie` 的有效获取范围，如果不设置，默认是服务器的虚拟目录  
	* `setDomain(String path)` 如果设置的是一级域名，那么多个子域名之间的服务器可以共享，这时候需要 `setPath("/")`
		* `setDomain(".baidu.com")` 那么 `api.baidu.com` 和 `tieba.baidu.com` 的服务器都可以共享到 


### HttpSession

使用 `request.getSession` 获取 `session` 对象。
在第一次使用 `Session` 对象的时候，会自动设置一个 `cookie` 对象，内容为 `JSESSIONID=xxxxxxxx`; 当第二次方法的时候，如果客户端 `cookie` 就会携带了 `JSESSIONID` 的信息，那么就会根据 `id` 去获取到 `session` 对象。所以 `session` 的实现是依赖于 `cookie `的，需要从 `cookie` 中的 `JSESSIONID` 中获取到 `session` 数据的值

`session`默认的失效时间是 `30` 分钟，可以在 `conf` 的 `web.xml` 中查看

```
<session-config>
    <session-timeout>30</session-timeout>
</session-config>
```


## Filter
<br>

* 自定义类实现 Filter 接口
* 在项目中配置
	* web.xml 中配置
	* 也可以使用注解配置
		* 设置 DispatcherType 可以指定过滤的范围，例如是请求过滤，还是转发过滤。。



## Lintener
<br>

* ServletContextListener 接口，用于监听 ServletContext 对象的创建和销毁

```

public class ContextLoaderListener implements ServletContextListener {
    ///  ServletContext创建，会触发监听者的这个方法
    @Override
    public void contextInitialized(ServletContextEvent servletContextEvent) {
        
    }
    ///  ServletContext销毁，会触发监听者的这个方法
    @Override
    public void contextDestroyed(ServletContextEvent servletContextEvent) {

    }
}
```

使用步骤

* 自定义类实现接口
* `web.xml` 或者注解进行配置 

```
<listener>com.gz.web.listener.ContextLoaderListener</listener> 即可

// 因为 ServletContext 是 tomcat 内部创建的，所以如果我们想要设置一些数据的话，可以在 web.mxl 中，使用 context-param 标签进行配置。
<context-param>
    <param-name>contextConfigLocation</param-name>
    <param-value>/WEB-INF/classes/applicationContext.xml</param-value>
</context-param>
```

使用 `servletContext.getInitParameter("contextConfigLocation")` 可以获取到配置的值。

这个在 springmvc 中用于 ServletContext 创建的时候，指定 Spring Ioc 容器创建的配置信息。



















