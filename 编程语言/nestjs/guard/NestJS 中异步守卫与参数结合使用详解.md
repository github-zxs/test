---
aliases:
tags:
  - nestjs
---
# NestJS 中异步守卫与参数结合使用详解

你想了解 NestJS 中**异步守卫 + 带参数守卫**结合使用的完整配置和用法，这是实际项目中最常用的权限控制方案（比如从数据库异步校验权限、动态指定需要的角色）。我会通过一个完整的实战示例，覆盖「定义参数装饰器」「实现异步带参守卫」「应用守卫」全流程。

### 核心思路

异步带参数守卫的核心逻辑是：

1. 用自定义装饰器（如 `@Roles()`）给路由标记**权限参数**（元数据）；
    
2. 守卫中通过 `Reflector` 读取这些参数；
    
3. 执行**异步逻辑**（如查数据库、调用权限接口）验证用户是否符合参数要求；
    
4. 返回 `Promise<boolean>` 或抛出异常，决定请求是否放行。
    

### 完整实现步骤（实战：基于数据库的角色权限验证）

#### 步骤 1：定义参数装饰器（存储权限参数）

先创建装饰器，用于给路由标记「需要的角色/权限」：

```TypeScript
// src/decorators/roles.decorator.ts
import { SetMetadata } from '@nestjs/common';

// 自定义装饰器：接收多个角色参数，存储到元数据（key: 'required_roles'）
export const Roles = (...roles: string[]) => SetMetadata('required_roles', roles);
```

- 这里把元数据 key 命名为 `required_roles`（更语义化），避免和其他元数据冲突；
    
- 支持传入多个角色，比如 `@Roles('admin', 'editor')`。
    

#### 步骤 2：准备异步权限校验的服务（模拟数据库查询）

创建一个服务，模拟从数据库异步查询用户的角色/权限：

```TypeScript
// src/services/permission.service.ts
import { Injectable } from '@nestjs/common';

// 模拟用户表（实际项目中替换为数据库查询）
const mockUsers = [
  { id: 1, username: 'admin', roles: ['admin', 'super-admin'] },
  { id: 2, username: 'editor', roles: ['editor'] },
  { id: 3, username: 'guest', roles: ['guest'] },
];

@Injectable()
export class PermissionService {
  /**
   * 异步查询用户的角色列表
   * @param userId 用户ID
   * @returns 角色数组
   */
  async getUserRoles(userId: number): Promise<string[]> {
    // 模拟数据库异步查询（延迟 100ms）
    await new Promise(resolve => setTimeout(resolve, 100));
    const user = mockUsers.find(u => u.id === userId);
    return user ? user.roles : [];
  }

  /**
   * 异步校验用户是否拥有指定角色
   * @param userId 用户ID
   * @param requiredRoles 需要的角色列表
   * @returns 是否有权限
   */
  async checkUserRoles(userId: number, requiredRoles: string[]): Promise<boolean> {
    const userRoles = await this.getUserRoles(userId);
    // 判断用户是否至少拥有一个需要的角色
    return requiredRoles.some(role => userRoles.includes(role));
  }
}
```

#### 步骤 3：实现异步带参数守卫（核心）

创建守卫类，注入 `Reflector`（读参数）和 `PermissionService`（异步校验），实现带参数的异步权限验证：

```TypeScript
// src/guards/async-role.guard.ts
import { Injectable, CanActivate, ExecutionContext, ForbiddenException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { PermissionService } from '../services/permission.service';

@Injectable() // 必须加，否则无法注入依赖
export class AsyncRoleGuard implements CanActivate {
  // 注入 Reflector（读元数据）和 PermissionService（异步校验）
  constructor(
    private reflector: Reflector,
    private permissionService: PermissionService,
  ) {}

  // 核心方法：返回 Promise<boolean> 实现异步逻辑
  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. 读取路由/控制器上的角色参数（元数据）
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('required_roles', [
      context.getHandler(), // 路由方法（如 getAdminData）
      context.getClass(),   // 控制器类（如 AdminController）
    ]);

    // 2. 无角色参数 → 直接放行（无权限限制）
    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
    }

    // 3. 获取请求上下文（HTTP 场景）：解析用户ID
    const request = context.switchToHttp().getRequest();
    // 实际项目中：从 JWT/Token 解析 userId，这里模拟从请求头获取
    const userId = Number(request.headers['x-user-id']);
    if (!userId) {
      throw new ForbiddenException('未登录，请先登录');
    }

    // 4. 异步校验权限（核心：调用服务查数据库）
    const hasPermission = await this.permissionService.checkUserRoles(userId, requiredRoles);
    if (!hasPermission) {
      // 权限不足 → 抛 403 异常（比返回 false 更友好，有自定义提示）
      throw new ForbiddenException(
        `权限不足：需要拥有 ${requiredRoles.join('/')} 角色之一`,
      );
    }

    return true;
  }
}
```

