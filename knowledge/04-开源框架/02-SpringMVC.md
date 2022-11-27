# SpringMVC
## SpringMVC处理请求的过程
- 用户请求DispatcherServlet；
- DispatcherServlet请求处理器映射器HandlerMapping，查找对应的handler，并返回给DispatcherServlet；
- DispatcherServlet请求处理器适配器HandlerAdapter去执行handler处理器；
- Handler执行完毕，返回DispatcherServlet ModeAndView对象；
- DispatcherServlet拿到ModeAndView后执行视图解析器ViewResolver将其解析成视图view并进行视图渲染；
- 视图解析器将渲染后的视图返回给DispatcherServlet，由其返回给客户端；

