---
tags:
  - nestjs/context
  - nestjs
---
在 NestJS 中，**没有内置的 “上下文工具”**，但框架提供了多层级的上下文获取能力（如请求上下文、执行上下文），结合社区最佳实践和内置 API，可灵活实现上下文管理。以下是 NestJS 中获取 / 管理上下文的核心方式，从「框架原生能力」到「自定义工具封装」分层讲解：

### 一、NestJS 原生上下文 API（核心）

NestJS 提供了多个内置类 / 装饰器，用于获取不同场景的上下文（如 HTTP 请求、微服务、WebSocket 等），是最基础的上下文获取方式。

#### 1. ExecutionContext（执行上下文）

`ExecutionContext` 是 NestJS 抽象的**通用执行上下文**，适配 HTTP、微服务、WebSocket、GraphQL 等所有传输层，可在**守卫、拦截器、过滤器、管道**中获取上下文信息。

##### 核心方法：

|方法|作用|
|---|---|
|`getType()`|获取上下文类型（`http`/`rpc`/`ws`/`graphql`）|
|`switchToHttp()`|切换到 HTTP 上下文（返回 `HttpArgumentsHost`）|
|`switchToRpc()`|切换到微服务 RPC 上下文|
|`switchToWs()`|切换到 WebSocket 上下文|

##### 示例（拦截器中获取 HTTP 上下文）：

typescript

运行

```typescript
// src/common/interceptors/logger.interceptor.ts
import { Injectable, NestInterceptor, ExecutionContext, CallHandler } from '@nestjs/common';
import { Observable } from 'rxjs';
import { tap } from 'rxjs/operators';

@Injectable()
export class LoggerInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    // 1. 判断上下文类型
    const ctxType = context.getType(); // "http"
    if (ctxType !== 'http') return next.handle();

    // 2. 切换到HTTP上下文，获取req/res
    const httpCtx = context.switchToHttp();
    const request = httpCtx.getRequest();
    const response = httpCtx.getResponse();

    // 3. 获取请求信息（原生Express/Fastify对象）
    const { method, url, ip } = request;
    const now = Date.now();

    // 4. 响应完成后打印日志
    return next.handle().pipe(
      tap(() => {
        console.log(`[${method}] ${url} | ${ip} | ${Date.now() - now}ms`);
      }),
    );
  }
}
```

#### 2. @Request () / @Response () 装饰器（控制器专用）

在控制器中，可通过 NestJS 提供的装饰器直接获取 HTTP 请求 / 响应对象（封装了 Express/Fastify 的原生对象）：

typescript

运行

```typescript
// src/user/user.controller.ts
import { Controller, Get, Request, Response, Ip } from '@nestjs/common';
import { Request as ExpressRequest, Response as ExpressResponse } from 'express';

@Controller('users')
export class UserController {
  @Get()
  getUsers(
    // 方式1：装饰器获取原生Request
    @Request() req: ExpressRequest,
    // 方式2：装饰器获取原生Response
    @Response() res: ExpressResponse,
    // 方式3：快捷装饰器获取IP
    @Ip() ip: string,
  ) {
    console.log('请求URL:', req.url);
    console.log('请求IP:', ip);
    // 注意：使用@Response()后需手动send响应，否则Nest不会自动处理
    res.status(200).json({ data: ['user1', 'user2'] });
  }
}
```

#### 3. ContextType 类型适配

针对不同传输层（如 GraphQL、微服务），NestJS 提供了对应的上下文获取方式：

- **GraphQL**：使用 `@Args()`/`@Context()` 装饰器获取 GraphQL 上下文；
- **微服务**：使用 `@MessagePattern()` 结合 `RpcException` 获取 RPC 上下文；
- **WebSocket**：使用 `@WebSocketServer()`/`@ConnectedSocket()` 获取 WS 上下文。

### 二、NestJS 官方推荐的上下文工具：AsyncLocalStorage