关键说明：

- `canActivate` 方法标记为 `async`，返回 `Promise<boolean>`，支持异步逻辑；
    
- 依赖注入：守卫中可以正常注入其他服务（如 `PermissionService`），只需加 `@Injectable()`；
    
- 异常处理：优先抛 `ForbiddenException`（403）或 `UnauthorizedException`（401），而非仅返回 `false`（返回 `false` 会默认抛 403，但无自定义提示）；
    
- `getAllAndOverride`：路由级别的参数优先级高于控制器级别（比如控制器标 `@Roles('user')`，路由标 `@Roles('admin')`，则用 `admin`）。
    

#### 步骤 4：注册模块与应用守卫

##### 第一步：注册模块（将服务和守卫加入 DI 容器）

```TypeScript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AppController } from './app.controller';
import { PermissionService } from './services/permission.service';
import { AsyncRoleGuard } from './guards/async-role.guard';

@Module({
  controllers: [AppController],
  providers: [PermissionService, AsyncRoleGuard], // 注册服务和守卫
})
export class AppModule {}
```

##### 第二步：应用守卫（路由/控制器/全局级别）

###### 场景 1：路由级别（最常用）

给单个接口配置异步带参守卫：

```TypeScript
// src/app.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { AsyncRoleGuard } from './guards/async-role.guard';
import { Roles } from './decorators/roles.decorator';

@Controller()
export class AppController {
  // 场景 1：需要 admin/super-admin 角色（异步校验）
  @Get('admin/dashboard')
  @Roles('admin', 'super-admin') // 传入参数：需要的角色
  @UseGuards(AsyncRoleGuard)    // 应用异步守卫
  getAdminDashboard() {
    return { code: 200, msg: '管理员仪表盘数据', data: { stats: '1000 访问量' } };
  }

  // 场景 2：需要 editor 角色
  @Get('editor/article')
  @Roles('editor')
  @UseGuards(AsyncRoleGuard)
  getEditorArticle() {
    return { code: 200, msg: '编辑文章数据', data: { title: 'NestJS 异步守卫' } };
  }

  // 场景 3：无参数 → 所有人可访问
  @Get('public')
  @UseGuards(AsyncRoleGuard)
  getPublicData() {
    return { code: 200, msg: '公开数据' };
  }
}
```

###### 场景 2：控制器级别（所有路由生效）

给整个控制器的所有接口配置统一参数：

```TypeScript
@Controller('admin')
@Roles('admin') // 控制器级别：所有接口默认需要 admin 角色
@UseGuards(AsyncRoleGuard)
export class AdminController {
  // 继承控制器的 admin 角色要求
  @Get('users')
  getAdminUsers() {
    return { code: 200, data: [{ id: 1, username: 'admin' }] };
  }

  // 路由级别覆盖：需要 super-admin 角色
  @Get('settings')
  @Roles('super-admin')
  getAdminSettings() {
    return { code: 200, data: { theme: 'dark' } };
  }
}
```

###### 场景 3：全局级别（所有接口生效）

如果希望所有接口都应用该异步守卫，仅给需要权限的接口加 `@Roles()`：

```TypeScript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AsyncRoleGuard } from './guards/async-role.guard';
import { PermissionService } from './services/permission.service';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // 全局注册异步守卫（需手动注入依赖）
  const permissionService = app.get(PermissionService);
  app.useGlobalGuards(new AsyncRoleGuard(app.get(Reflector), permissionService));
  
  await app.listen(3000);
}
bootstrap();
```

