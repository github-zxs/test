# 一.Pinia 中按模块分组管理全局 Loading 方案

你想要在 Pinia 中实现**按模块/分组管理**的全局 loading 状态，而不是单一的全局 loading，这样可以分别控制不同业务模块（比如用户模块、订单模块、商品模块）的加载状态，避免相互干扰。

下面我会提供一个更精细化的分组式全局 loading 实现方案，支持按模块/分组独立管理 loading 状态，同时保留全局汇总能力。

### 实现思路

1. 设计分层的 loading 状态结构：`{ 分组名: { 具体key: 布尔值 } }`
    
2. 提供按分组/按 key 操作 loading 的方法
    
3. 增加分组级别的计算属性（判断某个分组是否有加载中状态）
    
4. 保留全局汇总的计算属性（判断所有分组是否有加载中状态）
    

### 完整代码实现

#### 1. 创建分组式 Loading Store

在 `stores/loading.js` 文件中实现分组管理：

```JavaScript
// stores/loading.js
import { defineStore } from 'pinia'

export const useLoadingStore = defineStore('loading', {
  state: () => ({
    // 分层结构：groups -> groupKey -> loadingKey -> boolean
    groups: {
      // 示例：默认分组
      default: {},
      // 业务分组示例
      user: {},    // 用户模块：如 user/list、user/detail
      order: {},   // 订单模块：如 order/list、order/submit
      product: {}  // 商品模块：如 product/edit、product/upload
    }
  }),

  getters: {
    /**
     * 判断某个分组是否有加载中的状态
     * @returns {Function} 接收分组名，返回布尔值
     */
    isGroupLoading: (state) => (groupKey = 'default') => {
      const group = state.groups[groupKey] || {}
      return Object.values(group).some(loading => loading)
    },

    /**
     * 判断全局是否有任何加载中的状态
     * @returns {boolean}
     */
    isAnyLoading: (state) => {
      // 遍历所有分组，判断是否有任意 loading 为 true
      return Object.values(state.groups).some(group => 
        Object.values(group).some(loading => loading)
      )
    },

    /**
     * 获取指定分组的所有 loading 状态
     * @returns {Function} 接收分组名，返回该分组的 loading 对象
     */
    getGroupLoadings: (state) => (groupKey = 'default') => {
      return state.groups[groupKey] || {}
    }
  },

  actions: {
    /**
     * 初始化分组（防止分组不存在）
     * @param {string} groupKey 分组名
     */
    initGroup(groupKey) {
      if (!this.groups[groupKey]) {
        this.groups[groupKey] = {}
      }
    },

    /**
     * 开启指定分组下的 loading
     * @param {string} loadingKey 具体的 loading 标识
     * @param {string} groupKey 分组名，默认 default
     */
    startLoading(loadingKey, groupKey = 'default') {
      this.initGroup(groupKey)
      this.groups[groupKey][loadingKey] = true
    },

    /**
     * 关闭指定分组下的 loading
     * @param {string} loadingKey 具体的 loading 标识
     * @param {string} groupKey 分组名，默认 default
     */
    stopLoading(loadingKey, groupKey = 'default') {
      this.initGroup(groupKey)
      this.groups[groupKey][loadingKey] = false
    },

    /**
     * 关闭某个分组下的所有 loading
     * @param {string} groupKey 分组名，默认 default
     */
    stopGroupAllLoading(groupKey = 'default') {
      this.initGroup(groupKey)
      Object.keys(this.groups[groupKey]).forEach(key => {
        this.groups[groupKey][key] = false
      })
    },

    /**
     * 关闭所有分组的所有 loading
     */
    stopAllLoading() {
      Object.keys(this.groups).forEach(groupKey => {
        this.stopGroupAllLoading(groupKey)
      })
    },

    /**
     * 切换指定分组下的 loading 状态
     * @param {string} loadingKey 具体的 loading 标识
     * @param {string} groupKey 分组名，默认 default
     * @param {boolean} status 可选，指定状态
     */
    toggleLoading(loadingKey, groupKey = 'default', status) {
      this.initGroup(groupKey)
      if (typeof status === 'boolean') {
        this.groups[groupKey][loadingKey] = status
      } else {
        this.groups[groupKey][loadingKey] = !this.groups[groupKey][loadingKey]
      }
    }
  }
})
```

