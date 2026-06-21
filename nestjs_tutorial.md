# Khóa học NestJS — Toàn tập từ cơ bản đến nâng cao

> **Yêu cầu:** Node.js >= 16, npm >= 8, kiến thức cơ bản về TypeScript

---

## Mục lục

1. [Giới thiệu NestJS](#1-giới-thiệu-nestjs)
2. [Cài đặt & Khởi tạo dự án](#2-cài-đặt--khởi-tạo-dự-án)
3. [Modules](#3-modules)
4. [Controllers](#4-controllers)
5. [Providers & Services](#5-providers--services)
6. [Middleware](#6-middleware)
7. [Pipes & Validation](#7-pipes--validation)
8. [Guards (Bảo vệ route)](#8-guards-bảo-vệ-route)
9. [Interceptors](#9-interceptors)
10. [Exception Filters](#10-exception-filters)
11. [Custom Decorators](#11-custom-decorators)
12. [Database với TypeORM](#12-database-với-typeorm)

---

## 1. Giới thiệu NestJS

**NestJS** là một framework Node.js mạnh mẽ được xây dựng trên TypeScript, lấy cảm hứng từ Angular. NestJS cung cấp kiến trúc ứng dụng rõ ràng, dễ mở rộng và bảo trì.

### Tại sao nên dùng NestJS?

| Tính năng | Mô tả |
|-----------|-------|
| **TypeScript first** | Hỗ trợ TypeScript hoàn toàn, giúp code an toàn hơn |
| **Modular** | Chia nhỏ ứng dụng thành các module độc lập |
| **Dependency Injection** | Quản lý dependencies tự động |
| **Decorator-based** | Sử dụng decorator để cấu hình routes, guards, pipes... |
| **Extensible** | Tích hợp dễ dàng với Express hoặc Fastify |

### Kiến trúc tổng quan

```
src/
├── app.module.ts          # Module gốc
├── main.ts                # Điểm khởi chạy ứng dụng
├── user/
│   ├── user.module.ts     # Module người dùng
│   ├── user.controller.ts # Controller xử lý request
│   ├── user.service.ts    # Service chứa business logic
│   └── user.entity.ts     # Entity (model database)
└── product/
    ├── product.module.ts
    ├── product.controller.ts
    └── product.service.ts
```

---

## 2. Cài đặt & Khởi tạo dự án

### Cài đặt NestJS CLI

```bash
npm install -g @nestjs/cli
```

### Tạo dự án mới

```bash
nest new my-project
cd my-project
npm run start:dev
```

### Cấu trúc file `main.ts`

`main.ts` là điểm khởi đầu của ứng dụng NestJS:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  // Cấu hình global prefix cho tất cả routes
  app.setGlobalPrefix('api');

  // Bật CORS
  app.enableCors();

  await app.listen(3000);
  console.log(`Application is running on: http://localhost:3000`);
}
bootstrap();
```

---

## 3. Modules

**Module** là đơn vị tổ chức cơ bản trong NestJS. Mỗi ứng dụng có ít nhất một **root module** (`AppModule`).

### Tạo module mới

Sử dụng CLI để tạo nhanh:

```bash
nest generate module user
# hoặc viết tắt
nest g mo user
```

### Cấu trúc một Module

`user.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { UserController } from './user.controller';
import { UserService } from './user.service';

@Module({
  imports: [],        // Các module khác cần dùng
  controllers: [UserController],  // Controllers thuộc module này
  providers: [UserService],       // Services, repositories...
  exports: [UserService],         // Export để module khác sử dụng
})
export class UserModule {}
```

### Chia sẻ Module giữa các module

Khi muốn dùng `UserService` ở module khác:

`app.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { UserModule } from './user/user.module';
import { ProductModule } from './product/product.module';

@Module({
  imports: [UserModule, ProductModule],
})
export class AppModule {}
```

> **Lưu ý:** Phải `export` service trong module nguồn, và `import` module đó vào module đích.

---

## 4. Controllers

**Controller** chịu trách nhiệm nhận và xử lý HTTP request, sau đó trả về response cho client.

### Tạo Controller

```bash
nest g controller user
```

### Controller cơ bản

`user.controller.ts`:

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Body,
  Param,
  Query,
  HttpCode,
  HttpStatus,
} from '@nestjs/common';
import { UserService } from './user.service';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Controller('users')  // Route prefix: /users
export class UserController {
  constructor(private readonly userService: UserService) {}

  // GET /users
  @Get()
  findAll(@Query('page') page: number = 1) {
    return this.userService.findAll(page);
  }

  // GET /users/:id
  @Get(':id')
  findOne(@Param('id') id: string) {
    return this.userService.findOne(+id);
  }

  // POST /users
  @Post()
  @HttpCode(HttpStatus.CREATED)
  create(@Body() createUserDto: CreateUserDto) {
    return this.userService.create(createUserDto);
  }

  // PUT /users/:id
  @Put(':id')
  update(@Param('id') id: string, @Body() updateUserDto: UpdateUserDto) {
    return this.userService.update(+id, updateUserDto);
  }

  // DELETE /users/:id
  @Delete(':id')
  remove(@Param('id') id: string) {
    return this.userService.remove(+id);
  }
}
```

### Các Decorator thường dùng trong Controller

| Decorator | Mô tả |
|-----------|-------|
| `@Controller('path')` | Định nghĩa route prefix |
| `@Get()`, `@Post()`, `@Put()`, `@Delete()` | HTTP methods |
| `@Param('id')` | Lấy route parameter |
| `@Body()` | Lấy request body |
| `@Query('key')` | Lấy query string |
| `@Headers('key')` | Lấy header |
| `@Req()` | Lấy toàn bộ request object |
| `@Res()` | Lấy toàn bộ response object |
| `@HttpCode(201)` | Đặt HTTP status code |

---

## 5. Providers & Services

**Provider** là bất kỳ class nào có thể được inject vào các class khác thông qua **Dependency Injection**. **Service** là loại Provider phổ biến nhất.

### Tạo Service

```bash
nest g service user
```

### Service cơ bản

`user.service.ts`:

```typescript
import { Injectable, NotFoundException } from '@nestjs/common';
import { CreateUserDto } from './dto/create-user.dto';
import { UpdateUserDto } from './dto/update-user.dto';

@Injectable()  // Đánh dấu class có thể được inject
export class UserService {
  // Giả lập database bằng mảng
  private users = [
    { id: 1, name: 'Nguyễn Văn A', email: 'a@example.com' },
    { id: 2, name: 'Trần Thị B', email: 'b@example.com' },
  ];

  findAll(page: number = 1) {
    const limit = 10;
    const offset = (page - 1) * limit;
    return {
      data: this.users.slice(offset, offset + limit),
      total: this.users.length,
      page,
    };
  }

  findOne(id: number) {
    const user = this.users.find(u => u.id === id);
    if (!user) {
      throw new NotFoundException(`User với id ${id} không tồn tại`);
    }
    return user;
  }

  create(createUserDto: CreateUserDto) {
    const newUser = {
      id: this.users.length + 1,
      ...createUserDto,
    };
    this.users.push(newUser);
    return newUser;
  }

  update(id: number, updateUserDto: UpdateUserDto) {
    const user = this.findOne(id);
    Object.assign(user, updateUserDto);
    return user;
  }

  remove(id: number) {
    const index = this.users.findIndex(u => u.id === id);
    if (index === -1) {
      throw new NotFoundException(`User với id ${id} không tồn tại`);
    }
    this.users.splice(index, 1);
    return { message: 'Xóa thành công' };
  }
}
```

### Data Transfer Object (DTO)

DTO định nghĩa cấu trúc dữ liệu được truyền vào:

`dto/create-user.dto.ts`:

```typescript
import { IsString, IsEmail, IsNotEmpty, MinLength } from 'class-validator';

export class CreateUserDto {
  @IsNotEmpty({ message: 'Tên không được để trống' })
  @IsString()
  name: string;

  @IsEmail({}, { message: 'Email không hợp lệ' })
  email: string;

  @IsNotEmpty()
  @MinLength(6, { message: 'Mật khẩu tối thiểu 6 ký tự' })
  password: string;
}
```

---

## 6. Middleware

**Middleware** là hàm được gọi **trước khi** request đến route handler. Middleware có thể:
- Thực thi code bất kỳ
- Thay đổi request/response object
- Kết thúc request-response cycle
- Gọi `next()` để chuyển sang middleware tiếp theo

### Tạo Middleware

`logger.middleware.ts`:

```typescript
import { Injectable, NestMiddleware } from '@nestjs/common';
import { Request, Response, NextFunction } from 'express';

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const { method, originalUrl } = req;
    const start = Date.now();

    res.on('finish', () => {
      const duration = Date.now() - start;
      console.log(
        `[${new Date().toISOString()}] ${method} ${originalUrl} - ${res.statusCode} (${duration}ms)`
      );
    });

    next(); // Bắt buộc phải gọi next() để tiếp tục xử lý
  }
}
```

### Áp dụng Middleware globally

`main.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { AppModule } from './app.module';
import { LoggerMiddleware } from './logger.middleware';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.use(new LoggerMiddleware().use); // Áp dụng cho toàn bộ routes

  await app.listen(3000);
}
bootstrap();
```

### Áp dụng Middleware cho Module cụ thể

Áp dụng Middleware vào module bằng cách ghi đè phương thức `configure`:

`app.module.ts`:

```typescript
import { Module, MiddlewareConsumer, RequestMethod } from '@nestjs/common';
import { LoggerMiddleware } from './logger.middleware';
import { ProductModule } from './product/product.module';

@Module({
  imports: [ProductModule],
})
export class AppModule {
  configure(consumer: MiddlewareConsumer) {
    consumer
      .apply(LoggerMiddleware)
      .forRoutes({ path: 'products', method: RequestMethod.ALL }); // Middleware chỉ cho route products
  }
}
```

Hoặc áp dụng cho toàn bộ một controller:

```typescript
import { ProductController } from './product/product.controller';

configure(consumer: MiddlewareConsumer) {
  consumer
    .apply(LoggerMiddleware)
    .forRoutes(ProductController); // Áp dụng cho tất cả routes trong ProductController
}
```

---

## 7. Pipes & Validation

**Pipe** được sử dụng để **biến đổi** hoặc **validate** dữ liệu đầu vào trước khi đến route handler.

### Cài đặt thư viện validation

```bash
npm install class-validator class-transformer
```

### Bật ValidationPipe globally

`main.ts`:

```typescript
import { ValidationPipe } from '@nestjs/common';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  app.useGlobalPipes(
    new ValidationPipe({
      whitelist: true,           // Tự động loại bỏ fields không có trong DTO
      forbidNonWhitelisted: true, // Báo lỗi nếu có fields lạ
      transform: true,           // Tự động chuyển đổi kiểu dữ liệu
    })
  );

  await app.listen(3000);
}
```

### Sử dụng Built-in Pipes

```typescript
import { ParseIntPipe, ParseUUIDPipe } from '@nestjs/common';

@Get(':id')
findOne(@Param('id', ParseIntPipe) id: number) {
  // id đã được tự động chuyển thành number
  return this.userService.findOne(id);
}
```

### Custom Pipe

```typescript
import { PipeTransform, Injectable, BadRequestException } from '@nestjs/common';

@Injectable()
export class ParsePositiveIntPipe implements PipeTransform {
  transform(value: any) {
    const val = parseInt(value, 10);
    if (isNaN(val) || val <= 0) {
      throw new BadRequestException('ID phải là số nguyên dương');
    }
    return val;
  }
}
```

Sử dụng Custom Pipe:

```typescript
@Get(':id')
findOne(@Param('id', ParsePositiveIntPipe) id: number) {
  return this.userService.findOne(id);
}
```

---

## 8. Guards (Bảo vệ route)

**Guard** xác định liệu request có được phép tiếp tục xử lý hay không. Thường dùng để **xác thực** (authentication) và **phân quyền** (authorization).

### Tạo Auth Guard

`auth.guard.ts`:

```typescript
import {
  Injectable,
  CanActivate,
  ExecutionContext,
  UnauthorizedException,
} from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { Request } from 'express';

@Injectable()
export class AuthGuard implements CanActivate {
  constructor(private jwtService: JwtService) {}

  canActivate(context: ExecutionContext): boolean {
    const request = context.switchToHttp().getRequest<Request>();
    const token = this.extractTokenFromHeader(request);

    if (!token) {
      throw new UnauthorizedException('Token không được cung cấp');
    }

    try {
      const payload = this.jwtService.verify(token);
      request['user'] = payload; // Gắn user vào request
      return true;
    } catch {
      throw new UnauthorizedException('Token không hợp lệ hoặc đã hết hạn');
    }
  }

  private extractTokenFromHeader(request: Request): string | null {
    const [type, token] = request.headers.authorization?.split(' ') ?? [];
    return type === 'Bearer' ? token : null;
  }
}
```

### Sử dụng Guard

Áp dụng cho một route cụ thể:

```typescript
import { UseGuards } from '@nestjs/common';
import { AuthGuard } from '../auth/auth.guard';

@Controller('users')
export class UserController {
  // Chỉ người dùng đã đăng nhập mới truy cập được
  @Get('profile')
  @UseGuards(AuthGuard)
  getProfile(@Req() req) {
    return req.user;
  }
}
```

Áp dụng cho toàn bộ controller:

```typescript
@Controller('users')
@UseGuards(AuthGuard)  // Bảo vệ tất cả routes trong controller
export class UserController {}
```

### Roles Guard (Phân quyền theo role)

`roles.guard.ts`:

```typescript
import { Injectable, CanActivate, ExecutionContext } from '@nestjs/common';
import { Reflector } from '@nestjs/core';

@Injectable()
export class RolesGuard implements CanActivate {
  constructor(private reflector: Reflector) {}

  canActivate(context: ExecutionContext): boolean {
    const requiredRoles = this.reflector.get<string[]>('roles', context.getHandler());
    if (!requiredRoles) return true;

    const { user } = context.switchToHttp().getRequest();
    return requiredRoles.some(role => user?.roles?.includes(role));
  }
}
```

---

## 9. Interceptors

**Interceptor** cho phép bạn:
- Thêm logic **trước** và **sau** khi route handler thực thi
- Biến đổi kết quả trả về
- Xử lý exceptions
- Mở rộng hành vi cơ bản

### Response Transform Interceptor

`transform.interceptor.ts`:

```typescript
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler,
} from '@nestjs/common';
import { Observable } from 'rxjs';
import { map } from 'rxjs/operators';

export interface ResponseFormat<T> {
  success: boolean;
  data: T;
  timestamp: string;
}

@Injectable()
export class TransformInterceptor<T>
  implements NestInterceptor<T, ResponseFormat<T>>
{
  intercept(
    context: ExecutionContext,
    next: CallHandler,
  ): Observable<ResponseFormat<T>> {
    return next.handle().pipe(
      map(data => ({
        success: true,
        data,
        timestamp: new Date().toISOString(),
      })),
    );
  }
}
```

Áp dụng globally trong `main.ts`:

```typescript
import { TransformInterceptor } from './interceptors/transform.interceptor';

app.useGlobalInterceptors(new TransformInterceptor());
```

Kết quả trả về sẽ có dạng:

```json
{
  "success": true,
  "data": { "id": 1, "name": "Nguyễn Văn A" },
  "timestamp": "2024-01-15T10:30:00.000Z"
}
```

---

## 10. Exception Filters

**Exception Filter** xử lý tập trung tất cả các lỗi (exceptions) xảy ra trong ứng dụng.

### Custom Exception Filter

`http-exception.filter.ts`:

```typescript
import {
  ExceptionFilter,
  Catch,
  ArgumentsHost,
  HttpException,
  HttpStatus,
} from '@nestjs/common';
import { Request, Response } from 'express';

@Catch(HttpException)
export class HttpExceptionFilter implements ExceptionFilter {
  catch(exception: HttpException, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();
    const status = exception.getStatus();
    const exceptionResponse = exception.getResponse();

    const errorResponse = {
      success: false,
      statusCode: status,
      timestamp: new Date().toISOString(),
      path: request.url,
      method: request.method,
      message:
        typeof exceptionResponse === 'object'
          ? (exceptionResponse as any).message
          : exceptionResponse,
    };

    response.status(status).json(errorResponse);
  }
}
```

Áp dụng globally:

```typescript
app.useGlobalFilters(new HttpExceptionFilter());
```

Ví dụ response lỗi:

```json
{
  "success": false,
  "statusCode": 404,
  "timestamp": "2024-01-15T10:30:00.000Z",
  "path": "/api/users/999",
  "method": "GET",
  "message": "User với id 999 không tồn tại"
}
```

---

## 11. Custom Decorators

NestJS cho phép tạo **decorator** riêng để tái sử dụng logic.

### Tạo Parameter Decorator

Lấy thông tin user hiện tại từ request:

`decorators/current-user.decorator.ts`:

```typescript
import { createParamDecorator, ExecutionContext } from '@nestjs/common';

export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;

    // Nếu có data (tên field), trả về field đó
    return data ? user?.[data] : user;
  },
);
```

### Sử dụng Custom Decorator

```typescript
@Get('profile')
@UseGuards(AuthGuard)
getProfile(@CurrentUser() user: any) {
  return user; // Trả về toàn bộ user object
}

@Get('email')
@UseGuards(AuthGuard)
getEmail(@CurrentUser('email') email: string) {
  return { email }; // Chỉ trả về email
}
```

### Tạo Class Decorator (Roles)

`decorators/roles.decorator.ts`:

```typescript
import { SetMetadata } from '@nestjs/common';

export const ROLES_KEY = 'roles';
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);
```

Sử dụng:

```typescript
@Get('admin')
@Roles('admin', 'superadmin')
@UseGuards(AuthGuard, RolesGuard)
adminOnly() {
  return 'Chỉ admin mới thấy được';
}
```

---

## 12. Database với TypeORM

### Cài đặt

```bash
npm install @nestjs/typeorm typeorm pg
# pg là driver cho PostgreSQL
# Dùng mysql2 cho MySQL, sqlite3 cho SQLite
```

### Cấu hình kết nối Database

`app.module.ts`:

```typescript
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user/user.entity';

@Module({
  imports: [
    TypeOrmModule.forRoot({
      type: 'postgres',
      host: process.env.DB_HOST || 'localhost',
      port: +process.env.DB_PORT || 5432,
      username: process.env.DB_USER || 'postgres',
      password: process.env.DB_PASS || 'password',
      database: process.env.DB_NAME || 'mydb',
      entities: [User],
      synchronize: true, // Chỉ dùng trong development!
    }),
  ],
})
export class AppModule {}
```

### Tạo Entity

`user/user.entity.ts`:

```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn,
  UpdateDateColumn,
  BeforeInsert,
} from 'typeorm';
import * as bcrypt from 'bcrypt';

@Entity('users')
export class User {
  @PrimaryGeneratedColumn()
  id: number;

  @Column({ length: 100 })
  name: string;

  @Column({ unique: true })
  email: string;

  @Column({ select: false }) // Không trả về password khi query
  password: string;

  @Column({ default: 'user' })
  role: string;

  @Column({ default: true })
  isActive: boolean;

  @CreateDateColumn()
  createdAt: Date;

  @UpdateDateColumn()
  updatedAt: Date;

  @BeforeInsert()
  async hashPassword() {
    this.password = await bcrypt.hash(this.password, 10);
  }
}
```

### Service với Repository

`user/user.service.ts`:

```typescript
import { Injectable, NotFoundException, ConflictException } from '@nestjs/common';
import { InjectRepository } from '@nestjs/typeorm';
import { Repository } from 'typeorm';
import { User } from './user.entity';
import { CreateUserDto } from './dto/create-user.dto';

@Injectable()
export class UserService {
  constructor(
    @InjectRepository(User)
    private userRepository: Repository<User>,
  ) {}

  async findAll(page = 1, limit = 10) {
    const [users, total] = await this.userRepository.findAndCount({
      where: { isActive: true },
      skip: (page - 1) * limit,
      take: limit,
      order: { createdAt: 'DESC' },
    });

    return { data: users, total, page, totalPages: Math.ceil(total / limit) };
  }

  async findOne(id: number): Promise<User> {
    const user = await this.userRepository.findOne({ where: { id } });
    if (!user) {
      throw new NotFoundException(`User #${id} không tồn tại`);
    }
    return user;
  }

  async create(createUserDto: CreateUserDto): Promise<User> {
    const exists = await this.userRepository.findOne({
      where: { email: createUserDto.email },
    });
    if (exists) {
      throw new ConflictException('Email đã được sử dụng');
    }

    const user = this.userRepository.create(createUserDto);
    return this.userRepository.save(user);
  }

  async remove(id: number): Promise<void> {
    const user = await this.findOne(id);
    await this.userRepository.softDelete(user.id); // Xóa mềm
  }
}
```

### Module với TypeORM

`user/user.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { TypeOrmModule } from '@nestjs/typeorm';
import { User } from './user.entity';
import { UserController } from './user.controller';
import { UserService } from './user.service';

@Module({
  imports: [TypeOrmModule.forFeature([User])], // Đăng ký entity
  controllers: [UserController],
  providers: [UserService],
  exports: [UserService],
})
export class UserModule {}
```

---

## Tóm tắt luồng xử lý Request

```
Client Request
      │
      ▼
  Middleware       ← Logging, CORS, Body parsing
      │
      ▼
   Guards          ← Authentication, Authorization
      │
      ▼
  Interceptors     ← Transform request, Logging
      │
      ▼
    Pipes          ← Validation, Data transformation
      │
      ▼
Route Handler     ← Controller method
      │
      ▼
  Interceptors     ← Transform response
      │
      ▼
Exception Filter  ← Xử lý lỗi (nếu có)
      │
      ▼
Client Response
```

---

## Các lệnh CLI thường dùng

```bash
# Tạo module
nest g module <name>

# Tạo controller
nest g controller <name>

# Tạo service
nest g service <name>

# Tạo resource đầy đủ (module + controller + service + dto)
nest g resource <name>

# Tạo middleware
nest g middleware <name>

# Tạo guard
nest g guard <name>

# Tạo interceptor
nest g interceptor <name>

# Tạo pipe
nest g pipe <name>

# Build production
npm run build

# Chạy development
npm run start:dev

# Chạy production
npm run start:prod
```

---

*Tài liệu này được biên soạn cho mục đích học tập. Phiên bản NestJS: v10.x*
