你想要在 NestJS 中结合策略类和守卫实现更复杂的认证逻辑，核心思路是：**基于 Passport 策略实现基础身份验证，通过自定义守卫扩展权限校验、多条件判断、多策略组合等复杂逻辑**。下面我会从「自定义守卫扩展策略」「多策略组合」「动态权限校验」「全局守卫 + 排除路由」四个维度，带你实现复杂认证场景。

### 一、核心前提

先明确基础概念：

- **策略类（Strategy）**：负责「身份验证」（比如验证 JWT 是否合法、账号密码是否正确），核心是 `validate` 方法，只做 “是否是合法用户” 的判断。
- **守卫（Guard）**：负责「权限控制 / 复杂逻辑」（比如判断用户是否有某个角色、是否能访问某个接口、是否满足特定条件），核心是 `canActivate` 方法，决定 “合法用户是否能做某件事”。
- 复杂认证 = 策略（验身份） + 守卫（验权限 / 条件）。

### 二、场景 1：自定义守卫扩展 JWT 策略（角色 + 权限校验）

最常见的复杂场景：用户登录后（JWT 验证通过），还需要判断用户角色（如 Admin / 普通用户）、是否有接口访问权限。

#### 步骤 1：先定义基础 JWT 策略（复用之前的）

确保 `JwtStrategy` 能正常验证 JWT，并返回包含 `roles`/`permissions` 字段的用户信息：

typescript

运行

```typescript
// src/auth/strategies/jwt.strategy.ts
async validate(payload: any) {
  const user = await this.authService.findUserById(payload.sub);
  if (!user) throw new UnauthorizedException();
  // 返回包含角色/权限的用户信息（关键：给守卫提供判断依据）
  return {
    id: user.id,
    username: user.username,
    roles: user.roles, // 如 ['admin', 'user']
    permissions: user.permissions, // 如 ['user:read', 'user:write']
  };
}
```

#### 步骤 2：自定义守卫（结合策略 + 角色校验）

创建 `RolesGuard`，继承 `AuthGuard`（复用 JWT 策略的身份验证），再扩展角色判断：

typescript

运行

```typescript
// src/auth/guards/roles.guard.ts
import { CanActivate, ExecutionContext, Injectable, UnauthorizedException } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';

// 自定义装饰器：标记接口需要的角色
export const Roles = (...roles: string[]) => Reflector.define('roles', roles);

@Injectable()
export class RolesGuard extends AuthGuard('jwt') {
  // 注入 Reflector：用于读取装饰器标记的角色
  constructor(private reflector: Reflector) {
    super(); // 调用父类 AuthGuard 的构造函数，先执行 JWT 策略验证
  }

  // 核心方法：先执行父类的 canActivate（JWT 验证），再扩展角色校验
  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 1. 先执行 JWT 策略的身份验证（父类逻辑）
    const isAuthenticated = await super.canActivate(context);
    if (!isAuthenticated) {
      throw new UnauthorizedException('身份验证失败');
    }

    // 2. 获取当前请求上下文
    const request = context.switchToHttp().getRequest();
    const user = request.user; // JWT 策略 validate 返回的用户信息

    // 3. 读取装饰器标记的「需要的角色」
    const requiredRoles = this.reflector.getAllAndOverride<string[]>('roles', [
      context.getHandler(), // 优先读取接口级别的装饰器
      context.getClass(),   // 其次读取控制器级别的装饰器
    ]);

    // 4. 如果接口不需要角色，直接通过
    if (!requiredRoles || requiredRoles.length === 0) {
      return true;
    }

    // 5. 校验用户角色是否匹配
    const hasRole = user.roles.some(role => requiredRoles.includes(role));
    if (!hasRole) {
      throw new UnauthorizedException(`需要角色：${requiredRoles.join(', ')}，当前角色：${user.roles.join(', ')}`);
    }

    return true;
  }
}
```

#### 步骤 3：使用自定义守卫

在控制器中结合装饰器使用，实现 “JWT 验证 + 角色校验”：

typescript

运行

```typescript
// src/users/users.controller.ts
import { Controller, Get, UseGuards } from '@nestjs/common';
import { RolesGuard, Roles } from '../auth/guards/roles.guard';

@Controller('users')
export class UsersController {
  // 普通用户可访问：JWT 验证 + 角色为 user/admin
  @Get()
  @UseGuards(RolesGuard)
  @Roles('user', 'admin')
  getAllUsers() {
    return [{ id: 1, username: 'test' }];
  }

  // 仅管理员可访问：JWT 验证 + 角色为 admin
  @Get('admin-only')
  @UseGuards(RolesGuard)
  @Roles('admin')
  getAdminData() {
    return { msg: '仅管理员可见' };
  }
}
```

### 三、场景 2：多策略组合（JWT + 本地策略）

复杂场景：某些接口支持「账号密码登录」和「JWT 登录」两种方式，需要组合多个策略。

#### 步骤 1：定义多个策略

先确保 `LocalStrategy`（账号密码）和 `JwtStrategy`（JWT）都已注册到 `AuthModule`。

#### 步骤 2：自定义组合守卫

创建 `MultiAuthGuard`，同时支持多个策略：

typescript

运行

```typescript
// src/auth/guards/multi-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class MultiAuthGuard implements CanActivate {
  // 注入两个基础守卫
  constructor(
    private readonly localGuard: AuthGuard('local'),
    private readonly jwtGuard: AuthGuard('jwt'),
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    try {
      // 先尝试 JWT 验证
      return await this.jwtGuard.canActivate(context);
    } catch (jwtError) {
      try {
        // JWT 验证失败，尝试本地账号密码验证
        return await this.localGuard.canActivate(context);
      } catch (localError) {
        // 两个策略都失败，认证失败
        return false;
      }
    }
  }
}
```

