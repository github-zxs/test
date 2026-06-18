# NestJS 中同一注入令牌注册多个认证策略类

在 NestJS 中，**同一个注入令牌（token）注册多个类** 是实现「多实例策略」「批量注入同类型服务」的核心需求（比如：一个 `AuthStrategy` token 下注册密码、短信、OAuth 等多个策略类）。核心实现方式是利用 NestJS 的 `multi: true` 特性，结合动态模块和依赖注入，让同一个 token 绑定多个类实例。

### 一、核心原理

NestJS 的 `Provider` 配置中，`multi: true` 表示「该 token 对应多个提供者」，注入时会以**数组形式**返回所有绑定的实例。这是实现「同一个 token 注册多个类」的关键。

### 二、完整实现：同一个 token 注册多个策略类

以认证场景为例：将「密码策略」「短信策略」「GitHub OAuth 策略」都绑定到 `AUTH_STRATEGIES` token 下，实现批量注入和管理。

#### 步骤 1：定义基础类型与策略类

先实现多个策略类（统一接口），作为待注册的目标类：

```TypeScript
// src/auth/strategies/abstract-auth.strategy.ts
import { Strategy } from 'passport-strategy';

// 抽象策略接口（所有策略统一规范）
export abstract class AbstractAuthStrategy extends Strategy {
  abstract type: string; // 策略类型标识
  abstract validate(...args: any[]): Promise<any>; // 核心验证方法
}

// src/auth/strategies/password.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AbstractAuthStrategy } from './abstract-auth.strategy';

@Injectable()
export class PasswordStrategy extends PassportStrategy(Strategy, 'password') implements AbstractAuthStrategy {
  type = 'password';

  async validate(username: string, password: string) {
    return { id: 1, username, strategy: this.type };
  }
}

// src/auth/strategies/sms.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AbstractAuthStrategy } from './abstract-auth.strategy';

@Injectable()
export class SmsStrategy extends PassportStrategy(Strategy, 'sms') implements AbstractAuthStrategy {
  type = 'sms';

  async validate(mobile: string, code: string) {
    return { id: 1, mobile, strategy: this.type };
  }
}

// src/auth/strategies/github.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-github2';
import { AbstractAuthStrategy } from './abstract-auth.strategy';

@Injectable()
export class GithubStrategy extends PassportStrategy(Strategy, 'github') implements AbstractAuthStrategy {
  type = 'github';

  constructor() {
    super({
      clientID: 'GITHUB_CLIENT_ID',
      clientSecret: 'GITHUB_CLIENT_SECRET',
      callbackURL: 'http://localhost:3000/auth/github/callback',
    });
  }

  async validate(accessToken: string, refreshToken: string, profile: any) {
    return { id: profile.id, name: profile.username, strategy: this.type };
  }
}
```

#### 步骤 2：定义动态模块配置

```TypeScript
// src/auth/auth.module.interface.ts
export type EnabledStrategy = 'password' | 'sms' | 'github';

export interface AuthModuleOptions {
  enabledStrategies: EnabledStrategy[]; // 要启用的策略列表
}
```

#### 步骤 3：动态模块注册（同一个 token 绑定多个类）

核心是配置 `multi: true`，让 `AUTH_STRATEGIES` token 对应多个策略类：

```TypeScript
// src/auth/auth.module.ts
import { DynamicModule, Module, Provider } from '@nestjs/common';
import { AbstractAuthStrategy } from './strategies/abstract-auth.strategy';
import { PasswordStrategy } from './strategies/password.strategy';
import { SmsStrategy } from './strategies/sms.strategy';
import { GithubStrategy } from './strategies/github.strategy';
import { AuthModuleOptions, EnabledStrategy } from './auth.module.interface';

// 策略映射：标识 → 策略类
const STRATEGY_MAP: Record<EnabledStrategy, typeof AbstractAuthStrategy> = {
  password: PasswordStrategy,
  sms: SmsStrategy,
  github: GithubStrategy,
};

// 统一注入令牌（常量）
export const AUTH_STRATEGIES = 'AUTH_STRATEGIES';

@Module({})
export class AuthModule {
  static forRoot(options: AuthModuleOptions): DynamicModule {
    // 1. 为每个启用的策略创建 Provider，绑定到同一个 token（AUTH_STRATEGIES）
    const strategyProviders: Provider[] = options.enabledStrategies.map((strategy) => ({
      provide: AUTH_STRATEGIES, // 同一个 token
      useClass: STRATEGY_MAP[strategy], // 不同的类
      multi: true, // 关键：标识该 token 对应多个提供者
    }));

    // 2. 返回动态模块
    return {
      module: AuthModule,
      providers: [...strategyProviders],
      exports: [AUTH_STRATEGIES], // 导出令牌，供其他模块注入
    };
  }
}
```