#### 2. 封装分组式请求辅助函数（推荐）

创建通用的带分组 loading 的请求封装，简化使用：

```JavaScript
// utils/requestWithLoading.js
import { useLoadingStore } from '@/stores/loading'

/**
 * 带分组 loading 的请求封装
 * @param {Function} requestFn 请求函数（返回Promise）
 * @param {string} loadingKey 具体的 loading 标识
 * @param {string} groupKey 分组名，默认 default
 * @returns {Promise} 请求结果
 */
export async function requestWithLoading(requestFn, loadingKey = 'default', groupKey = 'default') {
  const loadingStore = useLoadingStore()
  
  // 开启指定分组下的 loading
  loadingStore.startLoading(loadingKey, groupKey)
  
  try {
    const result = await requestFn()
    return result
  } catch (error) {
    throw error // 抛出错误，让调用方处理
  } finally {
    // 无论成功失败，关闭 loading
    loadingStore.stopLoading(loadingKey, groupKey)
  }
}
```

#### 3. 在组件中使用（多分组示例）

```Plain
<template>
  <div class="loading-demo">
    <!-- 全局加载状态 -->
    <div v-if="loadingStore.isAnyLoading" class="global-loading">
      全局有加载任务中...
    </div>

    <!-- 用户分组加载状态 -->
    <div v-if="loadingStore.isGroupLoading('user')" class="group-loading user-loading">
      用户模块加载中...
    </div>

    <!-- 订单分组加载状态 -->
    <div v-if="loadingStore.isGroupLoading('order')" class="group-loading order-loading">
      订单模块加载中...
    </div>

    <!-- 具体key的loading -->
    <div v-if="loadingStore.groups.user.list" class="item-loading">
      用户列表加载中...
    </div>

    <button @click="fetchUserList">获取用户列表（user/list）</button>
    <button @click="fetchOrderDetail">获取订单详情（order/detail）</button>
    <button @click="stopUserGroupLoading">停止用户模块所有loading</button>
    <button @click="stopAllLoading">停止所有loading</button>
  </div>
</template>

<script setup>
import { useLoadingStore } from '@/stores/loading'
import { requestWithLoading } from '@/utils/requestWithLoading'

const loadingStore = useLoadingStore()

// 1. 使用封装函数 - 用户模块列表
async function fetchUserList() {
  try {
    const res = await requestWithLoading(
      // 模拟请求：2秒后返回用户列表
      () => new Promise(resolve => setTimeout(() => resolve([{ id: 1, name: '张三' }]), 2000)),
      'list', // loadingKey
      'user'  // groupKey
    )
    console.log('用户列表', res)
  } catch (err) {
    console.error('获取用户列表失败', err)
  }
}

// 2. 使用封装函数 - 订单模块详情
async function fetchOrderDetail() {
  try {
    const res = await requestWithLoading(
      // 模拟请求：1.5秒后返回订单详情
      () => new Promise(resolve => setTimeout(() => resolve({ id: 1001, amount: 99 }), 1500)),
      'detail', // loadingKey
      'order'   // groupKey
    )
    console.log('订单详情', res)
  } catch (err) {
    console.error('获取订单详情失败', err)
  }
}

// 3. 手动停止用户分组所有loading
function stopUserGroupLoading() {
  loadingStore.stopGroupAllLoading('user')
}

// 4. 手动停止所有loading
function stopAllLoading() {
  loadingStore.stopAllLoading()
}
</script>

<style scoped>
.global-loading {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  height: 30px;
  background: #409eff;
  color: white;
  text-align: center;
  line-height: 30px;
}

.group-loading {
  margin: 10px 0;
  padding: 8px;
  border-radius: 4px;
  color: white;
}

.user-loading {
  background: #67c23a;
}

.order-loading {
  background: #e6a23c;
}

.item-loading {
  margin: 10px 0;
  color: #909399;
}

button {
  margin: 0 5px 10px 0;
  padding: 6px 12px;
  cursor: pointer;
}
</style>
```

