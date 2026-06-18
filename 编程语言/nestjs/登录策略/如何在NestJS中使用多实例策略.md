# 如何在NestJS中使用多实例策略？

NestJS中的「多实例策略」，本质是通过依赖注入（DI）的 `multi: true` 特性，将多个同接口/同类型的类实例绑定到同一个注入令牌（Token），注入时以数组形式获取所有实例，实现批量管理、动态切换或按需调用。核心场景包括：多类型认证策略（密码、短信、OAuth）、多数据源适配、插件化扩展等。

本文将以「多认证策略」为核心案例，详细讲解多实例策略的实现步骤、进阶用法及注意事项，确保方案可落地、可扩展。

## 一、核心原理

NestJS的依赖注入容器中，默认一个Token对应一个实例。通过在`Provider`配置中设置 `multi: true`，可声明该Token对应「多个实例」，注入时会返回所有绑定实例的数组。

核心关键点：

- **统一令牌（Token）**：所有实例绑定到同一个Token（通常用常量或抽象类作为Token，保证类型安全）；
    
- **multi: true**：标记该Token对应多个实例，是实现多实例策略的核心配置；
    
- **数组注入**：注入时需以数组形式接收实例，再通过业务逻辑筛选、调用目标实例；
    
- **动态模块适配**：结合`forRoot`/`forRootAsync`动态模块，可根据配置启用/禁用实例，实现灵活扩展。
    

## 二、完整实现步骤（以多认证策略为例）

以下案例实现「密码登录」「短信登录」「GitHub OAuth登录」三个策略的多实例管理，通过同一个Token注入所有策略，按需调用。

### 步骤1：定义抽象接口（统一策略规范）

先定义抽象策略接口，约束所有实例的方法和属性，保证多实例的一致性：

```typescript
// src/auth/strategies/abstract-auth.strategy.ts
import { Strategy } from 'passport-strategy';

/**
 * 抽象认证策略接口：所有认证策略必须实现该接口
 */
export abstract class AbstractAuthStrategy extends Strategy {
  // 策略唯一标识（如：password、sms、github）
  abstract readonly type: string;
  // 核心验证方法（不同策略实现不同逻辑）
  abstract validate(...args: any[]): Promise<any>;
}
```

### 步骤2：实现具体策略类（多实例实现）

创建多个策略类，均实现抽象接口，各自实现专属业务逻辑：

#### 2.1 密码登录策略

```typescript
// src/auth/strategies/password.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AbstractAuthStrategy } from './abstract-auth.strategy';
import { UserService } from '../../user/user.service';

@Injectable()
export class PasswordStrategy 
  extends PassportStrategy(Strategy, 'password') 
  implements AbstractAuthStrategy 
{
  readonly type = 'password'; // 策略标识

  constructor(private readonly userService: UserService) {
    super({ usernameField: 'username', passwordField: 'password' });
  }

  // 密码验证逻辑：查询用户并校验密码
  async validate(username: string, password: string) {
    const user = await this.userService.findByUsername(username);
    if (!user) throw new Error('用户不存在');
    
    const isPwdValid = await this.userService.comparePassword(password, user.password);
    if (!isPwdValid) throw new Error('密码错误');
    
    return { id: user.id, username: user.username, role: user.role };
  }
}
```

#### 2.2 短信登录策略

```typescript
// src/auth/strategies/sms.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AbstractAuthStrategy } from './abstract-auth.strategy';
import { SmsService } from '../../sms/sms.service';
import { UserService } from '../../user/user.service';

@Injectable()
export class SmsStrategy 
  extends PassportStrategy(Strategy, 'sms') 
  implements AbstractAuthStrategy 
{
  readonly type = 'sms'; // 策略标识

  constructor(
    private readonly smsService: SmsService,
    private readonly userService: UserService
  ) {
    super({ usernameField: 'mobile', passwordField: 'smsCode' });
  }

  // 短信验证逻辑：校验验证码并查询/创建用户
  async validate(mobile: string, smsCode: string) {
    const isCodeValid = await this.smsService.verifyCode(mobile, smsCode);
    if (!isCodeValid) throw new Error('验证码错误或已过期');
    
    let user = await this.userService.findByMobile(mobile);
    if (!user) user = await this.userService.createUserByMobile(mobile);
    
    return { id: user.id, mobile: user.mobile, role: user.role };
  }
}
```

#### 2.3 GitHub OAuth策略