- 全局注册时，无法通过 DI 自动注入 `Reflector` 和 `PermissionService`，需手动从容器中获取并传入；
    
- 全局守卫生效后，只需给需要权限的接口加 `@Roles()` 即可，无需重复加 `@UseGuards()`。
    

### 测试示例（HTTP 请求）

|   |   |   |
|---|---|---|
|请求接口|请求头（x-user-id）|结果|
|GET /admin/dashboard|1（admin 角色）|成功返回仪表盘数据|
|GET /admin/dashboard|2（editor 角色）|403 权限不足|
|GET /editor/article|2（editor 角色）|成功返回文章数据|
|GET /public|任意/无|成功返回公开数据|

### 关键注意事项

1. **依赖注入**：异步守卫必须加 `@Injectable()`，否则无法注入 `Reflector`、数据库服务等依赖；
    
2. **异步逻辑边界**：所有异步操作（如数据库查询、接口调用）都要放在 `canActivate` 中，且方法必须标记 `async`；
    
3. **异常优先**：验证失败时优先抛出异常（`ForbiddenException`/`UnauthorizedException`），而非返回 `false`，便于前端处理错误信息；
    
4. **元数据读取**：使用 `getAllAndOverride` 而非 `get`，确保路由级别参数覆盖控制器级别；
    
5. **性能优化**：如果权限校验频繁（如高频接口），可给异步逻辑加缓存（如 Redis），避免重复查数据库。
    

### 总结

1. 异步带参数守卫的核心是「`async canActivate` + `Reflector` 读参数 + 异步权限校验」；
    
2. 步骤：定义 `@Roles()` 装饰器存参数 → 守卫中注入 `Reflector` 和权限服务 → 异步校验用户是否匹配参数；
    
3. 应用方式：支持路由/控制器/全局级别，路由级别优先级最高；
    
4. 核心优势：既能动态接收权限参数，又能处理数据库查询等异步逻辑，适配真实项目的权限需求。
    

核心关键点：`async/await` 让守卫支持异步操作，`Reflector` 让守卫能读取装饰器传入的参数，两者结合是实现动态异步权限控制的核心。

# NestJS 全局守卫配置详解

你想要在 NestJS 项目中配置**全局守卫**，全局守卫会作用于项目中所有的路由/控制器，无需在每个控制器/接口上单独声明，这是实现全局权限校验、请求拦截等功能的常用方式。

### 配置全局守卫的两种常用方法

NestJS 提供了两种主流方式配置全局守卫，我会分别说明并给出完整示例。

#### 方法 1：通过 `app.useGlobalGuards()` 直接注册（最常用）

这种方式适合在项目入口文件（`main.ts`）中直接注册，简单直观。

##### 步骤 1：创建自定义全局守卫

先创建一个基础的全局守卫示例（比如权限校验守卫）：

```TypeScript
// src/guards/global-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Observable } from 'rxjs';

@Injectable()
export class GlobalAuthGuard implements CanActivate {
  // 核心方法：返回 true 表示允许访问，false 表示拒绝
  canActivate(
    context: ExecutionContext,
  ): boolean | Promise<boolean> | Observable<boolean> {
    // 获取请求对象（HTTP 场景下）
    const request = context.switchToHttp().getRequest();
    
    // 这里可以写你的全局校验逻辑，比如：
    // 1. 校验 token 是否存在
    // 2. 校验用户权限
    // 3. 记录请求日志等
    console.log('全局守卫触发：', request.url);
    
    // 示例：简单返回 true 允许访问（实际项目中替换为你的业务逻辑）
    return true;
  }
}
```

##### 步骤 2：在 `main.ts` 中注册全局守卫

```TypeScript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { GlobalAuthGuard } from './guards/global-auth.guard';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // 核心：注册全局守卫
  app.useGlobalGuards(new GlobalAuthGuard());

  await app.listen(3000);
  console.log(`应用运行在: http://localhost:3000`);
}
bootstrap();
```

#### 方法 2：通过模块 `providers` 注册（支持依赖注入）

如果你的全局守卫需要依赖其他服务（比如 `ConfigService`、`UserService`），方法 1 无法直接注入依赖，此时需要通过模块注册：

##### 步骤 1：在根模块（`AppModule`）中配置

```TypeScript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { GlobalAuthGuard } from './guards/global-auth.guard';
import { ConfigModule } from '@nestjs/config'; // 示例依赖