### 前置条件

和之前的基础版一致，确保已安装并配置 Pinia：

```Bash
npm install pinia
```

```JavaScript
// main.js
import { createPinia } from 'pinia'
import { createApp } from 'vue'
import App from './App.vue'

const app = createApp(App)
app.use(createPinia())
app.mount('#app')
```

### 扩展场景：动态新增分组

如果你的业务分组是动态的（比如根据接口返回的模块名创建），可以直接调用 `initGroup` 方法：

```JavaScript
// 动态创建一个支付模块的分组
loadingStore.initGroup('payment')
// 开启支付模块的提交loading
loadingStore.startLoading('submit', 'payment')
```

### 总结

1. **核心结构**：采用 `groups[分组名][具体key]` 的分层结构，实现多分组 loading 独立管理，避免模块间干扰。
    
2. **核心方法**：支持按 `分组+key` 操作 loading（`startLoading/stopLoading`），也支持分组级/全局级批量停止（`stopGroupAllLoading/stopAllLoading`）。
    
3. **核心计算属性**：`isGroupLoading` 判断单个分组状态，`isAnyLoading` 判断全局状态，满足不同层级的展示需求。



# 二.Vue3 中基于 Hooks 和 customRef 实现全局 Loading

你希望基于 Vue3 的 Hooks + `customRef` 实现**针对异步请求的全局 Loading**，核心需求是：在所有异步请求（如接口调用）发起时自动显示全局 Loading，请求结束（成功/失败）后自动关闭，且能避免多请求叠加导致的 Loading 提前关闭问题。

下面我会提供一个**更健壮的实现方案**，解决多请求并发时的 Loading 状态管理问题（比如同时发 3 个请求，需等最后一个请求结束才关闭 Loading），完全基于 Hooks + `customRef` 设计，符合 Vue3 组合式 API 最佳实践。

### 完整实现步骤

#### 1. 封装核心 Hook（useRequestLoading.js）

这个 Hook 是核心，用 `customRef` 管理 Loading 状态，同时通过**请求计数器**解决多请求并发问题：

```JavaScript
// src/hooks/useRequestLoading.js
import { customRef, provide, inject } from 'vue'

// 全局唯一注入Key
const REQUEST_LOADING_KEY = Symbol('request-global-loading')

// 用customRef封装Loading状态 + 并发请求计数器
function createRequestLoadingRef() {
  let loading = false
  let requestCount = 0 // 并发请求计数器

  return customRef((track, trigger) => {
    return {
      // 读取Loading状态时收集依赖
      get() {
        track()
        return loading
      },
      // 禁止外部直接修改，仅内部通过方法控制
      set() {
        console.warn('全局Loading状态不允许直接修改，请使用start/stop方法')
      },
      // 内部方法：启动Loading（计数器+1）
      start: () => {
        requestCount++
        loading = true
        trigger() // 触发视图更新
      },
      // 内部方法：停止Loading（计数器-1，为0时关闭）
      stop: () => {
        if (requestCount <= 0) return
        requestCount--
        // 只有所有请求都结束（计数器为0），才关闭Loading
        if (requestCount === 0) {
          loading = false
          trigger() // 触发视图更新
        }
      },
      // 强制关闭（兜底用，比如请求超时）
      forceStop: () => {
        requestCount = 0
        loading = false
        trigger()
      }
    }
  })
}

// 根组件提供全局Loading状态和方法
export function provideRequestLoading() {
  const loadingRef = createRequestLoadingRef()
  
  // 暴露给外部的方法
  const loadingApi = {
    // 获取当前Loading状态
    get loading() {
      return loadingRef.value
    },
    // 启动Loading（请求开始时调用）
    startLoading: () => loadingRef.start(),
    // 停止Loading（请求结束时调用）
    stopLoading: () => loadingRef.stop(),
    // 强制关闭Loading
    forceStopLoading: () => loadingRef.forceStop()
  }

  // 提供给子组件使用
  provide(REQUEST_LOADING_KEY, loadingApi)
  
  return loadingApi
}

// 子组件/工具函数中注入使用
export function useRequestLoading() {
  const instance = inject(REQUEST_LOADING_KEY)
  
  if (!instance) {
    throw new Error('useRequestLoading 必须在 provideRequestLoading 之后使用（需在App.vue中调用provideRequestLoading）')
  }
  
  return instance
}
```

