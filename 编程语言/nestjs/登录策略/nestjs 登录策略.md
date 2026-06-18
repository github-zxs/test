# NestJS 中策略的自动扫描与注册实现

在 NestJS 中实现策略的**自动扫描和注册**，核心是结合「文件系统扫描」「反射元数据」和「动态模块」三大特性：通过文件扫描发现指定目录下的策略文件，利用反射识别标记的策略类，最后通过动态模块将策略自动注入到 Nest 的依赖注入（DI）容器中。以下是可落地的完整实现方案，包含核心原理、代码示例和最佳实践。

### 一、核心原理

|   |   |
|---|---|
|技术点|作用|
|文件系统（fs）扫描|递归读取指定目录（如 `strategies/`）下的所有策略文件（`.strategy.ts`）|
|反射元数据（Reflector）|识别标记了自定义装饰器的策略类（确保只扫描合法的策略）|
|动态模块（DynamicModule）|运行时将扫描到的策略类注册为 Provider，注入到 DI 容器|
|PassportStrategy|统一策略的基类，保证所有扫描到的策略遵循 Passport 规范|

### 二、完整实现步骤

#### 步骤 1：基础准备（装饰器 + 工具函数）

##### 1.1 定义策略标记装饰器

用装饰器标记策略类，便于后续反射识别：

```TypeScript
// src/auth/decorators/strategy.decorator.ts
import { SetMetadata, Type } from '@nestjs/common';

// 元数据 Key：标记策略类
export const AUTH_STRATEGY_KEY = 'auth_strategy';
// 装饰器：标记类为认证策略，并指定策略名称（如 password/wechat）
export const AuthStrategy = (name: string) => SetMetadata(AUTH_STRATEGY_KEY, name);
```

##### 1.2 实现扫描工具函数

封装通用的扫描逻辑，递归读取目录下的策略文件，并通过反射筛选出标记了 `@AuthStrategy` 的类：

```TypeScript
// src/auth/utils/strategy-scanner.ts
import { readdirSync, statSync } from 'fs';
import { join, resolve } from 'path';
import { Reflector } from '@nestjs/core';
import { AUTH_STRATEGY_KEY } from '../decorators/strategy.decorator';

/**
 * 自动扫描指定目录下的所有认证策略类
 * @param scanDir 扫描目录（绝对路径）
 * @param reflector Nest 反射工具
 * @returns 策略类数组
 */
export const scanAuthStrategies = (scanDir: string, reflector: Reflector): Type<any>[] => {
  const strategyClasses: Type<any>[] = [];

  // 递归读取目录
  const traverseDir = (dir: string) => {
    const files = readdirSync(dir, { withFileTypes: true });

    for (const file of files) {
      const fullPath = join(dir, file.name);

      // 递归处理子目录
      if (file.isDirectory()) {
        traverseDir(fullPath);
        continue;
      }

      // 只处理 .strategy.ts 文件（过滤临时文件/非策略文件）
      if (!file.name.endsWith('.strategy.ts') || file.name.startsWith('.')) {
        continue;
      }

      // 导入文件并获取导出的类
      const module = require(resolve(fullPath));
      const exportedKeys = Object.keys(module);

      // 遍历导出的内容，筛选标记了 @AuthStrategy 的类
      for (const key of exportedKeys) {
        const clazz = module[key];
        // 检查是否是类 + 标记了策略装饰器
        if (
          typeof clazz === 'function' &&
          reflector.getMetadata(AUTH_STRATEGY_KEY, clazz)
        ) {
          strategyClasses.push(clazz);
        }
      }
    }
  };

  traverseDir(scanDir);
  return strategyClasses;
};
```

#### 步骤 2：实现动态模块（自动注册策略）

创建 `AuthModule`，通过动态模块加载扫描到的策略，并注册到 DI 容器：

