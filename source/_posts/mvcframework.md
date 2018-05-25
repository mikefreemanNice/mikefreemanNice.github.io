---
title: 搭建一个简单的Java MVC框架
date: 2018-04-26 00:27:14
tags: [Java]
comments: true
categories: [Java]
description: 如何搭建一个Java MVC框架
---

# 路由设计
## 路由对象
```
public class Route {

	/**
	 * 路由path
	 */
	private String path;

	/**
	 * 执行路由的方法
	 */
	private Method action;

	/**
	 * 路由所在的控制器
	 */
	private Object controller;
}
```
所有的请求在程序中是一个路由，匹配在 path 上，执行靠 action，处于 controller 中。
## 路由管理
```
/**
 * 路由管理类
**/
public class Routers {

	private static final Logger LOGGER = Logger.getLogger(Routers.class.getName());
	
	private List<Route> routes = new ArrayList<Route>();
	
	public Routers() {
	}
	
	public void addRoute(List<Route> routes){
		routes.addAll(routes);
	}
	
	public void addRoute(Route route){
		routes.add(route);
	}
	
	public void removeRoute(Route route){
		routes.remove(route);
	}
	
	public void addRoute(String path, Method action, Object controller){
		Route route = new Route();
		route.setPath(path);
		route.setAction(action);
		route.setController(controller);
		
		routes.add(route);
		LOGGER.info("Add Route：[" + path + "]");
	}

	public List<Route> getRoutes() {
		return routes;
	}

	public void setRoutes(List<Route> routes) {
		this.routes = routes;
	}
	
}
```
## 路由匹配
```
/**
 * 路由匹配器，用于匹配路由
 */
public class RouteMatcher {

	private List<Route> routes;

	public RouteMatcher(List<Route> routes) {
		this.routes = routes;
	}
	
	public void setRoutes(List<Route> routes) {
		this.routes = routes;
	}

	/**
	 * 根据path查找路由
	 * @param path	请求地址
	 * @return		返回查询到的路由
	 */
	public Route findRoute(String path) {
		String cleanPath = parsePath(path);
		List<Route> matchRoutes = new ArrayList<Route>();
		for (Route route : this.routes) {
			if (matchesPath(route.getPath(), cleanPath)) {
				matchRoutes.add(route);
			}
		}
		// 优先匹配原则
        giveMatch(path, matchRoutes);
        
        return matchRoutes.size() > 0 ? matchRoutes.get(0) : null;
	}

	private void giveMatch(final String uri, List<Route> routes) {
		Collections.sort(routes, new Comparator<Route>() {
			@Override
			public int compare(Route o1, Route o2) {
				if (o2.getPath().equals(uri)) {
					return o2.getPath().indexOf(uri);
				}
				return -1;
			}
		});
	}
	
	private boolean matchesPath(String routePath, String pathToMatch) {
		routePath = routePath.replaceAll(PathUtil.VAR_REGEXP, PathUtil.VAR_REPLACE);
		return pathToMatch.matches("(?i)" + routePath);
	}

	private String parsePath(String path) {
		path = PathUtil.fixPath(path);
		try {
			URI uri = new URI(path);
			return uri.getPath();
		} catch (URISyntaxException e) {
			return null;
		}
	}

}
```
# 控制器设计
## 上下文环境

```
/**
 * 当前线程上下文环境
 */
public final class MarioContext {

	private static final ThreadLocal<MarioContext> CONTEXT = new ThreadLocal<MarioContext>();
	
	private ServletContext context;
	private Request request;
	private Response response;

	private MarioContext() {
	}
	
	public static MarioContext me(){
		return CONTEXT.get();
	}
	
    public static void initContext(ServletContext context, Request request, Response response) {
    	MarioContext marioContext = new MarioContext();
    	marioContext.context = context;
    	marioContext.request = request;
    	marioContext.response = response;
    	CONTEXT.set(marioContext);
    }

}
```

## 控制器核心对象
- 接收用户请求
- 查找路由
- 找到即执行配置的方法
- 找不到你看到的应该是404

