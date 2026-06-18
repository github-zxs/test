# NestJS 构造函数依赖注入：何时加 @Inject() 详解

你想知道在 NestJS 的构造函数依赖注入中，是否必须添加 `@Inject()` 装饰器，以及在什么情况下需要用、什么情况下可以不用，对吧？

简单来说：**绝大多数场景下不需要加** **`@Inject()`****，只有特定场景才必须加**。下面我会详细拆解这个问题。

### 1. 不需要加 `@Inject()` 的场景（主流场景）

当你的依赖是**类本身**（比如 `UserService`、`OrderService` 这类用 `@Injectable()` 装饰的服务类），并且使用 TypeScript 开发时，NestJS 会通过 **TypeScript 的类型元数据** 自动识别你要注入的依赖，此时完全不需要 `@Inject()`。

这也是 NestJS 官方最推荐的写法，简洁且易读：

```TypeScript
import { Controller, Get, Injectable } from '@nestjs/common';

@Injectable()
export class UserService {
  getUser() {
    return { id: 1, name: '张三' };
  }
}

@Controller('users')
export class UserController {
  // ✅ 无需 @Inject()，Nest 自动识别 UserService 类型
  constructor(private readonly userService: UserService) {}

  @Get()
  getUsers() {
    return this.userService.getUser();
  }
}
```

### 2. 必须加 `@Inject()` 的场景

以下 4 种情况，必须显式使用 `@Inject()` 装饰器，否则 NestJS 无法识别要注入的依赖：

#### 场景 1：注入非类依赖（自定义 Token）

当依赖不是类，而是通过自定义字符串/符号（Token）注册的提供者时，必须用 `@Inject(Token)` 指定要注入的标识。

```TypeScript
import { Controller, Get, Injectable, Inject, Module } from '@nestjs/common';

// 1. 定义自定义 Token
export const DB_CONFIG = 'DB_CONFIG';

// 2. 注册自定义提供者
@Module({
  providers: [
    {
      provide: DB_CONFIG,
      useValue: { host: 'localhost', port: 3306 },
    },
  ],
})
export class AppModule {}

// 3. 构造函数注入时必须加 @Inject(Token)
@Controller()
export class ConfigController {
  // ❗ 必须加 @Inject(DB_CONFIG)，否则无法识别
  constructor(@Inject(DB_CONFIG) private readonly dbConfig: any) {}

  @Get('config')
  getConfig() {
    return this.dbConfig;
  }
}
```

#### 场景 2：处理循环依赖

当两个类互相依赖（A 依赖 B，B 依赖 A），构造函数注入时需要配合 `forwardRef()` 和 `@Inject()`：

```TypeScript
import { Injectable, Inject, forwardRef } from '@nestjs/common';
import { OrderService } from './order.service';

@Injectable()
export class UserService {
  // ❗ 循环依赖时必须加 @Inject(forwardRef(...))
  constructor(
    @Inject(forwardRef(() => OrderService))
    private readonly orderService: OrderService,
  ) {}
}
```

#### 场景 3：使用 JavaScript 开发（无 TypeScript 类型）

如果你的项目不用 TypeScript，而是纯 JavaScript，NestJS 无法通过类型元数据识别依赖，必须加 `@Inject()`：

```JavaScript
// user.controller.js（纯 JS 写法）
import { Controller, Get, Inject } from '@nestjs/common';
import { UserService } from './user.service.js';

@Controller('users')
export class UserController {
  // ❗ 纯 JS 必须加 @Inject(UserService)
  constructor(@Inject(UserService) userService) {
    this.userService = userService;
  }

  @Get()
  getUsers() {
    return this.userService.getUser();
  }
}
```

#### 场景 4：注入抽象类/接口（TypeScript 接口无运行时元数据）

TypeScript 接口仅存在于编译阶段，运行时会被擦除，因此注入基于接口的抽象类时，需要用 `@Inject()` 指定具体实现：

```TypeScript
// 1. 定义抽象类（代替接口，因为接口无运行时元数据）
export abstract class BaseService {
  abstract getData(): any;
}

// 2. 实现抽象类
@Injectable()
export class UserService extends BaseService {
  getData() {
    return { name: '张三' };
  }
}

// 3. 注册提供者
@Module({
  providers: [
    {
      provide: BaseService, // 抽象类作为 Token
      useClass: UserService, // 具体实现
    },
  ],
})
export class AppModule {}

// 4. 构造函数注入时必须加 @Inject(BaseService)
@Controller()
export class TestController {
  // ❗ 必须加 @Inject(BaseService)
  constructor(@Inject(BaseService) private readonly baseService: BaseService) {}
}
```

### 总结

1. **默认情况（TypeScript + 类依赖）**：不需要加 `@Inject()`，NestJS 自动通过类型元数据识别依赖；
    
2. **必须加** **`@Inject()`** **的场景**：注入自定义 Token 依赖、处理循环依赖、纯 JavaScript 开发、注入抽象类/接口；
    
3. 核心原则：`@Inject()` 的作用是**显式指定要注入的依赖标识**，当 Nest 无法自动识别标识时，就需要手动加。