```TypeScript
// src/auth/auth.module.ts
import { DynamicModule, Module, Provider, Reflector } from '@nestjs/common';
import { PassportModule } from '@nestjs/passport';
import { join } from 'path';
import { scanAuthStrategies } from './utils/strategy-scanner';
import { AuthService } from './auth.service';

@Module({
  imports: [PassportModule], // 集成 Passport 核心模块
  providers: [AuthService, Reflector], // 提供反射工具和认证服务
  exports: [AuthService], // 导出服务供外部使用
})
export class AuthModule {
  /**
   * 自动扫描并注册策略（核心方法）
   * @param options 可选配置（如自定义扫描目录）
   */
  static forRoot(options: { scanDir?: string } = {}): DynamicModule {
    // 1. 初始化反射工具
    const reflector = new Reflector();

    // 2. 确定扫描目录（默认扫描 auth/strategies 目录）
    const defaultScanDir = join(__dirname, 'strategies');
    const scanDir = options.scanDir || defaultScanDir;

    // 3. 扫描目录下的所有策略类
    const strategyClasses = scanAuthStrategies(scanDir, reflector);

    // 4. 将策略类转为 Nest Provider（注入到 DI 容器）
    const strategyProviders: Provider[] = strategyClasses.map((clazz) => ({
      provide: clazz,
      useClass: clazz,
    }));

    // 5. 返回动态模块配置
    return {
      module: AuthModule,
      providers: [...strategyProviders], // 注册扫描到的策略
      exports: [...strategyProviders], // 导出策略供 [Guard](../guard/守卫排序(自定义元数据排序).md) 使用
    };
  }
}
```

#### 步骤 3：实现示例策略（标记装饰器）

创建策略类并标记 `@AuthStrategy` 装饰器，确保能被扫描到：

```TypeScript
// src/auth/strategies/password.strategy.ts（密码策略）
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AuthStrategy } from '../decorators/strategy.decorator';
import { AuthService } from '../auth.service';

// 标记为策略，名称为 password
@AuthStrategy('password')
@Injectable()
export class PasswordStrategy extends PassportStrategy(Strategy, 'password') {
  constructor(private authService: AuthService) {
    super({ usernameField: 'username', passwordField: 'password' });
  }

  async validate(username: string, password: string) {
    const user = await this.authService.validatePasswordUser(username, password);
    if (!user) throw new UnauthorizedException('用户名/密码错误');
    return user;
  }
}
```

```TypeScript
// src/auth/strategies/wechat.strategy.ts（微信策略）
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-wechat';
import { AuthStrategy } from '../decorators/strategy.decorator';
import { AuthService } from '../auth.service';

// 标记为策略，名称为 wechat
@AuthStrategy('wechat')
@Injectable()
export class WechatStrategy extends PassportStrategy(Strategy, 'wechat') {
  constructor(private authService: AuthService) {
    super({
      appID: process.env.WECHAT_APPID,
      appSecret: process.env.WECHAT_APPSECRET,
      callbackURL: '/auth/wechat/callback',
    });
  }

  async validate(accessToken: string, refreshToken: string, profile: any) {
    return this.authService.validateWechatUser(profile);
  }
}
```

#### 步骤 4：集成到应用模块

在 `AppModule` 中调用 `AuthModule.forRoot()`，自动扫描并注册所有策略：

```TypeScript
// src/app.module.ts
import { Module } from '@nestjs/common';
import { ConfigModule } from '@nestjs/config'; // 可选：加载环境变量
import { AuthModule } from './auth/auth.module';

@Module({
  imports: [
    ConfigModule.forRoot({ isGlobal: true }), // 可选：全局加载环境变量
    // 自动扫描 auth/strategies 目录下的所有策略并注册
    AuthModule.forRoot(),
    // 如需自定义扫描目录：AuthModule.forRoot({ scanDir: join(__dirname, 'custom-strategies') })
  ],
})
export class AppModule {}
```

### 三、验证自动扫描效果

#### 3.1 新增策略（无感扩展）

如需新增「手机号验证码策略」，只需创建 `sms.strategy.ts` 并标记 `@AuthStrategy('sms')`，无需修改任何核心代码：