@Module({
  imports: [ConfigModule.forRoot()], // 示例：导入配置模块
  providers: [
    {
      // 核心：使用 APP_GUARD 通过 `APP_GUARD` 令牌将守卫注册为全局提供者，兼容依赖注入，(是推荐方案)。
      provide: APP_GUARD,
      useClass: GlobalAuthGuard, // 指定守卫类   （自动支持依赖注入）
    },
  ],
})
export class AppModule {}
```

##### 步骤 2：在守卫中注入依赖（示例）

```TypeScript
// src/guards/global-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { ConfigService } from '@nestjs/config'; // 示例依赖

@Injectable()
export class GlobalAuthGuard implements CanActivate {
  // 注入 ConfigService
  constructor(private readonly configService: ConfigService) {}

  canActivate(context: ExecutionContext): boolean {
    // 使用注入的配置服务
    const secret = this.configService.get('JWT_SECRET');
    console.log('全局守卫读取配置：', secret);
    
    return true;
  }
}
```

#### 方法3:  全局带参数注入(手动注入依赖)
```TypeScript
// src/main.ts
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { AsyncRoleGuard } from './guards/async-role.guard';
import { PermissionService } from './services/permission.service';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  
  // 全局注册异步守卫（需手动注入依赖）
  const permissionService = app.get(PermissionService);
  app.useGlobalGuards(new AsyncRoleGuard(app.get(Reflector), permissionService));
  
  await app.listen(3000);
}
bootstrap();
```

#### 方法4:  模块化全局守卫（局部模块全局生效）
若需让守卫仅在**特定模块**下全局生效（而非全应用），可结合模块导出 + `APP_GUARD`。
##### 示例：用户模块内的全局守卫

typescript

运行

```typescript
// src/modules/user/user.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';

@Injectable()
export class UserGlobalGuard implements CanActivate {
  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest();
    // 仅对 /user 前缀的路由生效（示例逻辑）
    if (request.url.startsWith('/user')) {
      return !!request.user; // 校验用户是否登录
    }
    return true; // 非 /user 路由直接通过
  }
}

// src/modules/user/user.module.ts
import { Module } from '@nestjs/common';
import { APP_GUARD } from '@nestjs/core';
import { UserGlobalGuard } from './user.guard';
import { UserController } from './user.controller';

@Module({
  controllers: [UserController],
  providers: [
    {
      provide: APP_GUARD,
      useClass: UserGlobalGuard,
    },
  ],
})
export class UserModule {}
```

##### 说明

- 此守卫会对**整个应用**生效，但可通过逻辑限制仅作用于目标模块的路由。
- 若需严格隔离，可结合 `@Guard()` 装饰器 + 模块路由前缀。

### 关键说明

1. **适用场景**：
    - 方法 1：适合无依赖的简单全局守卫，配置简单；
	-  方法 2：适合需要依赖注入的复杂守卫（推荐在实际项目中使用）。
    
2. **执行顺序**：全局守卫会在控制器守卫、接口守卫之前执行，是请求进入业务逻辑的第一道全局拦截。
    
3. **异常处理**：如果校验不通过（返回 `false` 或抛出异常），NestJS 会自动返回 403 Forbidden 响应，你也可以手动抛出自定义异常：
    ```TypeScript
    import { ForbiddenException } from '@nestjs/common';
    
    // 在 canActivate 中抛出
    if (!token) {
      throw new ForbiddenException('无访问权限，请先登录');
    }
    ```

4.  **优先级控制**: 多个全局守卫的执行顺序由注册顺序决定（先注册先执行）；若需自定义优先级，可结合 `@SetMetadata()` + 元数据排序。
### 总结

1. 配置全局守卫有两种核心方式：`app.useGlobalGuards()`（简单无依赖）和模块注册 `APP_GUARD`（支持依赖注入）；
    
2. 全局守卫的核心是实现 `CanActivate` 接口并编写 `canActivate` 方法，返回 `true/false` 控制请求是否通过；
    
3. 若守卫需要依赖其他服务（如配置、数据库），优先使用模块注册的方式。