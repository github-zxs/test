---
aliases:
tags:
  - nestjs
---
# NestJS 全局守卫排序：框架原生、自定义容器与依赖注入

在 NestJS 中，除了「元数据 + 反射器排序」，还有多种更直接、更贴合框架设计的方式实现全局守卫排序，核心可分为**框架原生机制**、**自定义执行容器**、**依赖注入控制**三大类，以下是具体方案（附场景适配和完整示例）：

### 一、框架原生：利用注册顺序/数组顺序（最基础）

NestJS 对全局守卫的执行顺序有明确规则：

- 通过 `app.useGlobalGuards(guard1, guard2, guard3)` 注册时，**数组顺序即执行顺序**（先传的先执行）；
    
- 通过 `APP_GUARD` 令牌注册多个守卫时，**providers 数组顺序即执行顺序**（Nest 按数组遍历顺序实例化并执行）。
    

#### 示例1：`app.useGlobalGuards` 数组排序

```TypeScript
// main.ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule);
  // 执行顺序：Guard1 → Guard2 → Guard3
  app.useGlobalGuards(new Guard1(), new Guard2(), new Guard3());
  await app.listen(3000);
}
```

#### 示例2：`APP_GUARD` 多守卫数组排序

```TypeScript
// app.module.ts
@Module({
  providers: [
    // 执行顺序：Guard1 → Guard2 → Guard3
    { provide: APP_GUARD, useClass: Guard1 },
    { provide: APP_GUARD, useClass: Guard2 },
    { provide: APP_GUARD, useClass: Guard3 },
  ],
})
export class AppModule {}
```

#### 核心特点

- ✅ 零额外代码，纯框架原生机制；
    
- ❌ 硬编码顺序，无法动态调整；
    
- ❌ 仅支持「全局固定顺序」，无法按路由/场景差异化。
    

### 二、自定义执行容器：守卫聚合器（推荐大型项目）

创建一个「根守卫」作为所有全局守卫的执行入口，在其中手动控制子守卫的执行顺序（本质是「单一入口 + 手动排序」），是元数据排序的简化替代方案。

#### 步骤1：定义子守卫（业务逻辑解耦）

```TypeScript
// guards/auth.guard.ts（登录态校验）
@Injectable()
export class AuthGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    console.log('执行：登录态校验');
    return true;
  }
}

// guards/permission.guard.ts（权限校验）
@Injectable()
export class PermissionGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    console.log('执行：权限校验');
    return true;
  }
}

// guards/rate-limit.guard.ts（限流校验）
@Injectable()
export class RateLimitGuard implements CanActivate {
  async canActivate(context: ExecutionContext): Promise<boolean> {
    console.log('执行：限流校验');
    return true;
  }
}
```

#### 步骤2：创建守卫聚合器（控制顺序）

```TypeScript
// guards/root-guard.ts
@Injectable()
export class RootGuard implements CanActivate {
  // 注入所有子守卫
  constructor(
    private readonly authGuard: AuthGuard,
    private readonly permissionGuard: PermissionGuard,
    private readonly rateLimitGuard: RateLimitGuard,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    // 手动指定执行顺序：限流 → 登录态 → 权限
    const isRateLimitPass = await this.rateLimitGuard.canActivate(context);
    if (!isRateLimitPass) return false;

    const isAuthPass = await this.authGuard.canActivate(context);
    if (!isAuthPass) return false;

    const isPermissionPass = await this.permissionGuard.canActivate(context);
    return isPermissionPass;
  }
}
```

#### 步骤3：注册根守卫为全局守卫

```TypeScript
// app.module.ts
@Module({
  providers: [
    // 注册子守卫（供根守卫注入）
    AuthGuard,
    PermissionGuard,
    RateLimitGuard,
    // 注册根守卫为全局守卫
    { provide: APP_GUARD, useClass: RootGuard },
  ],
})
export class AppModule {}
```

#### 核心特点

- ✅ 顺序完全可控（代码显式指定），可读性高；
    
- ✅ 支持异步守卫、依赖注入；
    
- ✅ 可灵活扩展（如添加条件执行、失败降级逻辑）；
    
- ❌ 新增守卫需修改根守卫代码（需遵循开闭原则）。
    

### 三、依赖注入控制：利用 `useFactory` 动态排序

