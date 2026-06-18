## 内置环境变量
**作用**: 
- MODE  
- BASE_URL
- PROD
- DEV
- SSR

### 模式 MODE
-  .env.development env.development.loal 覆盖 env.development 
- vite-env.d.ts 语法提示
- 在页面中使用  %VITE_TEST_VAL% 
- node 中使用 import.meta.env.VITE_TEST_VAL 
### 自定义环境变量 .evn
- import.meta.env.VITE_TEST_VAL 

## HMR
**作用** : 热更新

## glob导入,动态导入

## 依赖预构建
- cjs 2 esm
- 多个依赖合并
