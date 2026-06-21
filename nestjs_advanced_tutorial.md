# Khóa học NestJS Nâng Cao — Phần 2

> **Yêu cầu:** Đã hoàn thành Phần 1 (Modules, Controllers, Services, Middleware, Guards, Pipes, Interceptors)

---

## Mục lục

1. [JWT Authentication hoàn chỉnh](#1-jwt-authentication-hoàn-chỉnh)
2. [Configuration & Environment Variables](#2-configuration--environment-variables)
3. [Caching](#3-caching)
4. [WebSockets (Real-time)](#4-websockets-real-time)
5. [Task Scheduling (Cron Jobs)](#5-task-scheduling-cron-jobs)
6. [File Upload](#6-file-upload)
7. [Swagger / OpenAPI Documentation](#7-swagger--openapi-documentation)
8. [Unit Testing & E2E Testing](#8-unit-testing--e2e-testing)
9. [Microservices](#9-microservices)
10. [Rate Limiting](#10-rate-limiting)
11. [Health Checks](#11-health-checks)
12. [Event Emitter](#12-event-emitter)
13. [Queue với Bull (Background Jobs)](#13-queue-với-bull-background-jobs)
14. [Logging nâng cao với Winston](#14-logging-nâng-cao-với-winston)

---

## 1. JWT Authentication hoàn chỉnh

### Cài đặt

```bash
npm install @nestjs/jwt @nestjs/passport passport passport-jwt passport-local
npm install -D @types/passport-jwt @types/passport-local
npm install bcrypt
npm install -D @types/bcrypt
```

### Auth Module structure

```
src/
└── auth/
    ├── auth.module.ts
    ├── auth.controller.ts
    ├── auth.service.ts
    ├── strategies/
    │   ├── jwt.strategy.ts
    │   └── local.strategy.ts
    ├── guards/
    │   ├── jwt-auth.guard.ts
    │   └── local-auth.guard.ts
    └── dto/
        ├── login.dto.ts
        └── register.dto.ts
```

### Local Strategy (xác thực bằng username/password)

`strategies/local.strategy.ts`:

```typescript
import { Injectable, UnauthorizedException } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { Strategy } from 'passport-local';
import { AuthService } from '../auth.service';

@Injectable()
export class LocalStrategy extends PassportStrategy(Strategy) {
  constructor(private authService: AuthService) {
    super({ usernameField: 'email' }); // Dùng email thay vì username
  }

  async validate(email: string, password: string) {
    const user = await this.authService.validateUser(email, password);
    if (!user) {
      throw new UnauthorizedException('Email hoặc mật khẩu không đúng');
    }
    return user;
  }
}
```

### JWT Strategy (xác thực bằng token)

`strategies/jwt.strategy.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { PassportStrategy } from '@nestjs/passport';
import { ExtractJwt, Strategy } from 'passport-jwt';
import { ConfigService } from '@nestjs/config';

@Injectable()
export class JwtStrategy extends PassportStrategy(Strategy) {
  constructor(private configService: ConfigService) {
    super({
      jwtFromRequest: ExtractJwt.fromAuthHeaderAsBearerToken(),
      ignoreExpiration: false,
      secretOrKey: configService.get<string>('JWT_SECRET'),
    });
  }

  async validate(payload: any) {
    // payload là nội dung đã được giải mã từ JWT
    return {
      id: payload.sub,
      email: payload.email,
      role: payload.role,
    };
  }
}
```

### Auth Service

`auth.service.ts`:

```typescript
import { Injectable } from '@nestjs/common';
import { JwtService } from '@nestjs/jwt';
import { UserService } from '../user/user.service';
import * as bcrypt from 'bcrypt';
import { RegisterDto } from './dto/register.dto';

@Injectable()
export class AuthService {
  constructor(
    private userService: UserService,
    private jwtService: JwtService,
  ) {}

  // Xác thực user với email + password
  async validateUser(email: string, password: string) {
    const user = await this.userService.findByEmail(email);
    if (user && (await bcrypt.compare(password, user.password))) {
      const { password: _, ...result } = user;
      return result;
    }
    return null;
  }

  // Đăng nhập → trả về access_token & refresh_token
  async login(user: any) {
    const payload = { email: user.email, sub: user.id, role: user.role };

    return {
      access_token: this.jwtService.sign(payload, { expiresIn: '15m' }),
      refresh_token: this.jwtService.sign(payload, { expiresIn: '7d' }),
      user: {
        id: user.id,
        name: user.name,
        email: user.email,
        role: user.role,
      },
    };
  }

  // Đăng ký tài khoản mới
  async register(registerDto: RegisterDto) {
    return this.userService.create(registerDto);
  }

  // Làm mới access_token bằng refresh_token
  async refreshToken(refreshToken: string) {
    try {
      const payload = this.jwtService.verify(refreshToken);
      const newPayload = { email: payload.email, sub: payload.sub, role: payload.role };
      return {
        access_token: this.jwtService.sign(newPayload, { expiresIn: '15m' }),
      };
    } catch {
      throw new Error('Refresh token không hợp lệ');
    }
  }
}
```

### Auth Controller

`auth.controller.ts`:

```typescript
import { Controller, Post, Body, UseGuards, Req, Get } from '@nestjs/common';
import { AuthGuard } from '@nestjs/passport';
import { AuthService } from './auth.service';
import { RegisterDto } from './dto/register.dto';

@Controller('auth')
export class AuthController {
  constructor(private authService: AuthService) {}

  // POST /auth/register
  @Post('register')
  register(@Body() registerDto: RegisterDto) {
    return this.authService.register(registerDto);
  }

  // POST /auth/login — LocalStrategy tự động validate
  @Post('login')
  @UseGuards(AuthGuard('local'))
  login(@Req() req) {
    return this.authService.login(req.user);
  }

  // GET /auth/profile — JwtStrategy bảo vệ route
  @Get('profile')
  @UseGuards(AuthGuard('jwt'))
  getProfile(@Req() req) {
    return req.user;
  }

  // POST /auth/refresh
  @Post('refresh')
  refreshToken(@Body('refresh_token') refreshToken: string) {
    return this.authService.refreshToken(refreshToken);
  }
}
```

### Auth Module

`auth.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { JwtModule } from '@nestjs/jwt';
import { PassportModule } from '@nestjs/passport';
import { ConfigModule, ConfigService } from '@nestjs/config';
import { AuthController } from './auth.controller';
import { AuthService } from './auth.service';
import { LocalStrategy } from './strategies/local.strategy';
import { JwtStrategy } from './strategies/jwt.strategy';
import { UserModule } from '../user/user.module';

@Module({
  imports: [
    UserModule,
    PassportModule,
    JwtModule.registerAsync({
      imports: [ConfigModule],
      useFactory: (configService: ConfigService) => ({
        secret: configService.get<string>('JWT_SECRET'),
        signOptions: { expiresIn: '15m' },
      }),
      inject: [ConfigService],
    }),
  ],
  controllers: [AuthController],
  providers: [AuthService, LocalStrategy, JwtStrategy],
  exports: [AuthService],
})
export class AuthModule {}
```

---

## 2. Configuration & Environment Variables

### Cài đặt

```bash
npm install @nestjs/config
```

### File `.env`

```env
# App
NODE_ENV=development
PORT=3000
APP_NAME=MyNestApp

# Database
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASS=secret
DB_NAME=mydb

# JWT
JWT_SECRET=super_secret_key_change_in_production
JWT_EXPIRES=15m
REFRESH_TOKEN_SECRET=another_super_secret

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
```

### Cấu hình ConfigModule

`app.module.ts`:

```typescript
import { ConfigModule } from '@nestjs/config';
import * as Joi from 'joi'; // npm install joi

@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true,        // Dùng được ở mọi nơi không cần import lại
      envFilePath: '.env',
      // Validation schema đảm bảo các biến môi trường bắt buộc phải có
      validationSchema: Joi.object({
        NODE_ENV: Joi.string().valid('development', 'production', 'test').required(),
        PORT: Joi.number().default(3000),
        DB_HOST: Joi.string().required(),
        JWT_SECRET: Joi.string().min(16).required(),
      }),
    }),
  ],
})
export class AppModule {}
```

### Custom Config file

`config/database.config.ts`:

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('database', () => ({
  host: process.env.DB_HOST,
  port: parseInt(process.env.DB_PORT, 10) || 5432,
  username: process.env.DB_USER,
  password: process.env.DB_PASS,
  name: process.env.DB_NAME,
}));
```

`config/jwt.config.ts`:

```typescript
import { registerAs } from '@nestjs/config';

export default registerAs('jwt', () => ({
  secret: process.env.JWT_SECRET,
  expiresIn: process.env.JWT_EXPIRES || '15m',
  refreshSecret: process.env.REFRESH_TOKEN_SECRET,
}));
```

Sử dụng trong Service:

```typescript
import { ConfigService } from '@nestjs/config';

@Injectable()
export class AppService {
  constructor(private configService: ConfigService) {}

  getDbConfig() {
    // Truy cập namespace config
    return this.configService.get('database');
  }

  getPort() {
    return this.configService.get<number>('PORT');
  }
}
```

---

## 3. Caching

### Cài đặt

```bash
npm install @nestjs/cache-manager cache-manager
# Dùng Redis làm cache store
npm install cache-manager-ioredis-yet ioredis
```

### Cấu hình Cache với Redis

`app.module.ts`:

```typescript
import { CacheModule } from '@nestjs/cache-manager';
import { redisStore } from 'cache-manager-ioredis-yet';

@Module({
  imports: [
    CacheModule.registerAsync({
      isGlobal: true,
      useFactory: async () => ({
        store: await redisStore({
          host: process.env.REDIS_HOST || 'localhost',
          port: +process.env.REDIS_PORT || 6379,
        }),
        ttl: 60,  // Cache 60 giây mặc định
      }),
    }),
  ],
})
export class AppModule {}
```

### Dùng Cache trong Service

```typescript
import { CACHE_MANAGER } from '@nestjs/cache-manager';
import { Inject, Injectable } from '@nestjs/common';
import { Cache } from 'cache-manager';

@Injectable()
export class ProductService {
  constructor(@Inject(CACHE_MANAGER) private cacheManager: Cache) {}

  async findAll() {
    // Thử lấy từ cache trước
    const cached = await this.cacheManager.get<any[]>('all_products');
    if (cached) {
      console.log('Lấy từ cache!');
      return cached;
    }

    // Không có cache → query database
    const products = await this.productRepository.find();

    // Lưu vào cache 5 phút
    await this.cacheManager.set('all_products', products, 300000);
    return products;
  }

  async update(id: number, data: any) {
    const product = await this.productRepository.update(id, data);

    // Xóa cache khi có thay đổi
    await this.cacheManager.del('all_products');
    await this.cacheManager.del(`product_${id}`);

    return product;
  }
}
```

### Cache Interceptor (tự động cache route)

```typescript
import { CacheInterceptor, CacheKey, CacheTTL } from '@nestjs/cache-manager';

@Controller('products')
@UseInterceptors(CacheInterceptor) // Tự động cache response
export class ProductController {
  // Cache 30 giây với key tùy chỉnh
  @Get()
  @CacheKey('all_products')
  @CacheTTL(30000)
  findAll() {
    return this.productService.findAll();
  }
}
```

---

## 4. WebSockets (Real-time)

### Cài đặt

```bash
npm install @nestjs/websockets @nestjs/platform-socket.io socket.io
```

### Tạo WebSocket Gateway

`chat/chat.gateway.ts`:

```typescript
import {
  WebSocketGateway,
  WebSocketServer,
  SubscribeMessage,
  MessageBody,
  ConnectedSocket,
  OnGatewayConnection,
  OnGatewayDisconnect,
} from '@nestjs/websockets';
import { Server, Socket } from 'socket.io';
import { UseGuards } from '@nestjs/common';

@WebSocketGateway({
  cors: { origin: '*' },
  namespace: '/chat',  // Namespace: ws://localhost:3000/chat
})
export class ChatGateway implements OnGatewayConnection, OnGatewayDisconnect {
  @WebSocketServer()
  server: Server;

  private connectedUsers = new Map<string, string>(); // socketId → username

  // Gọi khi client kết nối
  handleConnection(client: Socket) {
    console.log(`Client connected: ${client.id}`);
  }

  // Gọi khi client ngắt kết nối
  handleDisconnect(client: Socket) {
    const username = this.connectedUsers.get(client.id);
    this.connectedUsers.delete(client.id);
    this.server.emit('user_left', { username, message: `${username} đã rời phòng` });
    console.log(`Client disconnected: ${client.id}`);
  }

  // Lắng nghe event 'join_room'
  @SubscribeMessage('join_room')
  handleJoinRoom(
    @MessageBody() data: { username: string; room: string },
    @ConnectedSocket() client: Socket,
  ) {
    this.connectedUsers.set(client.id, data.username);
    client.join(data.room); // Tham gia room

    // Thông báo cho tất cả trong room
    this.server.to(data.room).emit('user_joined', {
      username: data.username,
      message: `${data.username} đã vào phòng`,
    });
  }

  // Lắng nghe event 'send_message'
  @SubscribeMessage('send_message')
  handleMessage(
    @MessageBody() data: { room: string; message: string },
    @ConnectedSocket() client: Socket,
  ) {
    const username = this.connectedUsers.get(client.id);

    // Broadcast tới tất cả trong room
    this.server.to(data.room).emit('receive_message', {
      username,
      message: data.message,
      timestamp: new Date().toISOString(),
    });
  }

  // Gửi thông báo từ server đến client cụ thể
  sendNotification(userId: string, notification: any) {
    this.server.to(userId).emit('notification', notification);
  }
}
```

### Client-side (JavaScript)

```javascript
import { io } from 'socket.io-client';

const socket = io('http://localhost:3000/chat');

// Kết nối và tham gia phòng
socket.on('connect', () => {
  socket.emit('join_room', { username: 'Nguyễn Văn A', room: 'general' });
});

// Nhận tin nhắn
socket.on('receive_message', (data) => {
  console.log(`${data.username}: ${data.message}`);
});

// Gửi tin nhắn
function sendMessage(message) {
  socket.emit('send_message', { room: 'general', message });
}
```

---

## 5. Task Scheduling (Cron Jobs)

### Cài đặt

```bash
npm install @nestjs/schedule
npm install -D @types/cron
```

### Cấu hình

`app.module.ts`:

```typescript
import { ScheduleModule } from '@nestjs/schedule';

@Module({
  imports: [ScheduleModule.forRoot()],
})
export class AppModule {}
```

### Tạo Task Service

`tasks/tasks.service.ts`:

```typescript
import { Injectable, Logger } from '@nestjs/common';
import { Cron, CronExpression, Interval, Timeout } from '@nestjs/schedule';

@Injectable()
export class TasksService {
  private readonly logger = new Logger(TasksService.name);

  // Chạy mỗi giờ
  @Cron(CronExpression.EVERY_HOUR)
  async cleanExpiredTokens() {
    this.logger.log('Đang xóa tokens hết hạn...');
    // await this.tokenService.deleteExpired();
  }

  // Chạy lúc 0:00 mỗi ngày
  @Cron('0 0 * * *')
  async generateDailyReport() {
    this.logger.log('Tạo báo cáo hàng ngày...');
    // await this.reportService.generateDaily();
  }

  // Chạy mỗi thứ Hai lúc 9:00 sáng
  @Cron('0 9 * * 1')
  async sendWeeklyNewsletter() {
    this.logger.log('Gửi newsletter hàng tuần...');
  }

  // Chạy mỗi 30 giây
  @Interval(30000)
  async checkSystemHealth() {
    this.logger.debug('Kiểm tra sức khỏe hệ thống...');
  }

  // Chạy 1 lần sau 5 giây khi app khởi động
  @Timeout(5000)
  async onAppStart() {
    this.logger.log('App đã khởi động được 5 giây!');
  }
}
```

### Cron Expression cheatsheet

| Expression | Ý nghĩa |
|------------|---------|
| `* * * * *` | Mỗi phút |
| `*/5 * * * *` | Mỗi 5 phút |
| `0 * * * *` | Mỗi giờ |
| `0 0 * * *` | Mỗi ngày lúc 0:00 |
| `0 0 * * 1` | Mỗi thứ Hai lúc 0:00 |
| `0 9 1 * *` | Ngày 1 mỗi tháng lúc 9:00 |

---

## 6. File Upload

### Cài đặt

```bash
npm install multer
npm install -D @types/multer
```

### Upload một file

`upload/upload.controller.ts`:

```typescript
import {
  Controller,
  Post,
  UploadedFile,
  UseInterceptors,
  BadRequestException,
  Get,
  Param,
  Res,
} from '@nestjs/common';
import { FileInterceptor } from '@nestjs/platform-express';
import { diskStorage } from 'multer';
import { extname, join } from 'path';
import { Response } from 'express';
import { v4 as uuidv4 } from 'uuid';

@Controller('upload')
export class UploadController {

  @Post('image')
  @UseInterceptors(
    FileInterceptor('file', {
      storage: diskStorage({
        destination: './uploads/images',
        filename: (req, file, cb) => {
          // Tạo tên file unique
          const uniqueName = `${uuidv4()}${extname(file.originalname)}`;
          cb(null, uniqueName);
        },
      }),
      fileFilter: (req, file, cb) => {
        // Chỉ chấp nhận ảnh
        const allowedMimes = ['image/jpeg', 'image/png', 'image/webp'];
        if (allowedMimes.includes(file.mimetype)) {
          cb(null, true);
        } else {
          cb(new BadRequestException('Chỉ chấp nhận file ảnh JPG, PNG, WEBP'), false);
        }
      },
      limits: { fileSize: 5 * 1024 * 1024 }, // Tối đa 5MB
    }),
  )
  uploadImage(@UploadedFile() file: Express.Multer.File) {
    if (!file) throw new BadRequestException('Không có file được upload');

    return {
      filename: file.filename,
      originalname: file.originalname,
      size: file.size,
      url: `/upload/images/${file.filename}`,
    };
  }

  // Phục vụ file
  @Get('images/:filename')
  getImage(@Param('filename') filename: string, @Res() res: Response) {
    const filePath = join(process.cwd(), 'uploads', 'images', filename);
    return res.sendFile(filePath);
  }
}
```

### Upload nhiều file

```typescript
import { FilesInterceptor } from '@nestjs/platform-express';

@Post('multiple')
@UseInterceptors(FilesInterceptor('files', 10)) // Tối đa 10 files
uploadMultiple(@UploadedFiles() files: Express.Multer.File[]) {
  return files.map(file => ({
    filename: file.filename,
    size: file.size,
    url: `/upload/images/${file.filename}`,
  }));
}
```

### Upload lên AWS S3

```bash
npm install @aws-sdk/client-s3 @aws-sdk/lib-storage
```

```typescript
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';

@Injectable()
export class S3Service {
  private s3 = new S3Client({
    region: process.env.AWS_REGION,
    credentials: {
      accessKeyId: process.env.AWS_ACCESS_KEY,
      secretAccessKey: process.env.AWS_SECRET_KEY,
    },
  });

  async upload(file: Express.Multer.File): Promise<string> {
    const key = `uploads/${uuidv4()}-${file.originalname}`;

    await this.s3.send(new PutObjectCommand({
      Bucket: process.env.AWS_BUCKET,
      Key: key,
      Body: file.buffer,
      ContentType: file.mimetype,
    }));

    return `https://${process.env.AWS_BUCKET}.s3.${process.env.AWS_REGION}.amazonaws.com/${key}`;
  }
}
```

---

## 7. Swagger / OpenAPI Documentation

### Cài đặt

```bash
npm install @nestjs/swagger
```

### Cấu hình Swagger

`main.ts`:

```typescript
import { SwaggerModule, DocumentBuilder } from '@nestjs/swagger';

async function bootstrap() {
  const app = await NestFactory.create(AppModule);

  const config = new DocumentBuilder()
    .setTitle('My API')
    .setDescription('API documentation cho dự án NestJS')
    .setVersion('1.0')
    .addBearerAuth()  // Thêm nút Authorize cho JWT
    .addTag('users', 'Quản lý người dùng')
    .addTag('products', 'Quản lý sản phẩm')
    .build();

  const document = SwaggerModule.createDocument(app, config);
  SwaggerModule.setup('docs', app, document); // Truy cập tại /docs

  await app.listen(3000);
}
```

### Decorator Swagger

```typescript
import {
  ApiTags,
  ApiOperation,
  ApiResponse,
  ApiBearerAuth,
  ApiParam,
  ApiQuery,
  ApiBody,
} from '@nestjs/swagger';

@ApiTags('users')
@ApiBearerAuth() // Đánh dấu route cần JWT
@Controller('users')
export class UserController {

  @Get()
  @ApiOperation({ summary: 'Lấy danh sách người dùng' })
  @ApiQuery({ name: 'page', required: false, type: Number, description: 'Số trang' })
  @ApiQuery({ name: 'limit', required: false, type: Number, description: 'Số item mỗi trang' })
  @ApiResponse({ status: 200, description: 'Thành công' })
  @ApiResponse({ status: 401, description: 'Chưa đăng nhập' })
  findAll(@Query('page') page = 1, @Query('limit') limit = 10) {
    return this.userService.findAll(page, limit);
  }

  @Get(':id')
  @ApiOperation({ summary: 'Lấy thông tin 1 người dùng' })
  @ApiParam({ name: 'id', type: Number, description: 'ID người dùng' })
  @ApiResponse({ status: 404, description: 'Không tìm thấy' })
  findOne(@Param('id', ParseIntPipe) id: number) {
    return this.userService.findOne(id);
  }
}
```

### Đánh dấu DTO cho Swagger

```typescript
import { ApiProperty, ApiPropertyOptional } from '@nestjs/swagger';

export class CreateUserDto {
  @ApiProperty({ example: 'Nguyễn Văn A', description: 'Họ và tên' })
  @IsString()
  name: string;

  @ApiProperty({ example: 'user@example.com', description: 'Địa chỉ email' })
  @IsEmail()
  email: string;

  @ApiProperty({ example: 'password123', minLength: 6 })
  @MinLength(6)
  password: string;

  @ApiPropertyOptional({ example: 'user', enum: ['user', 'admin'] })
  role?: string;
}
```

---

## 8. Unit Testing & E2E Testing

### Kiến trúc Testing trong NestJS

```
test/
├── app.e2e-spec.ts          # End-to-end tests
src/
└── user/
    ├── user.service.spec.ts # Unit test cho Service
    └── user.controller.spec.ts # Unit test cho Controller
```

### Unit Test — Service

`user/user.service.spec.ts`:

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { UserService } from './user.service';
import { getRepositoryToken } from '@nestjs/typeorm';
import { User } from './user.entity';
import { Repository } from 'typeorm';
import { NotFoundException } from '@nestjs/common';

// Tạo mock repository
const mockUserRepository = {
  find: jest.fn(),
  findOne: jest.fn(),
  create: jest.fn(),
  save: jest.fn(),
  delete: jest.fn(),
  findAndCount: jest.fn(),
};

describe('UserService', () => {
  let service: UserService;
  let repository: Repository<User>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        UserService,
        {
          provide: getRepositoryToken(User),
          useValue: mockUserRepository,
        },
      ],
    }).compile();

    service = module.get<UserService>(UserService);
    repository = module.get<Repository<User>>(getRepositoryToken(User));
  });

  afterEach(() => jest.clearAllMocks());

  describe('findOne', () => {
    it('should return a user when found', async () => {
      const mockUser = { id: 1, name: 'Test User', email: 'test@test.com' };
      mockUserRepository.findOne.mockResolvedValue(mockUser);

      const result = await service.findOne(1);

      expect(result).toEqual(mockUser);
      expect(mockUserRepository.findOne).toHaveBeenCalledWith({ where: { id: 1 } });
    });

    it('should throw NotFoundException when user not found', async () => {
      mockUserRepository.findOne.mockResolvedValue(null);

      await expect(service.findOne(999)).rejects.toThrow(NotFoundException);
    });
  });

  describe('create', () => {
    it('should create and return a new user', async () => {
      const dto = { name: 'New User', email: 'new@test.com', password: '123456' };
      const savedUser = { id: 1, ...dto };

      mockUserRepository.findOne.mockResolvedValue(null);
      mockUserRepository.create.mockReturnValue(savedUser);
      mockUserRepository.save.mockResolvedValue(savedUser);

      const result = await service.create(dto as any);

      expect(result).toEqual(savedUser);
      expect(mockUserRepository.save).toHaveBeenCalled();
    });
  });
});
```

### E2E Test

`test/user.e2e-spec.ts`:

```typescript
import { Test, TestingModule } from '@nestjs/testing';
import { INestApplication, ValidationPipe } from '@nestjs/common';
import * as request from 'supertest';
import { AppModule } from '../src/app.module';

describe('UserController (e2e)', () => {
  let app: INestApplication;
  let jwtToken: string;

  beforeAll(async () => {
    const moduleFixture: TestingModule = await Test.createTestingModule({
      imports: [AppModule],
    }).compile();

    app = moduleFixture.createNestApplication();
    app.useGlobalPipes(new ValidationPipe());
    await app.init();

    // Đăng nhập để lấy JWT token
    const loginRes = await request(app.getHttpServer())
      .post('/auth/login')
      .send({ email: 'admin@test.com', password: '123456' });
    jwtToken = loginRes.body.access_token;
  });

  afterAll(async () => {
    await app.close();
  });

  it('GET /users → 200', () => {
    return request(app.getHttpServer())
      .get('/users')
      .set('Authorization', `Bearer ${jwtToken}`)
      .expect(200)
      .expect(res => {
        expect(res.body).toHaveProperty('data');
        expect(Array.isArray(res.body.data)).toBe(true);
      });
  });

  it('GET /users/:id → 404 khi không tồn tại', () => {
    return request(app.getHttpServer())
      .get('/users/99999')
      .set('Authorization', `Bearer ${jwtToken}`)
      .expect(404);
  });

  it('POST /users → 201 tạo user mới', () => {
    return request(app.getHttpServer())
      .post('/users')
      .set('Authorization', `Bearer ${jwtToken}`)
      .send({ name: 'Test User', email: 'test@example.com', password: '123456' })
      .expect(201)
      .expect(res => {
        expect(res.body).toHaveProperty('id');
        expect(res.body.email).toBe('test@example.com');
      });
  });
});
```

---

## 9. Microservices

### Cài đặt

```bash
npm install @nestjs/microservices
# Dùng Redis làm transport
npm install ioredis
```

### Microservice Server

`main.microservice.ts`:

```typescript
import { NestFactory } from '@nestjs/core';
import { MicroserviceOptions, Transport } from '@nestjs/microservices';
import { AppModule } from './app.module';

async function bootstrap() {
  const app = await NestFactory.createMicroservice<MicroserviceOptions>(
    AppModule,
    {
      transport: Transport.REDIS,
      options: {
        host: 'localhost',
        port: 6379,
      },
    },
  );

  await app.listen();
  console.log('Microservice is listening');
}
bootstrap();
```

### Message Handler

`user/user.controller.ts` (trong microservice):

```typescript
import { Controller } from '@nestjs/common';
import { MessagePattern, Payload, EventPattern } from '@nestjs/microservices';

@Controller()
export class UserController {
  // Lắng nghe message và trả về response
  @MessagePattern({ cmd: 'get_user' })
  async getUser(@Payload() data: { id: number }) {
    return this.userService.findOne(data.id);
  }

  // Lắng nghe event (không trả về response)
  @EventPattern('user_created')
  async onUserCreated(@Payload() data: any) {
    console.log('User created event received:', data);
    await this.emailService.sendWelcomeEmail(data.email);
  }
}
```

### Client gọi Microservice

```typescript
import { ClientProxy, ClientProxyFactory, Transport } from '@nestjs/microservices';

@Injectable()
export class AppService {
  private client: ClientProxy;

  constructor() {
    this.client = ClientProxyFactory.create({
      transport: Transport.REDIS,
      options: { host: 'localhost', port: 6379 },
    });
  }

  async getUser(id: number) {
    // Gửi message và nhận response (Observable)
    return this.client.send({ cmd: 'get_user' }, { id }).toPromise();
  }

  async notifyUserCreated(user: any) {
    // Gửi event (không chờ response)
    this.client.emit('user_created', user);
  }
}
```

---

## 10. Rate Limiting

### Cài đặt

```bash
npm install @nestjs/throttler
```

### Cấu hình global

`app.module.ts`:

```typescript
import { ThrottlerModule, ThrottlerGuard } from '@nestjs/throttler';
import { APP_GUARD } from '@nestjs/core';

@Module({
  imports: [
    ThrottlerModule.forRoot([
      {
        name: 'short',
        ttl: 1000,    // 1 giây
        limit: 5,     // Tối đa 5 requests/giây
      },
      {
        name: 'long',
        ttl: 60000,   // 1 phút
        limit: 100,   // Tối đa 100 requests/phút
      },
    ]),
  ],
  providers: [
    {
      provide: APP_GUARD,
      useClass: ThrottlerGuard, // Apply globally
    },
  ],
})
export class AppModule {}
```

### Tùy chỉnh theo route

```typescript
import { Throttle, SkipThrottle } from '@nestjs/throttler';

@Controller('auth')
export class AuthController {
  // Giới hạn chặt hơn cho login (tránh brute force)
  @Post('login')
  @Throttle({ short: { limit: 3, ttl: 60000 } }) // 3 lần/phút
  login() {}

  // Bỏ qua rate limiting cho route này
  @Get('health')
  @SkipThrottle()
  health() {}
}
```

---

## 11. Health Checks

### Cài đặt

```bash
npm install @nestjs/terminus
```

### Health Module

`health/health.module.ts`:

```typescript
import { Module } from '@nestjs/common';
import { TerminusModule } from '@nestjs/terminus';
import { HttpModule } from '@nestjs/axios';
import { HealthController } from './health.controller';

@Module({
  imports: [TerminusModule, HttpModule],
  controllers: [HealthController],
})
export class HealthModule {}
```

`health/health.controller.ts`:

```typescript
import { Controller, Get } from '@nestjs/common';
import {
  HealthCheck,
  HealthCheckService,
  TypeOrmHealthIndicator,
  MemoryHealthIndicator,
  DiskHealthIndicator,
  HttpHealthIndicator,
} from '@nestjs/terminus';

@Controller('health')
export class HealthController {
  constructor(
    private health: HealthCheckService,
    private db: TypeOrmHealthIndicator,
    private memory: MemoryHealthIndicator,
    private disk: DiskHealthIndicator,
    private http: HttpHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      // Kiểm tra database
      () => this.db.pingCheck('database'),

      // Kiểm tra RAM (tối đa 512MB)
      () => this.memory.checkHeap('memory_heap', 512 * 1024 * 1024),

      // Kiểm tra disk (tối đa 90% usage)
      () => this.disk.checkStorage('disk', { thresholdPercent: 0.9, path: '/' }),

      // Kiểm tra external API
      () => this.http.pingCheck('external_api', 'https://api.example.com/health'),
    ]);
  }
}
```

Response mẫu:

```json
{
  "status": "ok",
  "info": {
    "database": { "status": "up" },
    "memory_heap": { "status": "up" },
    "disk": { "status": "up" }
  },
  "error": {},
  "details": {
    "database": { "status": "up" },
    "memory_heap": { "status": "up" },
    "disk": { "status": "up" }
  }
}
```

---

## 12. Event Emitter

### Cài đặt

```bash
npm install @nestjs/event-emitter
```

### Cấu hình

```typescript
import { EventEmitterModule } from '@nestjs/event-emitter';

@Module({
  imports: [EventEmitterModule.forRoot()],
})
export class AppModule {}
```

### Định nghĩa Events

`events/user.events.ts`:

```typescript
export class UserCreatedEvent {
  constructor(
    public readonly userId: number,
    public readonly email: string,
    public readonly name: string,
  ) {}
}

export class UserDeletedEvent {
  constructor(public readonly userId: number) {}
}
```

### Phát Event

```typescript
import { EventEmitter2 } from '@nestjs/event-emitter';
import { UserCreatedEvent } from '../events/user.events';

@Injectable()
export class UserService {
  constructor(private eventEmitter: EventEmitter2) {}

  async create(dto: CreateUserDto) {
    const user = await this.userRepository.save(dto);

    // Phát event sau khi tạo user thành công
    this.eventEmitter.emit(
      'user.created',
      new UserCreatedEvent(user.id, user.email, user.name),
    );

    return user;
  }
}
```

### Lắng nghe Event

```typescript
import { OnEvent } from '@nestjs/event-emitter';
import { UserCreatedEvent } from '../events/user.events';

@Injectable()
export class NotificationService {
  @OnEvent('user.created')
  async handleUserCreated(event: UserCreatedEvent) {
    // Gửi email chào mừng
    await this.emailService.sendWelcome(event.email, event.name);
    console.log(`Đã gửi email chào mừng tới ${event.email}`);
  }

  @OnEvent('user.created', { async: true })
  async syncToAnalytics(event: UserCreatedEvent) {
    // Đồng bộ dữ liệu lên analytics
    await this.analyticsService.track('new_user', { userId: event.userId });
  }
}
```

---

## 13. Queue với Bull (Background Jobs)

### Cài đặt

```bash
npm install @nestjs/bull bull
npm install -D @types/bull
```

### Cấu hình

`app.module.ts`:

```typescript
import { BullModule } from '@nestjs/bull';

@Module({
  imports: [
    BullModule.forRoot({
      redis: {
        host: 'localhost',
        port: 6379,
      },
    }),
    BullModule.registerQueue({
      name: 'email',        // Tên queue
    }),
    BullModule.registerQueue({
      name: 'image-process',
    }),
  ],
})
export class AppModule {}
```

### Producer (Thêm job vào queue)

`email/email.service.ts`:

```typescript
import { InjectQueue } from '@nestjs/bull';
import { Queue } from 'bull';

@Injectable()
export class EmailService {
  constructor(@InjectQueue('email') private emailQueue: Queue) {}

  async sendWelcomeEmail(to: string, name: string) {
    await this.emailQueue.add(
      'welcome',              // Tên job
      { to, name },          // Dữ liệu job
      {
        attempts: 3,          // Thử lại tối đa 3 lần nếu thất bại
        backoff: { type: 'exponential', delay: 5000 },
        removeOnComplete: true,
      },
    );
  }

  async scheduleReminder(to: string, delay: number) {
    await this.emailQueue.add(
      'reminder',
      { to },
      { delay }, // Trì hoãn gửi
    );
  }
}
```

### Consumer (Xử lý job)

`email/email.processor.ts`:

```typescript
import { Processor, Process, OnQueueFailed, OnQueueCompleted } from '@nestjs/bull';
import { Job } from 'bull';
import { Logger } from '@nestjs/common';

@Processor('email')
export class EmailProcessor {
  private readonly logger = new Logger(EmailProcessor.name);

  @Process('welcome')
  async sendWelcome(job: Job<{ to: string; name: string }>) {
    const { to, name } = job.data;
    this.logger.log(`Đang gửi email chào mừng tới ${to}`);

    // Thực hiện gửi email thực tế ở đây
    // await nodemailer.sendMail({ to, subject: '...', html: '...' });

    return { sent: true, to };
  }

  @Process('reminder')
  async sendReminder(job: Job) {
    this.logger.log(`Gửi reminder tới ${job.data.to}`);
  }

  @OnQueueCompleted()
  onCompleted(job: Job, result: any) {
    this.logger.log(`Job #${job.id} hoàn thành: ${JSON.stringify(result)}`);
  }

  @OnQueueFailed()
  onFailed(job: Job, error: Error) {
    this.logger.error(`Job #${job.id} thất bại: ${error.message}`);
  }
}
```

---

## 14. Logging nâng cao với Winston

### Cài đặt

```bash
npm install winston nest-winston
```

### Cấu hình

`app.module.ts`:

```typescript
import { WinstonModule } from 'nest-winston';
import * as winston from 'winston';

@Module({
  imports: [
    WinstonModule.forRoot({
      transports: [
        // Log ra console với màu sắc
        new winston.transports.Console({
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.colorize(),
            winston.format.printf(({ timestamp, level, message, context }) => {
              return `${timestamp} [${context}] ${level}: ${message}`;
            }),
          ),
        }),
        // Log ra file (chỉ error)
        new winston.transports.File({
          filename: 'logs/error.log',
          level: 'error',
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.json(),
          ),
        }),
        // Log tất cả ra file
        new winston.transports.File({
          filename: 'logs/combined.log',
          format: winston.format.combine(
            winston.format.timestamp(),
            winston.format.json(),
          ),
        }),
      ],
    }),
  ],
})
export class AppModule {}
```

### Sử dụng Logger

```typescript
import { Logger } from '@nestjs/common';

@Injectable()
export class UserService {
  private readonly logger = new Logger(UserService.name);

  async findOne(id: number) {
    this.logger.log(`Tìm user với id: ${id}`);

    try {
      const user = await this.userRepository.findOne({ where: { id } });
      if (!user) {
        this.logger.warn(`User ${id} không tồn tại`);
        throw new NotFoundException();
      }

      this.logger.debug(`Đã tìm thấy user: ${user.email}`);
      return user;
    } catch (error) {
      this.logger.error(`Lỗi khi tìm user ${id}`, error.stack);
      throw error;
    }
  }
}
```

---

## Tóm tắt công nghệ & thư viện

| Tính năng | Thư viện | Lệnh cài |
|-----------|---------|----------|
| Authentication | `@nestjs/jwt`, `passport` | `npm i @nestjs/jwt @nestjs/passport` |
| Configuration | `@nestjs/config` | `npm i @nestjs/config` |
| Caching | `@nestjs/cache-manager` | `npm i @nestjs/cache-manager cache-manager` |
| WebSockets | `@nestjs/websockets`, `socket.io` | `npm i @nestjs/websockets socket.io` |
| Cron Jobs | `@nestjs/schedule` | `npm i @nestjs/schedule` |
| File Upload | `multer` | `npm i multer` |
| Swagger | `@nestjs/swagger` | `npm i @nestjs/swagger` |
| Rate Limiting | `@nestjs/throttler` | `npm i @nestjs/throttler` |
| Health Check | `@nestjs/terminus` | `npm i @nestjs/terminus` |
| Events | `@nestjs/event-emitter` | `npm i @nestjs/event-emitter` |
| Queue | `@nestjs/bull`, `bull` | `npm i @nestjs/bull bull` |
| Logging | `winston`, `nest-winston` | `npm i winston nest-winston` |

---

*Phần 2 — NestJS Nâng Cao | Phiên bản: v10.x*