#### 步骤 3：注册并使用组合守卫

typescript

运行

```typescript
// src/auth/auth.module.ts
providers: [
  AuthService,
  JwtStrategy,
  LocalStrategy,
  MultiAuthGuard, // 注册组合守卫
  // 注入基础守卫（供组合守卫使用）
  { provide: 'LOCAL_GUARD', useClass: AuthGuard('local') },
  { provide: 'JWT_GUARD', useClass: AuthGuard('jwt') },
],

// src/app.controller.ts
@Post('multi-login')
@UseGuards(MultiAuthGuard)
multiLogin(req: any) {
  return { user: req.user, msg: '认证成功' };
}
```

### 四、场景 3：动态策略选择（根据请求参数切换策略）

更复杂的场景：根据请求头 / 参数动态选择认证策略（比如 `mode=jwt` 用 JWT 策略，`mode=apikey` 用 APIKey 策略）。

#### 步骤 1：定义 APIKey 策略（新增）

typescript

运行

```typescript
// src/auth/strategies/apikey.strategy.ts
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-headerapikey';
import { AuthService } from '../auth.service';

@Injectable()
export class ApiKeyStrategy extends PassportStrategy(Strategy, 'apikey') {
  constructor(private authService: AuthService) {
    super({ header: 'X-API-KEY', prefix: '' }, true, async (apiKey, done) => {
      const isValid = await this.authService.validateApiKey(apiKey);
      if (!isValid) {
        return done(new UnauthorizedException('无效的 API Key'), false);
      }
      return done(null, { apiKey, role: 'api_user' });
    });
  }
}
```

#### 步骤 2：自定义动态守卫

typescript

运行

```typescript
// src/auth/guards/dynamic-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Injectable()
export class DynamicAuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    // 1. 从请求头/参数获取策略类型
    const authMode = request.headers['x-auth-mode'] || 'jwt';

    // 2. 动态选择守卫
    let guard: AuthGuard;
    switch (authMode) {
      case 'jwt':
        guard = new (AuthGuard('jwt'))();
        break;
      case 'apikey':
        guard = new (AuthGuard('apikey'))();
        break;
      case 'local':
        guard = new (AuthGuard('local'))();
        break;
      default:
        throw new UnauthorizedException('不支持的认证方式');
    }

    // 3. 执行选中的守卫
    return guard.canActivate(context) as Promise<boolean>;
  }
}
```

#### 步骤 3：使用动态守卫

typescript

运行

```typescript
@Get('dynamic-auth')
@UseGuards(DynamicAuthGuard)
dynamicAuth(req: any) {
  return { user: req.user, mode: req.headers['x-auth-mode'] };
}
```

### 五、场景 4：全局守卫 + 排除路由（精细化控制）

复杂系统中，通常需要 “全局认证” + “部分路由免认证”（如登录、注册接口）。

#### 步骤 1：自定义全局守卫（支持排除路由）

typescript

运行

```typescript
// src/auth/guards/global-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { Reflector } from '@nestjs/core';
import { AuthGuard } from '@nestjs/passport';

// 自定义装饰器：标记路由无需认证
export const Public = () => Reflector.define('isPublic', true);

@Injectable()
export class GlobalAuthGuard extends AuthGuard('jwt') {
  constructor(private reflector: Reflector) {
    super();
  }

  canActivate(context: ExecutionContext): Promise<boolean> | boolean {
    // 1. 检查是否是公开路由
    const isPublic = this.reflector.getAllAndOverride<boolean>('isPublic', [
      context.getHandler(),
      context.getClass(),
    ]);
    if (isPublic) {
      return true; // 公开路由直接通过
    }

    // 2. 非公开路由执行 JWT 验证
    return super.canActivate(context) as Promise<boolean>;
  }
}
```

#### 步骤 2：注册全局守卫

typescript

运行

```typescript
// src/app.module.ts
providers: [
  {
    provide: APP_GUARD,
    useClass: GlobalAuthGuard,
  },
],
```

#### 步骤 3：标记公开路由

typescript

运行

```typescript
// src/auth/auth.controller.ts
@Controller('auth')
export class AuthController {
  // 登录接口免认证
  @Public()
  @Post('login')
  @UseGuards(AuthGuard('local'))
  login(@Body() loginDto: LoginDto) {
    return this.authService.login(loginDto);
  }

  // 注册接口免认证
  @Public()
  @Post('register')
  register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }
}
```

### 总结

实现 NestJS 复杂认证逻辑的核心要点：

1. **策略做 “身份验证”**：负责校验用户是否合法（JWT / 账号密码 / APIKey），`validate` 方法返回用户核心信息。
2. **守卫做 “权限 / 逻辑扩展”**：
    - 自定义守卫继承 `AuthGuard` 可复用策略的身份验证，再扩展角色 / 权限校验；
    - 多策略组合 / 动态策略选择通过自定义守卫的 `canActivate` 实现；
    - 全局守卫 +`Reflector` 可实现 “默认认证 + 部分路由免认证”。
3. **Reflector 是关键**：用于读取装饰器标记的元数据（如需要的角色、是否公开路由），实现灵活的动态控制。

这种分层设计（策略管身份，守卫管权限）既符合 NestJS 的模块化思想，也能适配从简单到复杂的各种认证场景。