```typescript
// src/auth/strategies/github.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-github2';
import { AbstractAuthStrategy } from './abstract-auth.strategy';
import { UserService } from '../../user/user.service';

@Injectable()
export class GithubStrategy 
  extends PassportStrategy(Strategy, 'github') 
  implements AbstractAuthStrategy 
{
  readonly type = 'github'; // 策略标识

  constructor(private readonly userService: UserService) {
    super({
      clientID: process.env.GITHUB_CLIENT_ID,
      clientSecret: process.env.GITHUB_CLIENT_SECRET,
      callbackURL: 'http://localhost:3000/auth/github/callback'
    });
  }

  // OAuth验证逻辑：通过GitHub信息查询/创建用户
  async validate(accessToken: string, refreshToken: string, profile: any) {
    const githubId = profile.id;
    let user = await this.userService.findByGithubId(githubId);
    
    if (!user) {
      user = await this.userService.createUserByGithub({
        githubId,
        username: profile.username,
        avatar: profile._json.avatar_url
      });
    }
    
    return { id: user.id, username: user.username, githubId };
  }
}
```

### 步骤3：配置多实例策略（绑定统一Token）

创建动态模块，将多个策略类绑定到同一个Token，并设置 `multi: true` 开启多实例模式：

#### 3.1 定义模块配置和Token

```typescript
// src/auth/auth.module.types.ts
// 可启用的策略类型
export type EnabledAuthStrategy = 'password' | 'sms' | 'github';

// 模块配置选项
export interface AuthModuleOptions {
  enabledStrategies: EnabledAuthStrategy[]; // 要启用的策略列表
}

// 多实例策略统一注入Token（常量，避免硬编码）
export const AUTH_STRATEGIES_TOKEN = 'AUTH_STRATEGIES_TOKEN';
```

#### 3.2 实现动态模块

```typescript
// src/auth/auth.module.ts
import { DynamicModule, Module, Provider } from '@nestjs/common';
import { AbstractAuthStrategy } from './strategies/abstract-auth.strategy';
import { PasswordStrategy } from './strategies/password.strategy';
import { SmsStrategy } from './strategies/sms.strategy';
import { GithubStrategy } from './strategies/github.strategy';
import { AUTH_STRATEGIES_TOKEN, AuthModuleOptions, EnabledAuthStrategy } from './auth.module.types';
import { UserService } from '../user/user.service';
import { SmsService } from '../sms/sms.service';

// 策略映射：类型标识 → 策略类（便于动态匹配）
const STRATEGY_CLASS_MAP: Record<EnabledAuthStrategy, typeof AbstractAuthStrategy> = {
  password: PasswordStrategy,
  sms: SmsStrategy,
  github: GithubStrategy
};

@Module({
  providers: [UserService, SmsService], // 依赖服务
  exports: [UserService, SmsService]
})
export class AuthModule {
  /**
   * 同步配置多实例策略
   * @param options 模块配置（指定启用的策略）
   */
  static forRoot(options: AuthModuleOptions): DynamicModule {
    // 1. 生成多实例Provider：绑定到同一个Token，设置multi: true
    const strategyProviders: Provider[] = options.enabledStrategies.map(strategyType => ({
      provide: AUTH_STRATEGIES_TOKEN, // 统一Token
      useClass: STRATEGY_CLASS_MAP[strategyType], // 对应策略类
      multi: true // 关键：开启多实例
    }));

    return {
      module: AuthModule,
      providers: [...strategyProviders],
      exports: [AUTH_STRATEGIES_TOKEN], // 导出Token，供其他模块注入
      global: true // 可选：设为全局模块，无需在其他模块重复导入
    };
  }

  /**
   * 异步配置多实例策略（支持从配置中心/数据库获取配置）
   * @param options 异步配置选项
   */
  static forRootAsync(options: {
    useFactory: (...args: any[]) => Promise<AuthModuleOptions> | AuthModuleOptions;
    inject?: any[]; // 注入依赖（如配置服务）
  }): DynamicModule {
    // 1. 异步获取配置的Provider
    const asyncOptionsProvider: Provider = {
      provide: 'AUTH_MODULE_ASYNC_OPTIONS',
      useFactory: options.useFactory,
      inject: options.inject || []
    };

    // 2. 动态生成多实例策略Provider（基于异步配置）
    const strategyProviders: Provider = {
      provide: AUTH_STRATEGIES_TOKEN,
      useFactory: (asyncOptions: AuthModuleOptions) => {
        // 根据异步配置，返回多个策略实例
        return asyncOptions.enabledStrategies.map(strategyType => {
          const StrategyClass = STRATEGY_CLASS_MAP[strategyType];
          // 手动实例化（若策略有依赖，需从DI容器获取，或让Nest自动注入）
          return new StrategyClass(
            ...(StrategyClass === SmsStrategy 
              ? [new SmsService(), new UserService()] 
              : [new UserService()])
          );
        });
      },
      inject: ['AUTH_MODULE_ASYNC_OPTIONS'],
      multi: true
    };

    return {
      module: AuthModule,
      providers: [asyncOptionsProvider, strategyProviders],
      exports: [AUTH_STRATEGIES_TOKEN],
      global: true
    };
  }
}
```

