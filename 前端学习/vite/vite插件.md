## 插件机制概述
- 插件是一个对象:包括属性,钩子
- 开发插件会写一个函数,返回一个对象,对象中包含属性,和钩子.
### 属性
- name: 插件名称
- enforce: 顺序
- apply: 应用场景
### 钩子
- 通用钩子
- vite独有钩子
	1. config
	2. configResolved
	3. configureServer
	4. configurePreviewServer
	5. transformIndexHtml
	6. handleHostUpdate

## 插件开发规范
- rollup-plugin-xxx (package.json rollup-plugin)
- vite-plugin-xxx(packge.json vite-plugin)
## 加载顺序
- 别名插件(vite:pre-alias @rollup/plugin-alias内置插件)
- 核心插件(vite:resolve, vite:transform, vitest：html css json)