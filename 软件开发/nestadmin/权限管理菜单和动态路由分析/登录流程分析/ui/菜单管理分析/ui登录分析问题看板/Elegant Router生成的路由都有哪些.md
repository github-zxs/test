### [Elegant Router自动化路由实践1](https://blog.csdn.net/gitblog_07450/article/details/148683264)
### [Elegant Router自动化路由实战2](https://blog.csdn.net/gitblog_00291/article/details/150496541)

### Elegant Router 生成的文件结构

启动项目后，ElegantRouter 会在配置的目录（默认为 `src/router/_generated`）下生成以下文件：
1.  **routes.ts** - 存储基于文件系统生成的完整路由配置
2.  **imports.ts** - 包含所有布局和视图组件的自动导入语句
3. **transformer.ts** - 提供路由转换函数，将一级路由转换为框架所需的嵌套结构
4.  **shared.ts** - 提供类型安全的路由工具函数，便于在代码中引用路由
5. **builtin.ts** - 根据路径自动生成的常量路由