### 步骤4：注入并使用多实例策略

通过统一Token注入多实例策略（数组形式），再根据业务逻辑（如请求参数）筛选目标策略：

#### 4.1 封装策略服务（管理多实例）

```typescript
// src/auth/auth.strategy.service.ts
import { Inject, Injectable, NotFoundException } from '@nestjs/common';
import { AbstractAuthStrategy } from './strategies/abstract-auth.strategy';
import { AUTH_STRATEGIES_TOKEN } from './auth.module.types';

@Injectable()
export class AuthStrategyService {
  // 注入多实例策略：数组形式
  constructor(
    @Inject(AUTH_STRATEGIES_TOKEN)
    private readonly strategies: AbstractAuthStrategy[]
  ) {}

  /**
   * 根据策略类型获取目标实例
   * @param type 策略类型（如：password、sms）
   */
  getStrategyByType(type: string): AbstractAuthStrategy {
    const strategy = this.strategies.find(s => s.type === type);
    if (!strategy) {
      throw new NotFoundException(`不支持的认证策略：${type}`);
    }
    return strategy;
  }

  /**
   * 批量执行所有策略的初始化逻辑（示例）
   */
  initAllStrategies() {
    this.strategies.forEach(strategy => {
      console.log(`✅ 策略 [${strategy.type}] 初始化完成`);
    });
  }

  /**
   * 获取所有启用的策略类型
   */
  getEnabledStrategyTypes(): string[] {
    return this.strategies.map(s => s.type);
  }
}
```

#### 4.2 控制器中使用策略

```typescript
// src/auth/auth.controller.ts
import { Controller, Post, Get, Body, Query, Request } from '@nestjs/common';
import { AuthStrategyService } from './auth.strategy.service';
import { AuthService } from './auth.service';

@Controller('auth')
export class AuthController {
  constructor(
    private readonly strategyService: AuthStrategyService,
    private readonly authService: AuthService
  ) {}

  // 初始化所有策略
  @Get('init-strategies')
  initStrategies() {
    this.strategyService.initAllStrategies();
    return { code: 200, message: '所有策略初始化完成' };
  }

  // 获取启用的策略列表
  @Get('enabled-strategies')
  getEnabledStrategies() {
    const types = this.strategyService.getEnabledStrategyTypes();
    return { code: 200, data: { enabledStrategies: types } };
  }

  // 通用登录接口：根据strategyType选择策略
  @Post('login')
  async login(@Body() body: { strategyType: string; [key: string]: any }) {
    const { strategyType, ...params } = body;
    // 1. 获取目标策略实例
    const strategy = this.strategyService.getStrategyByType(strategyType);
    
    // 2. 执行策略验证逻辑（根据策略类型传递对应参数）
    let user: any;
    switch (strategyType) {
      case 'password':
        user = await strategy.validate(params.username, params.password);
        break;
      case 'sms':
        user = await strategy.validate(params.mobile, params.smsCode);
        break;
      case 'github':
        user = await strategy.validate(params.accessToken, params.refreshToken, params.profile);
        break;
      default:
        throw new Error('不支持的认证类型');
    }
    
    // 3. 生成Token（业务逻辑）
    const token = await this.authService.generateToken(user);
    return { code: 200, data: { user, token } };
  }
}
```

### 步骤5：根模块注册动态模块

```typescript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { AuthModule } from './auth/auth.module';
import { AuthController } from './auth/auth.controller';
import { AuthStrategyService } from './auth/auth.strategy.service';
import { AuthService } from './auth/auth.service';
import { ConfigModule } from '@nestjs/config';

@Module({
  imports: [
    // 加载环境变量（供GitHub策略使用）
    ConfigModule.forRoot({ isGlobal: true }),
    // 同步注册多实例策略：启用password、sms、github
    AuthModule.forRoot({
      enabledStrategies: ['password', 'sms', 'github']
    })
    // 若需异步注册（从配置中心获取），可替换为：
    // AuthModule.forRootAsync({
    //   useFactory: async (configService: ConfigService) => {
    //     const enabledStrategies = configService.get<string[]>('AUTH_ENABLED_STRATEGIES');
    //     return { enabledStrategies: enabledStrategies as EnabledAuthStrategy[] };
    //   },
    //   inject: [ConfigService]
    // })
  ],
  controllers: [AuthController],
  providers: [AuthStrategyService, AuthService]
})
export class AppModule {}
```