#### 步骤 4：注入并使用多个类实例

通过 `@Inject(AUTH_STRATEGIES)` 注入，得到的是**所有绑定类的实例数组**：

```TypeScript
// src/auth/auth.service.ts
import { Inject, Injectable } from '@nestjs/common';
import { AbstractAuthStrategy } from './strategies/abstract-auth.strategy';
import { AUTH_STRATEGIES } from './auth.module';

@Injectable()
export class AuthService {
  // 注入同一个 token 下的所有策略实例（数组形式）
  constructor(
    @Inject(AUTH_STRATEGIES)
    private readonly strategies: AbstractAuthStrategy[],
  ) {}

  // 根据类型获取对应的策略实例
  getStrategy(type: string): AbstractAuthStrategy | undefined {
    return this.strategies.find((strategy) => strategy.type === type);
  }

  // 批量执行所有策略的初始化逻辑（示例）
  initAllStrategies() {
    this.strategies.forEach((strategy) => {
      console.log(`初始化策略: ${strategy.type}`);
    });
  }
}

// src/app.module.ts
import { Module } from '@nestjs/common';
import { AuthModule } from './auth/auth.module';
import { AuthService } from './auth/auth.service';

@Module({
  imports: [
    AuthModule.forRoot({
      enabledStrategies: ['password', 'sms', 'github'], // 注册3个策略到同一个token
    }),
  ],
  providers: [AuthService],
})
export class AppModule {}
```

#### 步骤 5：控制器中使用

```TypeScript
// src/auth/auth.controller.ts
import { Controller, Get, Query } from '@nestjs/common';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(private readonly authService: AuthService) {}

  // 根据类型获取策略并执行验证
  @Get('validate')
  async validate(@Query('type') type: string) {
    const strategy = this.authService.getStrategy(type);
    if (!strategy) {
      return { code: 400, message: '策略不存在' };
    }

    // 模拟执行验证逻辑
    const result = await strategy.validate('test', 'test');
    return { code: 200, data: result };
  }

  // 初始化所有策略
  @Get('init')
  init() {
    this.authService.initAllStrategies();
    return { code: 200, message: '所有策略初始化完成' };
  }
}
```

### 三、进阶场景：异步注册多个类（forRootAsync）

如果需要**异步配置**（比如从数据库/配置中心获取要启用的策略列表），只需调整动态模块的 `forRootAsync` 方法：

```TypeScript
// src/auth/auth.module.ts（补充 forRootAsync）
import { DynamicModule, Module, Provider } from '@nestjs/common';
import { AUTH_STRATEGIES, STRATEGY_MAP } from './auth.module';
import { AuthModuleOptions, EnabledStrategy } from './auth.module.interface';

@Module({})
export class AuthModule {
  // 同步 forRoot（同上）
  static forRoot(options: AuthModuleOptions): DynamicModule { /* ... */ }

  /**
   * 异步注册多个类到同一个 token
   */
  static forRootAsync(options: {
    useFactory: (...args: any[]) => Promise<AuthModuleOptions> | AuthModuleOptions;
    inject?: any[];
  }): DynamicModule {
    // 1. 异步获取配置
    const asyncOptionsProvider: Provider = {
      provide: 'AUTH_MODULE_OPTIONS',
      useFactory: options.useFactory,
      inject: options.inject || [],
    };

    // 2. 动态创建多实例 Provider
    const strategyProviders: Provider = {
      provide: AUTH_STRATEGIES,
      useFactory: (moduleOptions: AuthModuleOptions) => {
        // 返回多个策略实例（手动实例化，或让DI自动处理）
        return moduleOptions.enabledStrategies.map((strategy) => {
          return new (STRATEGY_MAP[strategy])();
        });
      },
      inject: ['AUTH_MODULE_OPTIONS'],
      multi: true, // 关键：保持 multi: true
    };

    return {
      module: AuthModule,
      providers: [asyncOptionsProvider, strategyProviders],
      exports: [AUTH_STRATEGIES],
    };
  }
}

// 使用示例（从配置中心异步获取策略列表）
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AuthModule } from './auth/auth.module';
import { ConfigCenterService } from './config-center/config-center.service';

@Module({
  imports: [
    AuthModule.forRootAsync({
      useFactory: async (configCenter: ConfigCenterService) => {
        // 异步获取启用的策略列表（比如从数据库/HTTP接口）
        const enabledStrategies = await configCenter.getEnabledAuthStrategies();
        return { enabledStrategies };
      },
      inject: [ConfigCenterService],
    }),
  ],
  providers: [ConfigCenterService],
})
export class AppModule {}
```