#### 2. 全局挂载 Loading 组件（App.vue）

在根组件中提供 Loading 状态，并渲染全屏 Loading 遮罩：

```Plain
<!-- src/App.vue -->
<template>
  <!-- 全局请求Loading遮罩 -->
  <div v-if="loading" class="request-loading-mask">
    <n-spin size="large" />
    <p class="loading-text">加载中...</p>
  </div>
  
  <!-- 页面主体内容 -->
  <router-view />
</template>

<script setup>
import { NSpin } from 'naive-ui'
import { provideRequestLoading } from './hooks/useRequestLoading'

// 提供全局请求Loading
const { loading } = provideRequestLoading()
</script>

<style scoped>
.request-loading-mask {
  position: fixed;
  top: 0;
  left: 0;
  width: 100vw;
  height: 100vh;
  background: rgba(255, 255, 255, 0.9);
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  z-index: 9999; /* 确保遮罩在最上层 */
  backdrop-filter: blur(1px); /* 可选：添加模糊效果，提升体验 */
}

.loading-text {
  margin-top: 16px;
  color: #666;
  font-size: 14px;
}
</style>
```

#### 3. 封装 Axios 拦截器（自动触发 Loading）

这是最核心的一步：让所有异步请求自动触发/关闭 Loading，无需手动调用。

```JavaScript
// src/utils/axios.js
import axios from 'axios'
import { useRequestLoading } from '../hooks/useRequestLoading'

// 创建Axios实例
const service = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || '/api', // 适配Vite环境变量
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json'
  }
})

// 请求拦截器：发起请求时启动Loading
service.interceptors.request.use(
  (config) => {
    // 排除不需要Loading的请求（可通过config配置控制）
    if (!config.headers?.noLoading) {
      const { startLoading } = useRequestLoading()
      startLoading()
    }
    return config
  },
  (error) => {
    // 请求失败（比如参数错误），立即停止Loading
    const { stopLoading } = useRequestLoading()
    stopLoading()
    return Promise.reject(error)
  }
)

// 响应拦截器：请求结束时停止Loading
service.interceptors.response.use(
  (response) => {
    // 排除不需要Loading的响应
    if (!response.config.headers?.noLoading) {
      const { stopLoading } = useRequestLoading()
      stopLoading()
    }
    return response
  },
  (error) => {
    // 请求失败（超时/接口错误等），停止Loading
    if (!error.config?.headers?.noLoading) {
      const { stopLoading } = useRequestLoading()
      stopLoading()
    }
    return Promise.reject(error)
  }
)

export default service
```

#### 4. 实际使用示例（组件中调用接口）

```Plain
<!-- src/components/ApiDemo.vue -->
<template>
  <n-button @click="fetchData">发起单个请求</n-button>
  <n-button @click="fetchMultipleData" type="primary">发起多个并发请求</n-button>
  <n-button @click="fetchNoLoadingData" type="warning">发起无Loading请求</n-button>
</template>

<script setup>
import axios from '../utils/axios'

// 单个请求
const fetchData = async () => {
  try {
    const res = await axios.get('/user/info')
    console.log('单个请求结果：', res.data)
  } catch (err) {
    console.error('请求失败：', err)
  }
}

// 多个并发请求（Loading会等所有请求结束才关闭）
const fetchMultipleData = async () => {
  try {
    // 同时发起3个请求
    const [res1, res2, res3] = await Promise.all([
      axios.get('/user/list'),
      axios.get('/goods/list'),
      axios.get('/order/list')
    ])
    console.log('并发请求结果：', res1.data, res2.data, res3.data)
  } catch (err) {
    console.error('并发请求失败：', err)
  }
}

// 不需要Loading的请求（通过headers控制）
const fetchNoLoadingData = async () => {
  try {
    const res = await axios.get('/common/config', {
      headers: { noLoading: true } // 标记不需要Loading
    })
    console.log('无Loading请求结果：', res.data)
  } catch (err) {
    console.error('无Loading请求失败：', err)
  }
}
</script>
```