```
/**
 * Mario MVC核心处理器
 *
 */
public class MarioFilter implements Filter {
	
	private static final Logger LOGGER = Logger.getLogger(MVCFilter.class.getName());
	
	private RouteMatcher routeMatcher = new RouteMatcher(new ArrayList<Route>());
	
	private ServletContext servletContext;
	
	@Override
	public void init(FilterConfig filterConfig) throws ServletException {
		Mario mario = Mario.me();
		if(!mario.isInit()){
			
			String className = filterConfig.getInitParameter("bootstrap");
			Bootstrap bootstrap = this.getBootstrap(className);
			bootstrap.init(mario);
			
			Routers routers = mario.getRouters();
			if(null != routers){
				routeMatcher.setRoutes(routers.getRoutes());
			}
			servletContext = filterConfig.getServletContext();
			
			mario.setInit(true);
		}
	}
	
	private Bootstrap getBootstrap(String className) {
		if(null != className){
			try {
				Class<?> clazz = Class.forName(className);
				Bootstrap bootstrap = (Bootstrap) clazz.newInstance();
				return bootstrap;
			} catch (ClassNotFoundException e) {
				throw new RuntimeException(e);
			} catch (InstantiationException e) {
				e.printStackTrace();
			} catch (IllegalAccessException e) {
				e.printStackTrace();
			}
		}
		throw new RuntimeException("init bootstrap class error!");
	}
	
	@Override
	public void doFilter(ServletRequest servletRequest, ServletResponse servletResponse, FilterChain chain) throws IOException, ServletException {
		HttpServletRequest request = (HttpServletRequest) servletRequest;
        HttpServletResponse response = (HttpServletResponse) servletResponse;
        request.setCharacterEncoding(Const.DEFAULT_CHAR_SET);
        response.setCharacterEncoding(Const.DEFAULT_CHAR_SET);
        
        // 请求的uri
        String uri = PathUtil.getRelativePath(request);
        
        LOGGER.info("Request URI：" + uri);
        
        Route route = routeMatcher.findRoute(uri);
        
        // 如果找到
		if (route != null) {
			// 实际执行方法
			handle(request, response, route);
		} else{
			chain.doFilter(request, response);
		}
	}
	
	private void handle(HttpServletRequest httpServletRequest, HttpServletResponse httpServletResponse, Route route){
		
		// 初始化上下文
		Request request = new Request(httpServletRequest);
		Response response = new Response(httpServletResponse);
		MarioContext.initContext(servletContext, request, response);
		
		Object controller = route.getController();
		// 要执行的路由方法
		Method actionMethod = route.getAction();
		// 执行route方法
		executeMethod(controller, actionMethod, request, response);
	}
	
	/**
	 * 获取方法内的参数
	 */
	private Object[] getArgs(Request request, Response response, Class<?>[] params){
		
		int len = params.length;
		Object[] args = new Object[len];
		
		for(int i=0; i<len; i++){
			Class<?> paramTypeClazz = params[i];
			if(paramTypeClazz.getName().equals(Request.class.getName())){
				args[i] = request;
			}
			if(paramTypeClazz.getName().equals(Response.class.getName())){
				args[i] = response;
			}
		}
		
		return args;
	}
	
	/**
	 * 执行路由方法
	 */
	private Object executeMethod(Object object, Method method, Request request, Response response){
		int len = method.getParameterTypes().length;
		method.setAccessible(true);
		if(len > 0){
			Object[] args = getArgs(request, response, method.getParameterTypes());
			return ReflectUtil.invokeMehod(object, method, args);
		} else {
			return ReflectUtil.invokeMehod(object, method);
		}
	}

	@Override
	public void destroy() {
	}

}
```

## Servlet过滤器

调用端配置

```
<web-app>
	<display-name>Archetype Created Web Application</display-name>

	<filter>
		<filter-name>test</filter-name>
		<filter-class>com.test.MarioFilter</filter-class>
		<init-param>
			<param-name>bootstrap</param-name>
			<param-value>com.test.App</param-value>
		</init-param>
	</filter>
	<filter-mapping>
		<filter-name>test</filter-name>
		<url-pattern>/*</url-pattern>
	</filter-mapping>
</web-app>

```


# 配置设计
- 添加路由
- 读取资源文件
- 读取配置

```
public final class Mario {

	/**
	 * 存放所有路由
	 */
	private Routers routers;
	
	/**
	 * 配置加载器
	 */
	private ConfigLoader configLoader;
	
	/**
	 * 框架是否已经初始化
	 */
	private boolean init = false;
	
	private Mario() {
		routers = new Routers();
		configLoader = new ConfigLoader();
	}
	
	public boolean isInit() {
		return init;
	}

	public void setInit(boolean init) {
		this.init = init;
	}
	
	private static class MarioHolder {
		private static Mario ME = new Mario();
	}
	
	public static Mario me(){
		return MarioHolder.ME;
	}
	
	public Mario addConf(String conf){
		configLoader.load(conf);
		return this;
	}
	
	public String getConf(String name){
		return configLoader.getConf(name);
	}
	
	public Mario addRoutes(Routers routers){
		this.routers.addRoute(routers.getRoutes());
		return this;
	}

	public Routers getRouters() {
		return routers;
	}
	
	/**
	 * 添加路由
	 * @param path			映射的PATH
	 * @param methodName	方法名称
	 * @param controller	控制器对象
	 * @return				返回Mario
	 */
	public Mario addRoute(String path, String methodName, Object controller){
		try {
			Method method = controller.getClass().getMethod(methodName, Request.class, Response.class);
			this.routers.addRoute(path, method, controller);
		} catch (NoSuchMethodException e) {
			e.printStackTrace();
		} catch (SecurityException e) {
			e.printStackTrace();
		}
		return this;
	}
		
}
```
调用端

```
public class App implements Bootstrap {
	@Override
	public void init(Mario mario) {
		Index index = new Index();
		mario.addRoute("/", "index", index);
		mario.addRoute("/html", "html", index);
	}
}
```

# 视图设计
```
/**
 * JSP渲染实现
 */
public class JspRender implements Render {
	
	@Override
	public void render(String view, Writer writer) {
		
		String viewPath = this.getViewPath(view);
		
		HttpServletRequest servletRequest = MarioContext.me().getRequest().getRaw();
		HttpServletResponse servletResponse = MarioContext.me().getResponse().getRaw();
		try {
			servletRequest.getRequestDispatcher(viewPath).forward(servletRequest, servletResponse);
		} catch (ServletException e) {
			e.printStackTrace();
		} catch (IOException e) {
			e.printStackTrace();
		}
		
	}

	private String getViewPath(String view){
		Mario mario = Mario.me();
		String viewPrfix = mario.getConf(Const.VIEW_PREFIX_FIELD);
		String viewSuffix = mario.getConf(Const.VIEW_SUFFIX_FIELD);

		if (null == viewSuffix || viewSuffix.equals("")) {
			viewSuffix = Const.VIEW_SUFFIX;
		}
		if (null == viewPrfix || viewPrfix.equals("")) {
			viewPrfix = Const.VIEW_PREFIX;
		}
		String viewPath = viewPrfix + "/" + view;
		if (!view.endsWith(viewSuffix)) {
			viewPath += viewSuffix;
		}
		return viewPath.replaceAll("[/]+", "/");
	}
	
	
}
```