如前所述，NestJS 本身不提供 “上下文传递工具”，但**官方文档推荐使用 Node.js 内置的 `AsyncLocalStorage`** 实现跨异步链的上下文传递（如请求 ID、用户信息）。

#### 封装通用上下文工具（Nest 风格）

结合 NestJS 的依赖注入（DI），封装可全局复用的上下文工具（比原生 ALS 更贴合 Nest 生态）：

typescript

运行

```typescript
// src/common/context/request.context.ts
import { Injectable, Scope } from '@nestjs/common';
import { AsyncLocalStorage } from 'async_hooks';
import { v4 as uuidv4 } from 'uuid';

// 定义请求上下文类型
export interface RequestContext {
  requestId: string;
  userId?: string;
  ip: string;
  [key: string]: any;
}

/**
 * 请求上下文工具（全局单例，请求级隔离）
 * Scope.DEFAULT：全局单例（推荐），Scope.REQUEST：请求级实例（性能略低）
 */
@Injectable({ scope: Scope.DEFAULT })
export class RequestContextService {
  private readonly als = new AsyncLocalStorage<RequestContext>();

  /**
   * 启动上下文（通常在中间件中调用）
   */
  createContext(req: any): void {
    const context: RequestContext = {
      requestId: uuidv4(),
      ip: req.ip || req.ips.join(',') || 'unknown',
      userId: req.headers['x-user-id'] as string,
    };
    // 启动ALS上下文，返回一个"绑定上下文的函数"
    return this.als.run(context, () => {});
  }

  /**
   * 获取当前请求上下文（任意层级可调用）
   */
  getContext(): RequestContext | undefined {
    return this.als.getStore();
  }

  /**
   * 运行带上下文的函数（手动绑定）
   */
  runWithContext<T>(context: RequestContext, callback: () => T): T {
    return this.als.run(context, callback);
  }
}
```

### 三、社区成熟的上下文工具库

如果不想手动封装，可使用 NestJS 社区维护的上下文工具库，开箱即用：

#### 1. @nestjs/core 内置的 REQUEST 提供者（请求级注入）

NestJS 提供了 `REQUEST` 令牌，可在**请求级作用域**的服务中注入原生请求对象（无需手动传参）：

typescript

运行

```typescript
// src/user/user.service.ts
import { Injectable, Scope, Inject } from '@nestjs/common';
import { REQUEST } from '@nestjs/core';
import { Request } from 'express';

// 注意：Scope.REQUEST 表示每个请求创建一个新实例（性能略低）
@Injectable({ scope: Scope.REQUEST })
export class UserService {
  constructor(
    // 注入请求对象（仅请求级作用域可用）
    @Inject(REQUEST) private readonly request: Request,
  ) {}

  async getCurrentUser() {
    // 从请求头获取用户ID
    const userId = this.request.headers['x-user-id'] as string;
    return { userId, requestId: this.request.headers['x-request-id'] };
  }
}
```

**缺点**：`Scope.REQUEST` 会导致服务每次请求创建新实例，频繁创建 / 销毁会有性能损耗，不推荐高频服务使用。

#### 2. nestjs-context（社区库）

`nestjs-context` 是专门为 NestJS 设计的上下文管理库，基于 `AsyncLocalStorage` 封装，简化了上下文传递：

##### 安装：

bash

运行

```bash
npm install nestjs-context
```

##### 使用：

typescript

运行

```typescript
// 1. 注册模块
import { Module } from '@nestjs/common';
import { ContextModule } from 'nestjs-context';

@Module({
  imports: [ContextModule.forRoot()],
})
export class AppModule {}

// 2. 中间件中设置上下文
import { ContextService } from 'nestjs-context';
import { NestMiddleware } from '@nestjs/common';

export class ContextMiddleware implements NestMiddleware {
  constructor(private readonly contextService: ContextService) {}

  use(req: any, res: any, next: () => void) {
    this.contextService.set('requestId', uuidv4());
    this.contextService.set('userId', req.headers['x-user-id']);
    next();
  }
}

// 3. 任意服务中获取上下文
import { Injectable } from '@nestjs/common';
import { ContextService } from 'nestjs-context';

@Injectable()
export class UserService {
  constructor(private readonly contextService: ContextService) {}

  async getInfo() {
    const requestId = this.contextService.get('requestId');
    const userId = this.contextService.get('userId');
    return { requestId, userId };
  }
}
```