通过 `APP_[[GUARD](NestJS%20中异步守卫与参数结合使用详解.md)](NestJS%20中异步守卫与参数结合使用详解.md)` 的 `useFactory` 工厂函数，动态生成排序后的守卫数组，本质是「注册阶段动态排序」，比硬编码更灵活（无需聚合器）。

#### 示例：按业务规则动态排序

```TypeScript
// app.module.ts
@Module({
  providers: [
    // 注册所有子守卫
    AuthGuard,
    PermissionGuard,
    RateLimitGuard,
    // 工厂函数动态排序
    {
      provide: APP_GUARD,
      useFactory: (
        authGuard: AuthGuard,
        permissionGuard: PermissionGuard,
        rateLimitGuard: RateLimitGuard,
      ) => {
        // 模拟动态排序规则：生产环境优先限流，测试环境优先登录态
        const isProd = process.env.NODE_ENV === 'production';
        return isProd
          ? [rateLimitGuard, authGuard, permissionGuard] // 生产环境顺序
          : [authGuard, rateLimitGuard, permissionGuard]; // 测试环境顺序
      },
      inject: [AuthGuard, PermissionGuard, RateLimitGuard], // 注入子守卫
    },
  ],
})
export class AppModule {}
```

#### 核心特点

- ✅ 支持环境/配置差异化排序；
    
- ✅ 无需聚合器，直接返回守卫数组；
    
- ✅ 兼容依赖注入和异步逻辑；
    
- ❌ 排序规则需在注册阶段确定（无法基于请求上下文动态调整）。
    

### 四、进阶：基于请求上下文的动态排序（终极方案）

若需根据**请求参数/路由/用户信息**动态调整守卫执行顺序（如「管理员路由优先权限校验，普通用户优先限流」），可结合「聚合器 + 上下文判断」实现。

#### 示例：按路由前缀动态排序

```TypeScript
// guards/root-guard.ts
@Injectable()
export class RootGuard implements CanActivate {
  constructor(
    private readonly authGuard: AuthGuard,
    private readonly permissionGuard: PermissionGuard,
    private readonly rateLimitGuard: RateLimitGuard,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    let guards: CanActivate[] = [];

    // 按路由前缀动态排序
    if (request.url.startsWith('/admin')) {
      // 管理员路由：权限 → 登录态 → 限流
      guards = [this.permissionGuard, this.authGuard, this.rateLimitGuard];
    } else {
      // 普通路由：限流 → 登录态 → 权限
      guards = [this.rateLimitGuard, this.authGuard, this.permissionGuard];
    }

    // 执行排序后的守卫
    for (const guard of guards) {
      const result = await guard.canActivate(context);
      if (!result) return false;
    }
    return true;
  }
}
```

### 五、各方案对比 & 选型建议

|   |   |   |   |   |
|---|---|---|---|---|
|排序方式|实现复杂度|动态性|可读性|适用场景|
|注册/数组顺序|极低|静态（固定）|高|小型项目、固定顺序的全局守卫|
|守卫聚合器|中|静态/动态|高|中大型项目、需显式控制顺序|
|useFactory 动态排序|中|环境/配置动态|中|多环境差异化排序|
|上下文动态排序（聚合器）|中高|请求级动态|中|复杂业务场景、路由差异化排序|
|元数据 + 反射器|中|路由级动态|中|需声明式标记优先级的场景|

### 选型原则

1. **简单场景**：优先用「注册/数组顺序」（零成本）；
    
2. **中大型项目**：优先用「守卫聚合器」（显式可控、易维护）；
    
3. **多环境/配置差异化**：用 `useFactory` 动态排序；
    
4. **请求级动态排序**：用「聚合器 + 上下文判断」；
    
5. **声明式优先级**：用「元数据 + 反射器」（如团队约定优先级规则）。
    

### 关键注意事项

1. **全局守卫 vs 局部守卫**：全局守卫始终先于 `@UseGuards()` 注册的局部守卫执行，排序仅作用于全局守卫内部；
    
2. **异步守卫**：所有方案均需用 `await` 执行异步 `canActivate` 方法，避免顺序错乱；
    
3. **异常处理**：聚合器模式下，建议为每个守卫添加异常捕获，避免单个守卫失败导致整体崩溃；
    
4. **依赖注入**：确保所有子守卫都被注册为 `providers`，避免注入失败。