```TypeScript
// src/auth/strategies/sms.strategy.ts
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-sms-verify'; // 示例依赖
import { AuthStrategy } from '../decorators/strategy.decorator';
import { AuthService } from '../auth.service';

@AuthStrategy('sms') // 标记策略名称
@Injectable()
export class SmsStrategy extends PassportStrategy(Strategy, 'sms') {
  constructor(private authService: AuthService) {
    super({ mobileField: 'mobile', codeField: 'code' });
  }

  async validate(mobile: string, code: string) {
    const user = await this.authService.validateSmsUser(mobile, code);
    if (!user) throw new UnauthorizedException('验证码错误');
    return user;
  }
}
```

重启服务后，该策略会被自动扫描并注册，可直接通过 `AuthGuard('sms')` 使用。

#### 3.2 测试策略使用

创建控制器，通过 `AuthGuard` 触发自动注册的策略：

```TypeScript
// src/auth/auth.controller.ts
import { Controller, Post, Get, UseGuards, Request } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';

@Controller('auth')
export class AuthController {
  // 使用自动注册的 password 策略
  @Post('password')
  @UseGuards(AuthGuard('password'))
  passwordLogin(@Request() req) {
    return { user: req.user, token: 'mock-token' };
  }

  // 使用自动注册的 wechat 策略
  @Get('wechat')
  @UseGuards(AuthGuard('wechat'))
  wechatLogin() {}
}
```

### 四、进阶优化

#### 4.1 扫描缓存（避免重复扫描）

在生产环境中，可缓存扫描结果，避免每次应用启动重复扫描文件：

```TypeScript
// src/auth/utils/strategy-scanner.ts
let cachedStrategies: Type<any>[] | null = null;

export const scanAuthStrategies = (scanDir: string, reflector: Reflector): Type<any>[] => {
  // 缓存命中则直接返回
  if (cachedStrategies) return cachedStrategies;
  
  // 原有扫描逻辑...
  
  // 缓存结果
  cachedStrategies = strategyClasses;
  return strategyClasses;
};
```

#### 4.2 策略合法性校验

在动态模块中增加校验，确保扫描到的策略符合规范：

```TypeScript
// src/auth/auth.module.ts
static forRoot(options: { scanDir?: string } = {}): DynamicModule {
  // 原有扫描逻辑...

  // 校验：策略名称唯一
  const strategyNames = strategyClasses.map(clazz => reflector.getMetadata(AUTH_STRATEGY_KEY, clazz));
  const duplicateNames = strategyNames.filter((name, index) => strategyNames.indexOf(name) !== index);
  if (duplicateNames.length) {
    throw new Error(`重复的策略名称：${duplicateNames.join(', ')}`);
  }

  // 校验：策略类必须继承 PassportStrategy
  strategyClasses.forEach(clazz => {
    if (!(clazz.prototype instanceof PassportStrategy)) {
      throw new Error(`策略类 ${clazz.name} 必须继承 PassportStrategy`);
    }
  });

  // 原有注册逻辑...
}
```

#### 4.3 异步扫描（支持配置中心）

如需从配置中心异步获取扫描目录或策略配置，可结合 `useFactory` 实现异步注册：

```TypeScript
// src/auth/auth.module.ts
static forRootAsync(options: {
  useFactory: (...args: any[]) => Promise<{ scanDir?: string }>;
  inject?: any[];
}): DynamicModule {
  return {
    module: AuthModule,
    providers: [
      {
        provide: 'STRATEGY_SCAN_OPTIONS',
        useFactory: options.useFactory,
        inject: options.inject || [],
      },
    ],
    imports: [PassportModule],
    exports: [AuthService],
  };
}
```

### 五、核心优势

1. **零配置扩展**：新增策略只需创建文件 + 标记装饰器，无需修改核心模块；
    
2. **规范统一**：通过装饰器和反射确保所有策略遵循相同规范；
    
3. **性能可控**：支持缓存和合法性校验，适配生产环境；
    
4. **兼容 Passport 生态**：可无缝扫描任意基于 `PassportStrategy` 的策略（local/jwt/oauth2 等）。
    

这套方案完全遵循 NestJS 的设计理念，既保证了「自动扫描」的灵活性，又通过装饰器和校验保证了「策略规范」，是中大型项目多策略认证的最佳实践。