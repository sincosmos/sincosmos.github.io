# Review
## Web 开发项目回顾
### 算法管理系统项目架构
项目基于 Spring Boot 搭建，引入了 Spring MVC, Shiro, Mybatis, RedisTemplate 等框架依赖。Spring MVC 做前后端交互支持，Shiro 用来做权限管理，mybatis 作为 ORM 工具，RedisTemplate 用以简化程序与 redis 的交互，将 web session 信息保存到 redis 中。
### Spring MVC
1. Spring MVC 由 DispatcherServlet 统一接受符合 pattern 的所有请求。
2. DispatcherServlet 调用 HandlerMapping（使用 BeanNameUrlHandlerMapping, DefaultAnnotationHandlerMapping 等）的 getHandler(HttpServletRequest request) 方法，该方法解析 request 的资源标志符 URI，获得为该请求配置的 HandlerExecutionChain (Interceptor & Servlet)，HandlerExecutionChain 对象被返回给 DispatcherServlet。handler 和 URI 的映射是在 Spring MVC 的阶段完成的，映射关系会被保存起来供查询。
3. DispatcherServlet 根据获得的 HandlerExecutionChain，选择一个合适的 HandlerAdapter (默认为 HttpRequestHandlerAdapter, SimpleControllerHandlerAdapter, AnnotationMethodHandlerAdapter)。
4. 成功后开始依次执行拦截器 preHandler() 方法。提取 Request 中的模型数据，填充 Handler 入参，开始执行依次执行 Handler (Controller)，填充入参的过程，Spring 会根据配置，额外进行入参格式转换（例如 HttpMessageConveter）、数据验证等工作。Handler 执行完成后，向 DispatcherServlet 返回 ModelAndView 对象。DispatcherServlet 根据获得的 ModelAndView 选择一个合适的 ViewResolver (InternalResourceViewResolver) 对返回给客户端的视图进行渲染，最后把渲染结果返回给客户端。
### Shiro
1. Filter 在 DispatcherServlet 之前拦截请求，并可以修改请求头和数据。实现 Filter 接口的 Bean 会被 Spring boot 自动加入到过滤器链中。在我们的项目中，ShiroFilter 拦截所有请求（如果请求是 http 请求，将其封装为 ShiroHttpServletRequest 后向后发送）。如果请求的资源允许匿名访问，shiroFilter 将允许其通过。否则 shiroFilter 将尝试获取用户 Session