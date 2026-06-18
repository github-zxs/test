# NestJS 中常用装饰器分类详解

装饰器是 NestJS 实现依赖注入、路由定义、请求处理的核心语法，也是掌握其核心开发模式的关键。本文按使用场景对常用装饰器进行分类梳理，搭配清晰示例，方便快速理解与落地使用。

## 一、核心装饰器分类与详解

NestJS 装饰器按功能可划分为六大核心类别：**模块/控制器/提供者（基础结构类）**、**路由与请求方法（HTTP 处理类）**、**参数提取（请求数据类）**、**依赖注入（服务关联类）**、**响应与异常（结果处理类）**、**生命周期（钩子回调类）**，以下是各类别下的常用装饰器详解。

### 1. 模块/控制器/提供者（基础结构类）

这类装饰器用于定义 NestJS 应用的核心骨架，规范模块、控制器、服务的结构与依赖关系。

|   |   |   |
|---|---|---|
|**装饰器**|**作用**|**示例**|
|`@Module()`|定义模块，组织控制器、提供者，管理导入/导出依赖|`import { Module } from '@nestjs/common';` `import { UserController } from './user.controller';` `import { UserService } from './user.service';` `@Module({` `imports: [], // 导入其他模块` `controllers: [UserController], // 注册控制器` `providers: [UserService], // 注册服务（提供者）` `exports: [UserService], // 导出供其他模块使用` `})` `export class UserModule {}`|
|`@Controller(prefix?)`|定义控制器，处理 HTTP 请求，可指定路由前缀（简化路由定义）|`import { Controller } from '@nestjs/common';` `// 路由前缀为 /users，该控制器下所有路由均以 /users 开头` `@Controller('users')` `export class UserController {}`|
|`@Injectable()`|标记类为“提供者”（通常是服务），使其可被依赖注入|`import { Injectable } from '@nestjs/common';` `@Injectable() // 标记为可注入服务` `export class UserService {` `findAll() {` `return [{ id: 1, name: '张三' }];` `}` `}`|

### 2. 路由与请求方法装饰器（HTTP 处理类）

用于定义 HTTP 请求方法与路由地址，是接口开发的核心，遵循 RESTful 规范设计。

|   |   |   |
|---|---|---|
|**装饰器**|**作用**|**示例**|
|`@Get(path?)`|处理 GET 请求，可指定子路由|`import { Controller, Get, Param } from '@nestjs/common';` `@Controller('users')` `export class UserController {` `// 匹配 GET /users` `@Get()` `findAll() {` `return '所有用户列表';` `}` `// 匹配 GET /users/:id（路径参数 id）` `@Get(':id')` `findOne(@Param('id') id: string) {` ``return `获取 ID 为 ${id} 的用户`;`` `}` `}`|
|`@Post(path?)`|处理 POST 请求（通常用于创建资源）|`import { Controller, Post, Body } from '@nestjs/common';` `import { CreateUserDto } from './dto/create-user.dto';` `@Controller('users')` `export class UserController {` `@Post()` `create(@Body() createUserDto: CreateUserDto) {` ``return `创建用户：${JSON.stringify(createUserDto)}`;`` `}` `}`|
|`@Put(path?)`|处理 PUT 请求（通常用于全量更新资源）|`import { Controller, Put, Param, Body } from '@nestjs/common';` `import { UpdateUserDto } from './dto/update-user.dto';` `@Controller('users')` `export class UserController {` `@Put(':id')` `update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {` ``return `更新 ID 为 ${id} 的用户：${JSON.stringify(updateUserDto)}`;`` `}` `}`|
|`@Delete(path?)`|处理 DELETE 请求（通常用于删除资源）|`import { Controller, Delete, Param } from '@nestjs/common';` `@Controller('users')` `export class UserController {` `@Delete(':id')` `remove(@Param('id') id: string) {` ``return `删除 ID 为 ${id} 的用户`;`` `}` `}`|
|`@Patch(path?)`|处理 PATCH 请求（通常用于部分更新资源）|`import { Controller, Patch, Param, Body } from '@nestjs/common';` `@Controller('users')` `export class UserController {` `@Patch(':id/name')` `updateName(@Param('id') id: string, @Body('name') name: string) {` ``return `更新 ID 为 ${id} 的用户名称为：${name}`;`` `}` `}`|
|`@Options()/@Head()`|处理 OPTIONS/HEAD 请求（较少用，用于跨域预检、获取响应头）|用法同 `@Get()`，无需额外配置|