### 四、关键技巧与注意事项

#### 1. `multi: true` 的核心规则

- 必须在**每个 Provider** 中声明 `multi: true`（或在批量创建时统一设置）；
    
- 注入时，该 token 对应的是**数组**，而非单个实例；
    
- 同一个 token 下的多个 Provider 会按注册顺序放入数组。
    

#### 2. 类型安全优化

通过泛型约束数组类型，避免运行时类型错误：

```TypeScript
// src/auth/auth.service.ts
import { Type } from '@nestjs/common';

// 注入时指定数组类型
constructor(
  @Inject(AUTH_STRATEGIES)
  private readonly strategies: AbstractAuthStrategy[],
) {}

// 封装获取策略的方法，增加类型提示
getStrategy<T extends AbstractAuthStrategy>(type: string): T | undefined {
  return this.strategies.find((s) => s.type === type) as T;
}
```

#### 3. 依赖注入兼容

如果策略类需要注入其他服务（比如 `PasswordStrategy` 依赖 `UserService`），无需额外处理——NestJS 的 DI 容器会自动解析每个策略类的构造函数依赖，只需确保依赖已注册：

```TypeScript
// src/auth/strategies/password.strategy.ts
import { Injectable } from '@nestjs/common';
import { UserService } from '../user/user.service';

@Injectable()
export class PasswordStrategy extends PassportStrategy(Strategy, 'password') implements AbstractAuthStrategy {
  // 注入 UserService
  constructor(private readonly userService: UserService) {
    super();
  }

  async validate(username: string, password: string) {
    return this.userService.validate(username, password);
  }
}
```

#### 4. 全局注册多个类

如果需要全局可用，在动态模块中设置 `global: true`：

```TypeScript
// src/auth/auth.module.ts
static forRoot(options: AuthModuleOptions): DynamicModule {
  return {
    module: AuthModule,
    providers: [...strategyProviders],
    exports: [AUTH_STRATEGIES],
    global: true, // 全局模块，其他模块无需 import
  };
}
```

#### 5. 动态移除/替换类

通过配置控制启用/禁用策略，实现动态调整同一个 token 下的类列表：

```TypeScript
// 只启用密码和短信策略
AuthModule.forRoot({
  enabledStrategies: ['password', 'sms'],
});

// 后续通过配置中心更新，只启用 GitHub 策略
AuthModule.forRoot({
  enabledStrategies: ['github'],
});
```

### 五、典型应用场景

1. **多策略认证**：同一个 `AUTH_STRATEGIES` token 下注册密码、短信、OAuth 等策略，按需调用；
    
2. **多数据源适配**：同一个 `DATA_SOURCES` token 下注册 MySQL、MongoDB、Redis 数据源，动态切换；
    
3. **多格式解析器**：同一个 `PARSERS` token 下注册 JSON、XML、YAML 解析器，根据请求头选择；
    
4. **插件化扩展**：同一个 `PLUGINS` token 下注册多个插件类，批量加载和执行。
    

### 六、总结

NestJS 中「同一个 token 注册多个类」的核心是：

1. 声明 Provider 时设置 `multi: true`；
    
2. 用统一的 token（如 `AUTH_STRATEGIES`）绑定所有目标类；
    
3. 注入时以数组形式接收所有实例；
    
4. 结合动态模块（`forRoot`/`forRootAsync`）实现配置驱动的多类注册。
    

这种方式既保留了 NestJS 依赖注入的灵活性，又能高效管理同类型的多个类实例，是实现「插件化」「多策略」架构的最佳实践。