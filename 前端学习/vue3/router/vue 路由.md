### [路由删除](https://blog.csdn.net/qq1195566313/article/details/123783173)
- 通过回调删除路由
    const removeRoute = router.addRoute(routeRecord)
	removeRoute() // 删除路由如果存在的话
- 通过使用 `router.removeRoute(name)` 按名称删除路由
-

### 添加路由
- **路由名称唯一性**：
- 若添加的路由 `name` 已存在，新路由会**覆盖旧路由**（Router4 会抛出警告，需提前检查）；
- 推荐先通过 `router.hasRoute('路由名')` 检查，再添加：
    ```javascript
    if (!router.hasRoute('User')) {
      router.addRoute(newRoute) // 避免重复添加
    }
    ```
    