### 3. 参数装饰器（请求数据类）

用于从 HTTP 请求中精准提取数据（路径参数、查询参数、请求体等），避免直接依赖原生请求对象。

|   |   |   |
|---|---|---|
|**装饰器**|**作用**|**示例**|
|`@Param(key?)`|提取 URL 路径参数（如 /users/:id 中的 id），不传 key 则返回所有参数对象|`// 提取单个参数` `@Get(':id')` `findOne(@Param('id') id: string) { ... }` `// 提取所有路径参数（返回对象）` `@Get(':id/:name')` `find(@Param() params: { id: string; name: string }) {` ``return `ID: ${params.id}, 名称: ${params.name}`;`` `}`|
|`@Query(key?)`|提取 URL 查询参数（如 /users?page=1&size=10），不传 key 则返回所有参数对象|`// 提取单个查询参数` `@Get()` `findAll(@Query('page') page: string, @Query('size') size: string) {` ``return `第 ${page} 页，每页 ${size} 条`;`` `}` `// 提取所有查询参数` `@Get()` `findAll(@Query() query: { page: string; size: string }) { ... }`|
|`@Body(key?)`|提取 POST/PUT/PATCH 请求体数据，不传 key 则返回完整请求体，推荐结合 DTO 校验|`// 结合 DTO 校验（推荐）` `class CreateUserDto {` `name: string;` `age: number;` `}` `@Post()` `create(@Body() createUserDto: CreateUserDto) {` `return createUserDto;` `}` `// 提取请求体中单个字段` `@Post()` `create(@Body('name') name: string) { ... }`|
|`@Headers(key?)`|提取请求头信息（如 Authorization、User-Agent），不传 key 则返回所有请求头对象|`// 提取单个请求头` `@Get()` `getHeaders(@Headers('user-agent') userAgent: string) {` ``return `User-Agent: ${userAgent}`;`` `}` `// 提取所有请求头` `@Get()` `getAllHeaders(@Headers() headers: Record<string, string>) { ... }`|
|`@Request()/@Req()`|提取原生 Express/Fastify 请求对象（慎用，避免耦合底层框架）|`import { Controller, Get, Req } from '@nestjs/common';` `import { Request } from 'express';` `@Controller()` `export class AppController {` `@Get()` `getRequest(@Req() req: Request) {` `return req.ip; // 访问原生请求对象属性` `}` `}`|
|`@Response()/@Res()`|提取原生响应对象（慎用，会接管 Nest 原生响应处理，需手动返回响应）|`import { Controller, Get, Res } from '@nestjs/common';` `import { Response } from 'express';` `@Controller()` `export class AppController {` `@Get()` `getResponse(@Res() res: Response) {` `res.status(200).json({ message: '自定义响应' }); // 手动返回响应` `}` `}`|
|`@Cookie(key?)`|提取请求 Cookie 中的值，不传 key 则返回所有 Cookie 对象|`@Get()` `getCookie(@Cookie('token') token: string) {` ``return `Cookie token: ${token}`;`` `}`|

### 4. 依赖注入装饰器（服务关联类）

用于实现依赖注入，解耦控制器与服务、服务与服务之间的依赖关系，是 NestJS 核心设计模式。

|   |   |   |
|---|---|---|
|**装饰器**|**作用**|**示例**|
|`@Inject(token)`|手动注入非类提供者（如自定义令牌、值提供者、工厂提供者）|`// 1. 定义自定义令牌` `export const DB_CONNECTION = 'DB_CONNECTION';` `// 2. 模块中注册提供者` `@Module({` `providers: [` `{` `provide: DB_CONNECTION,` `useValue: { host: 'localhost', port: 3306 },` `},` `],` `})` `export class AppModule {}` `// 3. 控制器/服务中注入` `@Controller()` `export class AppController {` `constructor(` `@Inject(DB_CONNECTION) private dbConnection: any,` `) {}` `@Get()` `getDbConfig() {` `return this.dbConnection;` `}` `}`|
|`@InjectRepository(Entity)`|TypeORM 专用，注入实体对应的仓库（Repository），用于操作数据库|`import { Injectable } from '@nestjs/common';` `import { InjectRepository } from '@nestjs/typeorm';` `import { Repository } from 'typeorm';` `import { User } from './user.entity';` `@Injectable()` `export class UserService {` `constructor(` `@InjectRepository(User)` `private userRepository: Repository<User>,` `) {}` `// 数据库查询示例` `findAll() {` `return this.userRepository.find();` `}` `}`|
|`@InjectModel(ModelName)`|Mongoose 专用，注入 MongoDB 对应的模型（Model），用于操作 MongoDB|用法同 `@InjectRepository()`，需配合 Schema 定义使用|

