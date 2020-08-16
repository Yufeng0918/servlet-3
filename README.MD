# Servlet 3



## 1. Servlet映射

@WebServlet映射路劲

```JAVA
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
	
	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		System.out.println(Thread.currentThread()+" start...");
		try {
			sayHello();
		} catch (Exception e) {
			e.printStackTrace();
		}
		resp.getWriter().write("hello...");
		System.out.println(Thread.currentThread()+" end...");
	}
}
```



## 2. SerrvletContainerInitializer启动容器Context

+ arg0 中传入HandlesTypes里面的类
+ sc 是web应用的ServletContext，以编码的方式注入组件
  + 注入serlvet
  + 注入listener
  + 注入filter

```java
@HandlesTypes(value={HelloService.class})
public class MyServletContainerInitializer implements ServletContainerInitializer {

	@Override
	public void onStartup(Set<Class<?>> arg0, ServletContext sc) throws ServletException {
		// TODO Auto-generated method stub
		System.out.println("感兴趣的类型：");
		for (Class<?> claz : arg0) {
			System.out.println(claz);
		}

		//注册组件  ServletRegistration
		ServletRegistration.Dynamic servlet = sc.addServlet("userServlet", new UserServlet());
		//配置servlet的映射信息
		servlet.addMapping("/user");


		//注册Listener
		sc.addListener(UserListener.class);

		//注册Filter  FilterRegistration
		FilterRegistration.Dynamic filter = sc.addFilter("userFilter", UserFilter.class);
		//配置Filter的映射信息
		filter.addMappingForUrlPatterns(EnumSet.of(DispatcherType.REQUEST), true, "/*");

	}
}
```



## 3. 整合spring mvc

### 整合原理

+ web容器在启动的时候，会扫描每个jar包下的**META-INF/services/javax.servlet.ServletContainerInitializer**
+ 加载这个文件指定的类SpringServletContainerInitializer
+ spring的**应用一启动会加载感兴趣的WebApplicationInitializer接口的下的所有组件**；
+ 为WebApplicationInitializer组件创建对象（组件不是接口，不是抽象类）
  + AbstractContextLoaderInitializer：创建根容器；createRootApplicationContext()；
  + **AbstractDispatcherServletInitializer: xml方式配置DispatchServlet**
    + 创建一个web的ioc容器；createServletApplicationContext();
    + 创建了DispatcherServlet；createDispatcherServlet()；
    + 将创建的DispatcherServlet添加到ServletContext中；
  + **AbstractAnnotationConfigDispatcherServletInitializer：注解方式配置的DispatcherServlet初始化器**
    + 创建根容器：**createRootApplicationContext**
    + 创建web的ioc容器： **createServletApplicationContext**
    + 获取配置类；getServletConfigClasses();
      	



![](./images/servlet-01.png)

### RootConfig

+ service
+ repositories

```JAVA
@ComponentScan(value= "com.yufeng",excludeFilters={
		@Filter(type=FilterType.ANNOTATION,classes={Controller.class})
})
public class RootConfig {

}

```

### AppConfig

+ controller
+ viewResolver
+ handlerMapping
+ interceptor

```JAVA
@ComponentScan(value="com.atguigu",includeFilters={
		@Filter(type=FilterType.ANNOTATION,classes={Controller.class})
},useDefaultFilters=false)
@EnableWebMvc
public class AppConfig  implements WebMvcConfigurer {

	//定制

	//视图解析器
	@Override
	public void configureViewResolvers(ViewResolverRegistry registry) {
		//默认所有的页面都从 /WEB-INF/ xxx .jsp
		//registry.jsp();
		registry.jsp("/WEB-INF/views/", ".jsp");
	}

	//静态资源访问
	@Override
	public void configureDefaultServletHandling(DefaultServletHandlerConfigurer configurer) {
		configurer.enable();
	}

	//拦截器
	@Override
	public void addInterceptors(InterceptorRegistry registry) {
		registry.addInterceptor(new MyFirstInterceptor()).addPathPatterns("/**");
	}
}
```



### WebAppInitializer完成初始化

```JAVA
//web容器启动的时候创建对象；调用方法来初始化容器以前前端控制器
public class MyWebAppInitializer extends AbstractAnnotationConfigDispatcherServletInitializer {

	//获取根容器的配置类；（Spring的配置文件）   父容器；
	@Override
	protected Class<?>[] getRootConfigClasses() {
		return new Class<?>[]{RootConfig.class};
	}

	//获取web容器的配置类（SpringMVC配置文件）  子容器；
	@Override
	protected Class<?>[] getServletConfigClasses() {
		return new Class<?>[]{AppConfig.class};
	}

	//获取DispatcherServlet的映射信息
	//  /：拦截所有请求（包括静态资源（xx.js,xx.png）），但是不包括*.jsp；
	//  /*：拦截所有请求；连*.jsp页面都拦截；jsp页面是tomcat的jsp引擎解析的；
	@Override
	protected String[] getServletMappings() {
		return new String[]{"/"};
	}
}

```



## 5. 异步请求

### Servlet 3

使用AysnContext开启异步处理

```java
@WebServlet(value="/async",asyncSupported=true)
public class HelloAsyncServlet extends HttpServlet {

	@Override
	protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws ServletException, IOException {
		//1、支持异步处理asyncSupported=true
		//2、开启异步模式
		System.out.println("主线程开始。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
		AsyncContext startAsync = req.startAsync();

		//3、业务逻辑进行异步处理;开始异步处理
		startAsync.start(() -> {
				try {
					System.out.println("副线程开始。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
					sayHello();
					startAsync.complete();
					//获取到异步上下文
					AsyncContext asyncContext = req.getAsyncContext();
					//4、获取响应
					ServletResponse response = asyncContext.getResponse();
					response.getWriter().write("hello async...");
					System.out.println("副线程结束。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
				} catch (Exception e) {
					e.printStackTrace();
				}
			});
		System.out.println("主线程结束。。。"+Thread.currentThread()+"==>"+System.currentTimeMillis());
	}
}	
```



### Spring mvc

+ 控制器返回Callable
+ Spring异步处理，将Callable 提交到 TaskExecutor 使用一个隔离的线程进行执行
+ **DispatcherServlet和所有的Filter退出web容器的线程，但是response 保持打开状态**
+ **Callable返回结果，SpringMVC将请求重新派发给容器，恢复之前的处理**
+ 根据Callable返回的结果。SpringMVC继续进行视图渲染流程等（从收请求-视图渲染）

```JAVA
@Controller
public class AsyncController {

	@ResponseBody
	@RequestMapping("/async01")
	public Callable<String> async01(){
		System.out.println("主线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());

		Callable<String> callable = new Callable<String>() {
			@Override
			public String call() throws Exception {
				System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
				Thread.sleep(2000);
				System.out.println("副线程开始..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
				return "Callable<String> async01()";
			}
		};

		System.out.println("主线程结束..."+Thread.currentThread()+"==>"+System.currentTimeMillis());
		return callable;
	}
}
```