#### 3. nestjs-cls（推荐，功能最全）

`nestjs-cls` 是基于 `AsyncLocalStorage` 封装的 NestJS 专用上下文库，支持：

- 自动绑定 HTTP/GraphQL/ 微服务上下文；
- 请求级 DI 注入；
- 无需手动管理 ALS 生命周期；
- 兼容 Express/Fastify。

##### 核心优势：

- 无需手动写中间件，通过模块配置自动启动上下文；
- 支持 `ClsService` 全局注入，任意层级获取上下文；
- 支持装饰器（`@ClsInject()`）注入上下文数据。

##### 快速使用：

bash

运行

```bash
npm install nestjs-cls
```

typescript

运行

```typescript
// 1. 注册ClsModule
import { Module } from '@nestjs/common';
import { ClsModule } from 'nestjs-cls';

@Module({
  imports: [
    ClsModule.forRoot({
      // 自动为HTTP请求创建上下文
      global: true,
      middleware: {
        mount: true,
        // 自定义上下文初始化
        setup: (cls, req) => {
          cls.set('requestId', uuidv4());
          cls.set('userId', req.headers['x-user-id']);
        },
      },
    }),
  ],
})
export class AppModule {}

// 2. 服务中获取上下文
import { Injectable } from '@nestjs/common';
import { ClsService } from 'nestjs-cls';

@Injectable()
export class UserService {
  constructor(private readonly cls: ClsService) {}

  async getUsers() {
    const requestId = this.cls.get('requestId');
    const userId = this.cls.get('userId');
    console.log(`[${requestId}] 用户${userId}查询列表`);
    return [{ id: 1, name: '张三' }];
  }
}
```

### 四、不同上下文方案对比

| 方案                     | 适用场景                        | 优点                  | 缺点                |
| ---------------------- | --------------------------- | ------------------- | ----------------- |
| ExecutionContext + 装饰器 | 控制器 / 守卫 / 拦截器中获取单次请求信息     | 原生支持，无额外依赖          | 无法跨异步链传递（如服务→数据库） |
| AsyncLocalStorage 手动封装 | 跨异步链传递上下文（请求 ID、用户信息）       | 轻量，原生 Node.js API   | 需手动管理中间件 / 生命周期   |
| nestjs-context         | 简单的上下文传递                    | 开箱即用，API 简洁         | 功能较基础，兼容性一般       |
| nestjs-cls             | 生产级上下文管理（HTTP/GraphQL/ 微服务） | 功能全，自动绑定，兼容 Nest 生态 | 第三方依赖，略增加包体积      |

### 五、最佳实践

1. **基础场景**：仅需在控制器 / 拦截器中获取请求信息 → 使用 `ExecutionContext` + `@Request()` 装饰器；
2. **跨异步链传递**：需要在服务、数据库、日志中传递请求 ID / 用户信息 → 使用 `nestjs-cls`（推荐）或手动封装 `AsyncLocalStorage`；
3. **性能敏感场景**：避免使用 `Scope.REQUEST` 作用域的服务，优先用 `AsyncLocalStorage` 全局单例；
4. **日志链路追踪**：结合 `nestjs-cls` + 日志库（如 winston），自动在日志中注入 `requestId`。

### 总结

NestJS 本身没有内置的 “上下文工具”，但提供了：

- 原生 API（`ExecutionContext`、`@Request()`）获取单次请求上下文；
- 推荐使用 Node.js `AsyncLocalStorage` 实现跨异步链的上下文传递；
- 社区库（如 `nestjs-cls`）封装了更贴合 Nest 生态的上下文管理能力，是生产环境的首选。

核心原则：**轻量场景用原生 API，跨异步链用 ALS/CLS，避免过度封装**。