### 5. 响应与异常装饰器（结果处理类）

用于自定义 HTTP 响应（状态码、响应头、重定向）或处理异常，规范响应格式。

|   |   |   |
|---|---|---|
|**装饰器**|**作用**|**示例**|
|`@HttpCode(code)`|自定义响应状态码（默认 GET 200、POST 201）|`import { Controller, Post, Body, HttpCode, HttpStatus } from '@nestjs/common';` `import { CreateUserDto } from './dto/create-user.dto';` `@Controller('users')` `export class UserController {` `@Post()` `@HttpCode(HttpStatus.CREATED) // 显式指定 201 状态码` `create(@Body() createUserDto: CreateUserDto) {` `return createUserDto;` `}` `}`|
|`@Header(key, value)`|自定义响应头信息|`import { Controller, Get, Header } from '@nestjs/common';` `@Controller('users')` `export class UserController {` `@Get()` `@Header('Cache-Control', 'no-cache') // 禁止缓存` `@Header('X-Custom-Header', 'nestjs-demo') // 自定义头部` `findAll() {` `return '用户列表';` `}` `}`|
|`@Redirect(url, status?)`|实现 URL 重定向，默认 302 临时重定向|`import { Controller, Get, Redirect } from '@nestjs/common';` `@Controller()` `export class AppController {` `// 静态重定向` `@Get('nest')` `@Redirect('https://nestjs.com', 302)` `redirectToNest() {}` `// 动态重定向（返回对象覆盖配置）` `@Get('docs')` `@Redirect('https://nestjs.com', 302)` `redirectToDocs() {` `return { url: 'https://docs.nestjs.com', statusCode: 301 }; // 301 永久重定向` `}` `}`|
|`@Catch(exceptionTypes)`|异常过滤器装饰器，指定需要捕获的异常类型，统一处理异常响应|`import { Catch, ExceptionFilter, ArgumentsHost } from '@nestjs/common';` `import { HttpException } from '@nestjs/common';` `// 只捕获 HttpException 及其子类` `@Catch(HttpException)` `export class HttpExceptionFilter implements ExceptionFilter {` `catch(exception: HttpException, host: ArgumentsHost) {` `const ctx = host.switchToHttp();` `const response = ctx.getResponse();` `const status = exception.getStatus();` `// 统一异常响应格式` `response.status(status).json({` `statusCode: status,` `message: exception.message,` `timestamp: new Date().toISOString(),` `});` `}` `}` `// 控制器中使用` `@Controller('users')` `export class UserController {` `@Get(':id')` `@UseFilters(HttpExceptionFilter) // 应用异常过滤器` `findOne(@Param('id') id: string) {` `if (id === '0') {` `throw new HttpException('ID 不能为 0', 400);` `}` ``return `用户 ID: ${id}`;`` `}` `}`|
|`HttpException`（非装饰器，常用）|手动抛出 HTTP 异常，触发异常处理流程|`import { Controller, Get, Param, HttpException, HttpStatus } from '@nestjs/common';` `@Controller('users')` `export class UserController {` `@Get(':id')` `findOne(@Param('id') id: string) {` `if (!id) {` `throw new HttpException('ID 不能为空', HttpStatus.BAD_REQUEST);` `}` ``return `用户 ID: ${id}`;`` `}` `}`|

### 6. 生命周期装饰器（钩子回调类）

用于监听 NestJS 实例（模块、控制器、服务）的生命周期事件，实现初始化、清理等逻辑。