### 核心逻辑解释

1. **customRef 核心作用**：
    
    1. 封装了 `loading` 状态和 `requestCount` 计数器，避免外部直接修改状态，保证逻辑可控；
        
    2. `track()` 收集依赖（让 App.vue 中的 Loading 遮罩能感知状态变化）；
        
    3. `trigger()` 触发更新（状态变化时自动渲染遮罩的显示/隐藏）。
        
2. **请求计数器的意义**：
    
    1. 每次请求发起时 `requestCount++`，并开启 Loading；
        
    2. 每次请求结束时 `requestCount--`，只有当 `requestCount === 0` 时才关闭 Loading；
        
    3. 解决了“多请求叠加时，第一个请求结束就关闭 Loading”的问题。
        
3. **灵活控制 Loading**：
    
    1. 通过 `headers.noLoading: true` 可以单独控制某个请求不显示 Loading；
        
    2. 提供 `forceStopLoading` 方法，可在特殊场景（如页面跳转）强制关闭 Loading。
        

### 总结

1. 核心方案：用 `customRef` 封装带**并发计数器**的 Loading 状态，结合 `provide/inject` 实现全局共享，再通过 Axios 拦截器自动绑定请求生命周期。
    
2. 关键特性：
    
    1. 自动触发：所有接口请求无需手动调用 Loading 方法，拦截器自动处理；
        
    2. 并发兼容：多请求叠加时，Loading 会等最后一个请求结束才关闭；
        
    3. 灵活控制：支持通过 `headers.noLoading` 排除特定请求的 Loading。
        
3. 优势：相比普通 ref，`customRef` 让 Loading 状态的读写逻辑更封闭、可控，Hook 封装后复用性极强，符合 Vue3 组合式 API 设计思想。


# 三.Pinia 全局加载状态管理：从基础到实用

你想要在 Pinia 中实现一个全局的 loading 状态管理，用于统一控制整个应用的加载状态，比如请求接口时显示加载动画。

下面我会提供一个完整、易用的 Pinia 全局 loading 实现方案，包含基础的状态管理和实用的辅助方法。

### 实现思路

1. 创建一个全局的 Pinia store 来管理 loading 状态
    
2. 支持单个/多个加载状态的独立控制（避免全局 loading 冲突）
    
3. 提供通用的开启/关闭/切换方法
    
4. 封装成可复用的工具函数，方便在请求中直接使用
    

### 完整代码实现

#### 1. 创建 Loading Store

在 `stores/loading.js` 文件中创建全局 loading 状态管理：

```JavaScript
// stores/loading.js
import { defineStore } from 'pinia'

export const useLoadingStore = defineStore('loading', {
  // 状态定义
  state: () => ({
    // 使用对象存储多个loading状态，key为加载标识，value为布尔值
    loadings: {},
    // 全局通用loading（适用于不需要区分的场景）
    globalLoading: false
  }),

  // 计算属性
  getters: {
    // 判断是否有任何加载中的状态
    isAnyLoading: (state) => {
      return state.globalLoading || Object.values(state.loadings).some(loading => loading)
    }
  },

  // 方法
  actions: {
    /**
     * 开启指定的loading
     * @param {string} key loading标识，不传则开启全局loading
     */
    startLoading(key) {
      if (key) {
        this.loadings[key] = true
      } else {
        this.globalLoading = true
      }
    },

    /**
     * 关闭指定的loading
     * @param {string} key loading标识，不传则关闭全局loading
     */
    stopLoading(key) {
      if (key) {
        this.loadings[key] = false
      } else {
        this.globalLoading = false
      }
    },

    /**
     * 切换指定loading的状态
     * @param {string} key loading标识
     * @param {boolean} status 可选，指定状态
     */
    toggleLoading(key, status) {
      if (typeof status === 'boolean') {
        this.loadings[key] = status
      } else {
        this.loadings[key] = !this.loadings[key]
      }
    },

    /**
     * 清空所有loading状态
     */
    clearAllLoading() {
      this.globalLoading = false
      this.loadings = {}
    }
  }
})
```

