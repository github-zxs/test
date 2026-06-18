## 一、`$attrs` 的核心特性

#### 1. **包含的内容**
- 父组件传递的**非 props 属性**（如 `class`、`style`、`id`、自定义属性 `data-*` 等）。
- 父组件传递的**事件监听器**（如 `@click`、`@custom-event`，在 Vue 3 中事件监听器会被包含在 `$attrs` 中，Vue 2 中事件监听器在 `$listeners` 里，Vue 3 已移除 `$listeners`，统一合并到 `$attrs`）。
#### 2. **排除的内容**
    
- 子组件通过 `props` 显式声明的属性（这些属性会被单独提取到 `props` 对象中）。
- `ref`、`key` 等 Vue 内置特殊属性（这些属性由 Vue 内部处理，不会出现在 `$attrs` 中）。
#### 3. **响应式**
- `$attrs` 是**响应式**的，当父组件传递的属性变化时，子组件的 `$attrs` 会自动更新。
- 注意：**不要直接修改 `$attrs`**，它是只读的（修改不会生效，且会触发警告）。
#### 4.  **`inheritAttrs` 选项** 
- 默认情况下，Vue 会将 `$attrs` 中的**非事件属性**（如 `class`、`style`、`id`）自动应用到子组件的**根元素**上。如果想要关闭这个行为，可以使用 `inheritAttrs` 选项。
- 影响根元素的属性继承
	
#### 5. **组合式 API 中访问 `$attrs`**
- 在 `<script setup>` 中，可以通过 `useAttrs()` 函数来访问 `$attrs`（替代直接使用 `this.$attrs`，因为 `<script setup>` 中没有 `this`）。
-  注意：attrs 是一个代理对象，不是普通对象，直接打印可能看不到属性，需通过 attrs.xxx 访问

### 总结

- `$attrs` 是 Vue 3 中处理属性透传的核心工具，包含父组件传递的非 props 属性和事件监听器。
- 用 `v-bind="$attrs"` 实现批量属性透传，用 `inheritAttrs: false` 控制根元素的属性继承。
- 在 `<script setup>` 中用 `useAttrs()` 访问 `$attrs`，替代 `this.$attrs`。
- 相比 Vue 2，Vue 3 移除了 `$listeners`，统一到 `$attrs` 中，简化了属性处理逻辑。

## 二 `class`/`style` 在 `$attrs` 中的使用
#### 1、`class`/`style` 在 `$attrs` 中的基本特性

1. **包含性**：`class` 和 `style` 会被包含在 `$attrs` 中（即使子组件通过 `props` 声明了其他属性，这两个属性仍会出现在 `$attrs` 里）。
2. **合并规则**：
    - **`class`**：子组件自身的 `class` 与父组件传递的 `class` 会**自动合并**（数组 / 字符串形式都会合并）。
    - **`style`**：子组件自身的 `style` 与父组件传递的 `style` 会**按属性合并**（相同的 CSS 属性，子组件的会覆盖父组件的）。
3. **响应式**：和 `$attrs` 其他属性一样，`class`/`style` 也是响应式的，父组件修改后子组件会实时更新。

#### 2、`class`/`style` 的默认继承行为

默认情况下（`inheritAttrs: true`），Vue 会将 `$attrs` 中的 `class`/`style` **自动应用到子组件的根元素**，并与根元素自身的 `class`/`style` 合并。

##### 示例：默认的 `class`/`style` 合并

**父组件**

vue

```vue
<template>
  <Child 
    class="parent-class" 
    style="color: red; font-size: 14px;"
  />
</template>

<script setup>
import Child from './Child.vue';
</script>
```

**子组件（Child.vue）**

vue

```vue
<template>
  <!-- 根元素的 class 是：child-class parent-class；style 是：color: red; font-size: 16px; (font-size 子组件覆盖父组件) -->
  <div class="child-class" style="font-size: 16px;">
    子组件内容
  </div>
</template>

<script setup>
// 未设置 inheritAttrs，默认 true
</script>
```

**渲染结果**：

html

预览

```html
<div class="child-class parent-class" style="color: red; font-size: 16px;">
  子组件内容
</div>
```

#### 3、`inheritAttrs: false` 对 `class`/`style` 的影响

当设置 `inheritAttrs: false` 时，**子组件根元素不会自动继承 `$attrs` 中的任何属性（包括 `class`/`style`）**，但 `class`/`style` 仍然存在于 `$attrs` 中，需要手动通过 `v-bind="$attrs"` 透传。

#### 示例：手动控制 `class`/`style` 的透传

**子组件（Child.vue）**

vue

```vue
<template>
  <div class="wrapper">
    <!-- 手动将 attrs 透传给内部的 p 元素 -->
    <p v-bind="$attrs" class="child-p" style="font-weight: bold;">
      子组件内容
    </p>
  </div>
</template>

<script setup>
// 关闭根元素的自动属性继承
defineOptions({
  inheritAttrs: false
});
</script>
```