## 三、进阶用法

### 1. 动态添加/移除策略实例

若需在运行时动态调整多实例策略（如插件化场景），可结合`ModuleRef`手动管理实例：

```typescript
// src/auth/auth.strategy.service.ts（补充）
import { ModuleRef } from '@nestjs/core';

@Injectable()
export class AuthStrategyService {
  constructor(
    @Inject(AUTH_STRATEGIES_TOKEN)
    private readonly strategies: AbstractAuthStrategy[],
    private readonly moduleRef: ModuleRef // 模块引用
  ) {}

  /**
   * 动态添加策略实例
   * @param strategyClass 策略类
   */
  async addStrategy(strategyClass: typeof AbstractAuthStrategy) {
    // 实例化策略类（自动注入依赖）
    const strategyInstance = await this.moduleRef.resolve(strategyClass);
    // 添加到策略数组
    this.strategies.push(strategyInstance);
  }

  /**
   * 动态移除策略实例
   * @param type 策略类型
   */
  removeStrategy(type: string) {
    this.strategies = this.strategies.filter(s => s.type !== type);
  }
}
```

### 2. 多实例策略与守卫结合

结合NestJS的`AuthGuard`，实现动态策略守卫：

```typescript
// src/auth/dynamic-auth.guard.ts
import { CanActivate, ExecutionContext, Injectable } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { AuthStrategyService } from './auth.strategy.service';

@Injectable()
export class DynamicAuthGuard implements CanActivate {
  constructor(
    private readonly strategyService: AuthStrategyService
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    // 从请求头/参数中获取策略类型
    const strategyType = request.headers['x-auth-strategy'] || request.body.strategyType;
    if (!strategyType) throw new Error('请指定认证策略类型');

    // 动态创建对应策略的AuthGuard
    const strategy = this.strategyService.getStrategyByType(strategyType);
    const guard = new (AuthGuard(strategy.type))();
    
    return guard.canActivate(context) as Promise<boolean>;
  }
}

// 控制器使用
@Post('login-with-guard')
@UseGuards(DynamicAuthGuard)
async loginWithGuard(@Request() req) {
  const token = await this.authService.generateToken(req.user);
  return { code: 200, data: { user: req.user, token } };
}
```

## 四、注意事项

1. **上下文流畅性**：多实例策略需基于统一接口实现，确保所有实例的方法、属性规范一致，避免注入后因类型不匹配导致的错误；
    
2. **multi: true 必选**：所有绑定到同一Token的Provider必须设置 `multi: true`，否则会覆盖前一个实例；
    
3. **依赖注入兼容**：若策略类有构造函数依赖（如`UserService`），需确保依赖已注册到模块中，Nest会自动为每个策略实例注入依赖；
    
4. **类型安全**：注入多实例时，建议通过抽象接口约束数组类型（如`AbstractAuthStrategy[]`），避免运行时类型错误；
    
5. **性能优化**：若多实例数量较多，可缓存常用策略实例，避免每次调用都通过`find`筛选；
    
6. **异常处理**：获取策略实例时需处理「策略不存在」的异常，统一返回业务错误信息；
    
7. **全局模块慎用**：若多实例策略仅在部分模块使用，无需设置`global: true`，避免污染全局依赖容器。
    

## 五、典型应用场景

- **多类型认证**：如本文案例，支持密码、短信、OAuth等多种登录方式，按需切换；
    
- **多数据源适配**：同一Token绑定MySQL、MongoDB、Redis数据源实例，根据业务需求动态切换；
    
- **插件化扩展**：将插件类绑定到同一Token，批量加载、初始化插件，支持动态启停；
    
- **多格式解析**：绑定JSON、XML、YAML解析器实例，根据请求头`Content-Type`选择解析器；
    
- **多规则校验**：绑定多个校验规则实例，批量执行校验逻辑。
    

## 六、总结

NestJS多实例策略的核心是 `multi: true` + 统一Token + 抽象接口，实现步骤可概括为：

1. 定义抽象接口，约束多实例规范；

2. 实现多个具体类，适配接口；

3. 配置动态模块，将多实例绑定到同一Token并设置`multi: true`；

4. 数组形式注入多实例，通过业务逻辑筛选使用；

5. （可选）结合`ModuleRef`实现动态添加/移除实例。

该方案既保留了NestJS依赖注入的灵活性，又能高效管理同类型多实例，是实现插件化、多策略架构的最佳实践。