|   |   |   |
|---|---|---|
|**装饰器/接口**|**作用**|**示例**|
|`OnModuleInit`（接口）|模块初始化完成后执行回调|`import { Injectable, OnModuleInit } from '@nestjs/common';` `@Injectable()` `export class UserService implements OnModuleInit {` `// 模块初始化完成后执行` `onModuleInit() {` `console.log('UserModule 初始化完成，执行数据库连接等初始化逻辑');` `}` `}`|
|`OnModuleDestroy`（接口）|模块销毁前执行回调（用于清理资源）|`import { Injectable, OnModuleDestroy } from '@nestjs/common';` `@Injectable()` `export class UserService implements OnModuleDestroy {` `// 模块销毁前执行` `onModuleDestroy() {` `console.log('UserModule 即将销毁，执行关闭数据库连接等清理逻辑');` `}` `}`|
|`OnInit`（接口）|实例（控制器/服务）初始化完成后执行，触发时机早于 OnModuleInit|用法同 OnModuleInit，仅触发时机不同|

### 7. 其他高频装饰器

除上述分类外，以下装饰器在开发中高频使用，主要用于权限校验、数据处理等场景。

|   |   |   |
|---|---|---|
|**装饰器**|**作用**|**示例**|
|`@UseGuards(guards...)`|应用守卫（用于认证、权限校验），可作用于模块、控制器、路由级别|`import { Controller, Get, UseGuards } from '@nestjs/common';` `import { AuthGuard } from './guards/auth.[guard](../guard/NestJS%20全局守卫排序：框架原生、自定义容器与依赖注入.md)';` `import { RoleGuard } from './guards/role.guard';` `// 控制器级别：所有路由都需认证` `@Controller('users')` `@UseGuards(AuthGuard)` `export class UserController {` `// 路由级别：额外增加角色校验` `@Get('admin')` `@UseGuards(new RoleGuard('admin'))` `getAdminUsers() {` `return '管理员用户列表';` `}` `}`|
|`@UseInterceptors(interceptors...)`|应用拦截器（用于请求/响应拦截、数据转换、日志记录等）|`import { Controller, Get, UseInterceptors } from '@nestjs/common';` `import { UserAgentInterceptor } from './interceptors/user-agent.interceptor';` `@Controller('users')` `@UseInterceptors(UserAgentInterceptor) // 应用 User-Agent 解析拦截器` `export class UserController {` `@Get()` `findAll() {` `return '用户列表';` `}` `}`|
|`@UsePipes(pipes...)`|应用管道（用于数据校验、类型转换），常用 ValidationPipe 做请求体校验|`import { Controller, Post, Body, UsePipes, ValidationPipe } from '@nestjs/common';` `import { CreateUserDto } from './dto/create-user.dto';` `@Controller('users')` `export class UserController {` `@Post()` `// 应用数据校验管道，自动校验请求体是否符合 DTO 规则` `@UsePipes(new ValidationPipe({ whitelist: true }))` `create(@Body() createUserDto: CreateUserDto) {` `return createUserDto;` `}` `}`|

## 二、核心使用原则

1. **装饰器是语法糖**：本质是函数，底层封装了路由注册、依赖注入等逻辑，简化开发流程，但需理解其底层原理。
    
2. **优先精准装饰器**：提取请求数据时，优先用 `@Param()`/`@Body()`/`@Query()`，而非 `@Req()`，减少与底层框架耦合。
    
3. **全局与局部合理搭配**：通用组件（如 ValidationPipe、日志拦截器）全局注册，特殊需求（如某路由的权限守卫）局部注册。
    
4. **类型安全优先**：结合 TypeScript 接口/DTO 使用装饰器，避免 any 类型，提升代码健壮性与可维护性。
    
5. **避免过度装饰**：单个路由/控制器装饰器不超过 4 个，重复组合的装饰器可封装为自定义装饰器（如 `@Auth()` 封装认证+权限校验）。
    

## 三、总结

NestJS 装饰器按功能可归纳为四大核心类别：

- **结构类**：`@Module()`/`@Controller()`/`@Injectable()`，搭建应用骨架；
    
- **路由类**：`@Get()`/`@Post()`/`@Put()`，定义接口路由；
    
- **数据类**：`@Param()`/`@Body()`/`@Query()`，提取请求数据；
    
- **增强类**：`@UseGuards()`/`@UsePipes()`/`@UseInterceptors()`，实现权限、校验、拦截等增强功能。
    

装饰器的核心价值是**解耦**与**标准化**，合理使用能大幅提升代码质量与开发效率。实际开发中，需结合业务场景按需选择，避免为了使用装饰器而过度设计。