**渲染结果**：

html

预览

```html
<div class="wrapper">
  <p class="child-p parent-class" style="color: red; font-size: 14px; font-weight: bold;">
    子组件内容
  </p>
</div>
```

**说明**：

- 根元素 `div.wrapper` 没有任何来自父组件的 `class`/`style`。
- 内部的 `p` 元素通过 `v-bind="$attrs"` 接收了父组件的 `class`/`style`，并与自身的 `class`/`style` 合并。

#### 4、v-bind="$attrs"` 与 `class`/`style 的显式绑定

如果在元素上**同时显式绑定 `class`/`style` 和使用 `v-bind="$attrs"`**，它们会按照「显式绑定 + `$attrs` 中的对应属性」的规则合并，而非覆盖。

**示例：显式绑定 + `$attrs` 合并*

**子组件**

vue

```vue
<template>
  <button
    class="btn-base"
    style="border: 1px solid #ccc;"
    v-bind="$attrs"
  >
    按钮
  </button>
</template>

<script setup>
defineOptions({
  inheritAttrs: false
});
</script>
```

**父组件使用**

vue

```vue
<Child class="btn-primary" style="background: blue; color: white;" />
```

**渲染结果**：

html

预览

```html
<button class="btn-base btn-primary" style="border: 1px solid #ccc; background: blue; color: white;">
  按钮
</button>
```

#### 5、组合式 API 中访问 `$attrs` 里的 `class`/`style`

在 `<script setup>` 中，通过 `useAttrs()` 可以访问 `$attrs` 中的 `class` 和 `style`，但需要注意：

- `attrs.class` 可能是**字符串、数组或对象**（取决于父组件的传递方式）。
- `attrs.style` 可能是**字符串或对象**。

#### 示例：脚本中访问 `class`/`style`

vue

```vue
<template>
  <div v-bind="$attrs">子组件</div>
</template>

<script setup>
import { useAttrs, watch } from 'vue';

const attrs = useAttrs();

// 监听 class 变化
watch(() => attrs.class, (newClass) => {
  console.log('父组件传递的 class 变化：', newClass);
});

// 访问 style
console.log('父组件传递的 style：', attrs.style);
</script>
```

#### 6、特殊场景：`class`/`style` 透传多层组件

当需要跨多层组件透传 `class`/`style` 时，只需在每一层组件中通过 `v-bind="$attrs"` 将 `$attrs` 透传给下一层即可，`class`/`style` 会自动合并。

#### 示例：三层组件透传

**父组件 → 中间组件 → 子组件**

1. **父组件**：传递 `class="parent"`、`style="color: red"`。
2. **中间组件**：用 `v-bind="$attrs"` 透传给子组件。
3. **子组件**：接收并合并自身的 `class`/`style`。

**中间组件（Middle.vue）**

vue

```vue
<template>
  <Child v-bind="$attrs" />
</template>

<script setup>
import Child from './Child.vue';
defineOptions({ inheritAttrs: false });
</script>
```

**子组件（Child.vue）**

vue

```vue
<template>
  <div class="child" style="font-size: 16px;">子组件</div>
</template>
```

**渲染结果**：

html

预览

```html
<div class="child parent" style="color: red; font-size: 16px;">子组件</div>
```

#### 7、常见问题与注意事项

1. **为什么 `$attrs` 中能看到 `class`/`style`，但根元素却没有？**
    
    → 可能是设置了 `inheritAttrs: false`，此时需要手动通过 `v-bind="$attrs"` 透传。
    
2. **`class`/`style` 合并顺序是什么？**
    
    → 父组件传递的 `class`/`style` 会和子组件自身的 `class`/`style` **合并**，而非覆盖：
    
    - `class`：子组件自身的 `class` 在前，父组件的在后（CSS 优先级由选择器决定，与顺序无关）。
    - `style`：相同的 CSS 属性，**子组件的会覆盖父组件的**（行内样式的优先级）。
3. **能否单独从 `$attrs` 中剔除 `class`/`style`？**
    
    → 可以通过解构赋值手动过滤，例如：
    
    vue
    
    ```vue
    <template>
      <button v-bind="{ ...attrs, class: undefined, style: undefined }">按钮</button>
    </template>
    
    <script setup>
    import { useAttrs } from 'vue';
    const attrs = useAttrs();
    </script>
    ```
    

## 总结

`class` 和 `style` 在 Vue 3 的 `$attrs` 中是**特殊且常用**的属性，核心特点是：

1. 会被自动包含在 `$attrs` 中，且支持合并。
2. 默认情况下会应用到子组件根元素，可通过 `inheritAttrs: false` 关闭。
3. 可通过 `v-bind="$attrs"` 手动透传，与显式绑定的 `class`/`style` 友好合并。

掌握这一特性，能更灵活地封装通用组件，同时保证 `class`/`style` 的透传和样式的可定制性。