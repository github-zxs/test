### app启动挂载的路由
- 在app挂载路由时先创建createBuiltinVueRoutes(),ROOT_ROUTE, NOT_FOUND_ROUTE
### 守卫中初始化的路由
- 在路由守卫的initConstantRoute 和 initDynamicAuthRoute() 调用  handleConstantAndAuthRoutes() 将路由信息添加到vue router
- handleConstantAndAuthRoutes():  ->resetVueRoutes(); 当存在动态路由时从新清除常量路由初始化中的路由使用动态的路由
- initConstantRoute 和 initDynamicAuthRoute()都把获取到的路由信息存储在 [...constantRoutes.value, ...authRoutes.value]; 变量中动态数据会覆盖静态初始化的数据 ,这也是handleConstantAndAuthRoutes()处理常量路由和动态路由的关键,
- 授权菜单的处理也是同样道理
