---
name: nestjs-patterns
description: NestJS 模块、控制器、Provider、DTO 校验、守卫、拦截器、配置及生产级 TypeScript 后端的架构模式。
origin: ECC
---

# NestJS 开发模式

用于模块化 TypeScript 后端的生产级 NestJS 模式。

## 何时激活

- 构建 NestJS API 或服务
- 组织模块、控制器和 Provider
- 添加 DTO 校验、守卫、拦截器或异常过滤器
- 配置环境感知设置和数据库集成
- 测试 NestJS 单元或 HTTP 端点

## 项目结构

```text
src/
├── app.module.ts
├── main.ts
├── common/
│   ├── filters/
│   ├── guards/
│   ├── interceptors/
│   └── pipes/
├── config/
│   ├── configuration.ts
│   └── validation.ts
├── modules/
│   ├── auth/
│   │   ├── auth.controller.ts
│   │   ├── auth.module.ts
│   │   ├── auth.service.ts
│   │   ├── dto/
│   │   ├── guards/
│   │   └── strategies/
│   └── users/
│       ├── dto/
│       ├── entities/
│       ├── users.controller.ts
│       ├── users.module.ts
│       └── users.service.ts
└── prisma/ or database/
```

- 将领域代码放在功能模块内。
- 将跨切面的过滤器、装饰器、守卫和拦截器放在 `common/` 中。
- 将 DTO 紧靠拥有它们的模块存放。

## 引导与全局校验

```ts
async function bootstrap() {
  const app = await NestFactory.create(AppModule, { bufferLogs: true });

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,
      forbidNonWhitelisted: true,
      transform: true,
      transformOptions: { enableImplicitConversion: true },
    }),
  );

  app.useGlobalInterceptors(new ClassSerializerInterceptor(app.get(Reflector)));
  app.useGlobalFilters(new HttpExceptionFilter());

  await app.listen(process.env.PORT ?? 3000);
}
bootstrap();
```

- 对公开 API 始终启用 `whitelist` 和 `forbidNonWhitelisted`。
- 优先使用一个全局校验管道，而不是在每个路由上重复配置校验。

## 模块、控制器与 Provider

```ts
@Module({
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}

@Controller('users')
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get(':id')
  getById(@Param('id', ParseUUIDPipe) id: string) {
    return this.usersService.getById(id);
  }

  @Post()
  create(@Body() dto: CreateUserDto) {
    return this.usersService.create(dto);
  }
}

@Injectable()
export class UsersService {
  constructor(private readonly usersRepo: UsersRepository) {}

  async create(dto: CreateUserDto) {
    return this.usersRepo.create(dto);
  }
}
```

- 控制器应保持精简：解析 HTTP 输入、调用 Provider、返回响应 DTO。
- 将业务逻辑放在可注入的服务中，而非控制器中。
- 只导出其他模块真正需要的 Provider。

## DTO 与校验

```ts
export class CreateUserDto {
  @IsEmail()
  email!: string;

  @IsString()
  @Length(2, 80)
  name!: string;

  @IsOptional()
  @IsEnum(UserRole)
  role?: UserRole;
}
```

- 使用 `class-validator` 对每个请求 DTO 进行校验。
- 使用专用响应 DTO 或序列化器，而不是直接返回 ORM 实体。
- 避免暴露内部字段，如密码哈希、令牌或审计字段。

## 认证、守卫与请求上下文

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Get('admin/report')
getAdminReport(@Req() req: AuthenticatedRequest) {
  return this.reportService.getForUser(req.user.id);
}
```

- 除非真正共享，否则将认证策略和守卫保持在模块本地。
- 在守卫中编码粗粒度访问规则，然后在服务中进行资源级别的授权。
- 对已认证请求对象优先使用明确的请求类型。

## 异常过滤器与错误形式

```ts
@Catch()
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    const response = host.switchToHttp().getResponse<Response>();
    const request = host.switchToHttp().getRequest<Request>();

    if (exception instanceof HttpException) {
      return response.status(exception.getStatus()).json({
        path: request.url,
        error: exception.getResponse(),
      });
    }

    return response.status(500).json({
      path: request.url,
      error: 'Internal server error',
    });
  }
}
```

- 在整个 API 中保持一致的错误信封格式。
- 对预期的客户端错误抛出框架异常；集中记录并包装意外失败。

## 配置与环境校验

```ts
ConfigModule.forRoot({
  isGlobal: true,
  load: [configuration],
  validate: validateEnv,
});
```

- 在启动时校验环境变量，而不是在第一个请求时懒加载。
- 将配置访问封装在类型化的辅助函数或配置服务后面。
- 在配置工厂中分离开发/预发布/生产关注点，而不是在功能代码中到处添加分支。

## 持久化与事务

- 将仓库 / ORM 代码封装在使用领域语言的 Provider 后面。
- 对于 Prisma 或 TypeORM，将事务性工作流隔离在拥有工作单元的服务中。
- 不要让控制器直接协调多步写入。

## 测试

```ts
describe('UsersController', () => {
  let app: INestApplication;

  beforeAll(async () => {
    const moduleRef = await Test.createTestingModule({
      imports: [UsersModule],
    }).compile();

    app = moduleRef.createNestApplication();
    app.useGlobalPipes(new ValidationPipe({ whitelist: true, transform: true }));
    await app.init();
  });
});
```

- 使用模拟依赖对 Provider 进行单元测试。
- 为守卫、校验管道和异常过滤器添加请求级别测试。
- 在测试中复用与生产相同的全局管道/过滤器。

## 生产默认值

- 启用结构化日志和请求关联 ID。
- 在环境/配置无效时终止启动，而不是部分启动。
- 对数据库/缓存客户端优先使用带有明确健康检查的异步 Provider 初始化。
- 将后台作业和事件消费者放在各自的模块中，而不是 HTTP 控制器内。
- 对公开端点明确配置速率限制、认证和审计日志。
