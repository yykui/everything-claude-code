---
name: nestjs-patterns
description: NestJS 架构模式，涵盖模块、控制器、提供者、DTO 验证、守卫、拦截器、配置及生产级 TypeScript 后端。
origin: ECC
---

# NestJS 开发模式

用于模块化 TypeScript 后端的生产级 NestJS 模式。

## 何时激活

- 构建 NestJS API 或服务
- 组织模块、控制器和提供者
- 添加 DTO 验证、守卫、拦截器或异常过滤器
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

- 将领域代码保存在功能模块内。
- 将跨切面的过滤器、装饰器、守卫和拦截器放入 `common/`。
- 将 DTO 放在拥有它们的模块附近。

## 启动与全局验证

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

- 公共 API 始终启用 `whitelist` 和 `forbidNonWhitelisted`。
- 优先使用一个全局验证管道，而非在每个路由上重复验证配置。

## 模块、控制器和提供者

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

- 控制器应保持精简：解析 HTTP 输入、调用提供者、返回响应 DTO。
- 将业务逻辑放在可注入的服务中，而非控制器中。
- 只导出其他模块真正需要的提供者。

## DTO 与验证

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

- 使用 `class-validator` 验证每个请求 DTO。
- 使用专用的响应 DTO 或序列化器，而非直接返回 ORM 实体。
- 避免泄露内部字段，如密码哈希、令牌或审计列。

## 认证、守卫和请求上下文

```ts
@UseGuards(JwtAuthGuard, RolesGuard)
@Roles('admin')
@Get('admin/report')
getAdminReport(@Req() req: AuthenticatedRequest) {
  return this.reportService.getForUser(req.user.id);
}
```

- 除非真正共享，否则将认证策略和守卫保持在模块本地。
- 在守卫中编码粗粒度访问规则，然后在服务中进行资源特定的授权。
- 对已认证的请求对象优先使用显式请求类型。

## 异常过滤器与错误结构

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

- 在整个 API 中保持一致的错误信封。
- 对预期的客户端错误抛出框架异常；集中记录并包装意外失败。

## 配置与环境变量验证

```ts
ConfigModule.forRoot({
  isGlobal: true,
  load: [configuration],
  validate: validateEnv,
});
```

- 在启动时验证环境变量，而非在第一次请求时懒加载。
- 将配置访问封装在类型化的辅助函数或配置服务后面。
- 在配置工厂中拆分开发/预发布/生产的关注点，而非在功能代码中到处分支。

## 持久化与事务

- 将仓库 / ORM 代码封装在使用领域语言的提供者后面。
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

- 使用模拟依赖项对提供者进行隔离的单元测试。
- 为守卫、验证管道和异常过滤器添加请求级测试。
- 在测试中复用与生产环境相同的全局管道/过滤器。

## 生产默认设置

- 启用结构化日志和请求关联 ID。
- 在环境/配置无效时终止，而非部分启动。
- 对数据库/缓存客户端优先使用带显式健康检查的异步提供者初始化。
- 将后台任务和事件消费者保留在各自的模块中，而非放在 HTTP 控制器内。
- 对公共端点明确设置速率限制、认证和审计日志。