#### 2. 封装请求辅助函数（实用增强）

创建一个工具函数，自动管理请求的 loading 状态：

```JavaScript
// utils/requestWithLoading.js
import { useLoadingStore } from '@/stores/loading'

/**
 * 带loading的请求封装
 * @param {Function} requestFn 请求函数（返回Promise）
 * @param {string} loadingKey loading标识
 * @returns {Promise} 请求结果
 */
export async function requestWithLoading(requestFn, loadingKey = 'default') {
  const loadingStore = useLoadingStore()
  
  // 开启loading
  loadingStore.startLoading(loadingKey)
  
  try {
    // 执行请求
    const result = await requestFn()
    return result
  } catch (error) {
    // 抛出错误，让调用方处理
    throw error
  } finally {
    // 无论成功失败，都关闭loading
    loadingStore.stopLoading(loadingKey)
  }
}
```

#### 3. 在组件中使用

```Plain
<template>
  <div>
    <!-- 全局loading -->
    <div v-if="loadingStore.globalLoading" class="global-loading">加载中...</div>
    
    <!-- 特定loading -->
    <div v-if="loadingStore.loadings.userList" class="list-loading">列表加载中...</div>
    
    <button @click="fetchUserList">获取用户列表</button>
    <button @click="fetchData">获取数据</button>
  </div>
</template>

<script setup>
import { useLoadingStore } from '@/stores/loading'
import { requestWithLoading } from '@/utils/requestWithLoading'

// 获取loading store实例
const loadingStore = useLoadingStore()

// 方式1：手动控制loading
async function fetchData() {
  try {
    // 开启全局loading
    loadingStore.startLoading()
    
    // 模拟接口请求
    await new Promise(resolve => setTimeout(resolve, 2000))
    console.log('数据获取成功')
  } catch (error) {
    console.error('请求失败', error)
  } finally {
    // 关闭全局loading
    loadingStore.stopLoading()
  }
}

// 方式2：使用封装的工具函数（推荐）
async function fetchUserList() {
  try {
    const result = await requestWithLoading(
      // 传入请求函数
      () => new Promise(resolve => setTimeout(() => resolve([{ id: 1, name: '张三' }]), 1500)),
      // 指定loading标识
      'userList'
    )
    console.log('用户列表', result)
  } catch (error) {
    console.error('获取用户列表失败', error)
  }
}
</script>

<style scoped>
.global-loading {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.5);
  color: white;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 20px;
}

.list-loading {
  color: #666;
  padding: 20px;
  text-align: center;
}
</style>
```

### 前置条件

1. 已正确安装并配置 Pinia：
    
    ```Bash
    npm install pinia
    # 或
    yarn add pinia
    ```
    
2. 在项目入口文件（如 `main.js`）中注册 Pinia：
    
    ```JavaScript
    import { createPinia } from 'pinia'
    import { createApp } from 'vue'
    import App from './App.vue'
    
    const app = createApp(App)
    app.use(createPinia())
    app.mount('#app')
    ```
    

### 总结

1. **核心实现**：通过 Pinia 的 `loading` store 管理全局加载状态，使用对象存储多个独立的 loading 标识，避免状态冲突。
    
2. **使用方式**：支持手动控制（`startLoading`/`stopLoading`）和自动控制（`requestWithLoading` 封装）两种方式，后者更简洁且不易出错。
    
3. **关键特性**：提供 `isAnyLoading` 计算属性判断是否有任意加载状态，`clearAllLoading` 方法可重置所有 loading 状态，满足不同场景需求。