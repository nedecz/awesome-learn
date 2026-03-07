# Node.js and Backend TypeScript

## Table of Contents

1. [Overview](#overview)
   - [Target Audience](#target-audience)
   - [Scope](#scope)
2. [Setting Up Node.js with TypeScript](#setting-up-nodejs-with-typescript)
   - [Project Initialization](#project-initialization)
   - [tsconfig for Node.js](#tsconfig-for-nodejs)
   - [Running TypeScript with ts-node and tsx](#running-typescript-with-ts-node-and-tsx)
3. [Express.js with TypeScript](#expressjs-with-typescript)
   - [Typed Routes](#typed-routes)
   - [Middleware Typing](#middleware-typing)
   - [Request and Response Types](#request-and-response-types)
4. [NestJS Framework](#nestjs-framework)
   - [Modules](#modules)
   - [Controllers](#controllers)
   - [Services and Dependency Injection](#services-and-dependency-injection)
   - [Decorators](#decorators)
5. [Fastify with TypeScript](#fastify-with-typescript)
   - [Schema-Based Validation](#schema-based-validation)
   - [Typed Plugins](#typed-plugins)
6. [Database Integration](#database-integration)
   - [TypeORM](#typeorm)
   - [Prisma](#prisma)
   - [Drizzle ORM](#drizzle-orm)
7. [API Development](#api-development)
   - [REST API Patterns](#rest-api-patterns)
   - [GraphQL with TypeScript](#graphql-with-typescript)
8. [Authentication and Middleware Patterns](#authentication-and-middleware-patterns)
   - [JWT Authentication](#jwt-authentication)
   - [Role-Based Access Control](#role-based-access-control)
9. [Error Handling](#error-handling)
   - [Custom Error Classes](#custom-error-classes)
   - [Global Error Middleware](#global-error-middleware)
10. [Environment Configuration](#environment-configuration)
    - [Type-Safe Environment Variables](#type-safe-environment-variables)
11. [Deployment and Build Considerations](#deployment-and-build-considerations)
    - [Build Pipeline](#build-pipeline)
    - [Docker for Node.js TypeScript Apps](#docker-for-nodejs-typescript-apps)
12. [Next Steps](#next-steps)
13. [Version History](#version-history)

---

## Overview

Node.js combined with TypeScript provides a **type-safe, scalable foundation** for building backend applications. TypeScript catches entire categories of runtime errors at compile time, making server-side code more reliable and maintainable. This guide covers the essential tools, frameworks, and patterns for professional backend TypeScript development.

### Target Audience

- **Backend developers adopting TypeScript** — Developers with Node.js experience looking to add type safety to their server-side code
- **Full-stack TypeScript developers** — Engineers who want a unified language across frontend and backend
- **Teams building production APIs** — Teams that need reliable, well-typed API layers with frameworks like Express, NestJS, or Fastify

### Scope

- Project setup and configuration for Node.js with TypeScript
- Express.js routing, middleware, and request/response typing
- NestJS architecture including modules, controllers, services, and decorators
- Fastify with schema-based type inference
- Database integration with TypeORM, Prisma, and Drizzle
- REST and GraphQL API development patterns
- Authentication, error handling, and environment configuration
- Deployment and containerization strategies

---

## Setting Up Node.js with TypeScript

### Project Initialization

Every Node.js TypeScript project starts with proper initialization. Install TypeScript and the Node.js type definitions as dev dependencies.

```typescript
// package.json (relevant scripts section)
{
  "name": "my-node-api",
  "scripts": {
    "build": "tsc",
    "start": "node dist/index.js",
    "dev": "tsx watch src/index.ts",
    "lint": "eslint src/"
  },
  "dependencies": {
    "express": "^4.18.0"
  },
  "devDependencies": {
    "@types/express": "^4.17.0",
    "@types/node": "^20.0.0",
    "tsx": "^4.0.0",
    "typescript": "^5.4.0"
  }
}
```

### tsconfig for Node.js

The `tsconfig.json` for a Node.js backend differs from a frontend config. Target a modern ECMAScript version and use `NodeNext` module resolution.

```typescript
// tsconfig.json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "NodeNext",
    "moduleResolution": "NodeNext",
    "lib": ["ES2022"],
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "resolveJsonModule": true,
    "declaration": true,
    "declarationMap": true,
    "sourceMap": true
  },
  "include": ["src/**/*"],
  "exclude": ["node_modules", "dist", "**/*.test.ts"]
}
```

### Running TypeScript with ts-node and tsx

During development you want to run TypeScript directly without a manual compile step. Two popular tools handle this — **ts-node** for traditional setups and **tsx** for a faster, zero-config experience.

```typescript
// src/index.ts — basic Node.js server entry point
import http from "node:http";

const PORT = process.env.PORT ?? 3000;

const server = http.createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "application/json" });
  res.end(JSON.stringify({ message: "Hello from TypeScript!" }));
});

server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

Run during development with:

- **tsx** — `npx tsx watch src/index.ts` (fast, uses esbuild under the hood)
- **ts-node** — `npx ts-node src/index.ts` (uses the TypeScript compiler)

---

## Express.js with TypeScript

Express remains the most widely used Node.js web framework. TypeScript adds type safety to routes, middleware, and request handling.

### Typed Routes

Define typed route handlers that enforce the shape of request parameters, query strings, and response bodies.

```typescript
import express, { Router } from "express";

interface User {
  id: string;
  name: string;
  email: string;
  role: "admin" | "user";
}

// In-memory store for demonstration
const users: Map<string, User> = new Map();

const router = Router();

// GET /users/:id — typed params
router.get<{ id: string }, User | { error: string }>(
  "/users/:id",
  (req, res) => {
    const user = users.get(req.params.id);
    if (!user) {
      res.status(404).json({ error: "User not found" });
      return;
    }
    res.json(user);
  }
);

// POST /users — typed request body
router.post<Record<string, never>, User, Omit<User, "id">>(
  "/users",
  (req, res) => {
    const id = crypto.randomUUID();
    const user: User = { id, ...req.body };
    users.set(id, user);
    res.status(201).json(user);
  }
);

export default router;
```

### Middleware Typing

Custom middleware benefits greatly from explicit typing. Define middleware that attaches properties to the request object using **declaration merging**.

```typescript
import { Request, Response, NextFunction } from "express";

// Extend the Express Request interface via declaration merging
declare global {
  namespace Express {
    interface Request {
      userId?: string;
      correlationId: string;
    }
  }
}

// Correlation ID middleware — attaches a unique ID to every request
function correlationMiddleware(
  req: Request,
  _res: Response,
  next: NextFunction
): void {
  req.correlationId =
    (req.headers["x-correlation-id"] as string) ?? crypto.randomUUID();
  next();
}

// Logging middleware with typed request
function requestLogger(
  req: Request,
  _res: Response,
  next: NextFunction
): void {
  console.log(
    `[${req.correlationId}] ${req.method} ${req.path}`
  );
  next();
}

export { correlationMiddleware, requestLogger };
```

### Request and Response Types

Express provides generic type parameters for `Request` and `Response`. Use them to enforce contracts at the handler level.

```typescript
import { Request, Response } from "express";

// Define the shape of each part of the request
interface CreateOrderParams {
  customerId: string;
}

interface CreateOrderBody {
  items: Array<{ productId: string; quantity: number }>;
  shippingAddress: string;
}

interface CreateOrderQuery {
  coupon?: string;
}

interface OrderResponse {
  orderId: string;
  total: number;
  status: "pending" | "confirmed";
}

// Fully typed handler
function createOrder(
  req: Request<CreateOrderParams, OrderResponse, CreateOrderBody, CreateOrderQuery>,
  res: Response<OrderResponse>
): void {
  const { customerId } = req.params;
  const { items, shippingAddress } = req.body;
  const { coupon } = req.query;

  // All variables above are correctly typed
  const order: OrderResponse = {
    orderId: crypto.randomUUID(),
    total: items.reduce((sum, item) => sum + item.quantity * 10, 0),
    status: "pending",
  };

  res.status(201).json(order);
}

export { createOrder };
```

---

## NestJS Framework

NestJS is an opinionated, enterprise-grade framework built on TypeScript. It uses **decorators**, **dependency injection**, and a modular architecture inspired by Angular.

### Modules

Every NestJS application is organized into modules. A module groups related controllers, services, and providers.

```typescript
import { Module } from "@nestjs/common";
import { UsersController } from "./users.controller";
import { UsersService } from "./users.service";
import { TypeOrmModule } from "@nestjs/typeorm";
import { User } from "./user.entity";

@Module({
  imports: [TypeOrmModule.forFeature([User])],
  controllers: [UsersController],
  providers: [UsersService],
  exports: [UsersService],
})
export class UsersModule {}
```

### Controllers

Controllers handle incoming requests and return responses. Decorators define routes and HTTP methods.

```typescript
import {
  Controller,
  Get,
  Post,
  Put,
  Delete,
  Param,
  Body,
  HttpCode,
  HttpStatus,
  ParseUUIDPipe,
  NotFoundException,
} from "@nestjs/common";
import { UsersService } from "./users.service";
import { CreateUserDto, UpdateUserDto, UserResponseDto } from "./users.dto";

@Controller("users")
export class UsersController {
  constructor(private readonly usersService: UsersService) {}

  @Get()
  async findAll(): Promise<UserResponseDto[]> {
    return this.usersService.findAll();
  }

  @Get(":id")
  async findOne(
    @Param("id", ParseUUIDPipe) id: string
  ): Promise<UserResponseDto> {
    const user = await this.usersService.findOne(id);
    if (!user) {
      throw new NotFoundException(`User with ID ${id} not found`);
    }
    return user;
  }

  @Post()
  @HttpCode(HttpStatus.CREATED)
  async create(@Body() createUserDto: CreateUserDto): Promise<UserResponseDto> {
    return this.usersService.create(createUserDto);
  }

  @Put(":id")
  async update(
    @Param("id", ParseUUIDPipe) id: string,
    @Body() updateUserDto: UpdateUserDto
  ): Promise<UserResponseDto> {
    return this.usersService.update(id, updateUserDto);
  }

  @Delete(":id")
  @HttpCode(HttpStatus.NO_CONTENT)
  async remove(@Param("id", ParseUUIDPipe) id: string): Promise<void> {
    await this.usersService.remove(id);
  }
}
```

### Services and Dependency Injection

Services contain business logic and are injected into controllers via the constructor. NestJS manages the lifecycle automatically.

```typescript
import { Injectable, NotFoundException } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { User } from "./user.entity";
import { CreateUserDto, UpdateUserDto, UserResponseDto } from "./users.dto";

@Injectable()
export class UsersService {
  constructor(
    @InjectRepository(User)
    private readonly userRepository: Repository<User>
  ) {}

  async findAll(): Promise<UserResponseDto[]> {
    const users = await this.userRepository.find();
    return users.map((user) => this.toResponse(user));
  }

  async findOne(id: string): Promise<UserResponseDto | null> {
    const user = await this.userRepository.findOneBy({ id });
    return user ? this.toResponse(user) : null;
  }

  async create(dto: CreateUserDto): Promise<UserResponseDto> {
    const user = this.userRepository.create(dto);
    const saved = await this.userRepository.save(user);
    return this.toResponse(saved);
  }

  async update(id: string, dto: UpdateUserDto): Promise<UserResponseDto> {
    const user = await this.userRepository.findOneBy({ id });
    if (!user) {
      throw new NotFoundException(`User ${id} not found`);
    }
    Object.assign(user, dto);
    const updated = await this.userRepository.save(user);
    return this.toResponse(updated);
  }

  async remove(id: string): Promise<void> {
    const result = await this.userRepository.delete(id);
    if (result.affected === 0) {
      throw new NotFoundException(`User ${id} not found`);
    }
  }

  private toResponse(user: User): UserResponseDto {
    return {
      id: user.id,
      name: user.name,
      email: user.email,
      role: user.role,
      createdAt: user.createdAt,
    };
  }
}
```

### Decorators

NestJS relies heavily on decorators for metadata. You can create **custom decorators** for cross-cutting concerns.

```typescript
import {
  createParamDecorator,
  ExecutionContext,
  SetMetadata,
} from "@nestjs/common";

// Custom decorator to extract the current user from the request
export const CurrentUser = createParamDecorator(
  (data: string | undefined, ctx: ExecutionContext) => {
    const request = ctx.switchToHttp().getRequest();
    const user = request.user;
    return data ? user?.[data] : user;
  }
);

// Role-based access decorator
export const ROLES_KEY = "roles";
export const Roles = (...roles: string[]) => SetMetadata(ROLES_KEY, roles);

// Usage in a controller:
// @Roles("admin")
// @Get("admin/dashboard")
// async dashboard(@CurrentUser() user: UserPayload) {
//   return { message: `Welcome, ${user.name}` };
// }
```

---

## Fastify with TypeScript

Fastify is a high-performance alternative to Express with **first-class TypeScript support** and built-in schema validation.

### Schema-Based Validation

Fastify uses JSON Schema for request/response validation and **automatically infers TypeScript types** from schemas.

```typescript
import Fastify, { FastifyInstance } from "fastify";

const app: FastifyInstance = Fastify({ logger: true });

// Define schema — Fastify infers types automatically
const getUserSchema = {
  params: {
    type: "object" as const,
    properties: {
      id: { type: "string" as const },
    },
    required: ["id"] as const,
  },
  response: {
    200: {
      type: "object" as const,
      properties: {
        id: { type: "string" as const },
        name: { type: "string" as const },
        email: { type: "string" as const },
      },
    },
  },
};

app.get(
  "/users/:id",
  { schema: getUserSchema },
  async (request, reply) => {
    const { id } = request.params as { id: string };
    return { id, name: "Alice", email: "alice@example.com" };
  }
);

app.listen({ port: 3000 });
```

### Typed Plugins

Fastify's plugin architecture is fully typed. Decorate the Fastify instance and the types propagate everywhere.

```typescript
import fp from "fastify-plugin";
import { FastifyInstance, FastifyPluginAsync } from "fastify";

// Extend Fastify's type system with custom decorations
declare module "fastify" {
  interface FastifyInstance {
    config: {
      databaseUrl: string;
      jwtSecret: string;
    };
  }
}

const configPlugin: FastifyPluginAsync = async (app: FastifyInstance) => {
  const config = {
    databaseUrl: process.env.DATABASE_URL ?? "postgres://localhost:5432/mydb",
    jwtSecret: process.env.JWT_SECRET ?? "dev-secret",
  };
  app.decorate("config", config);
};

export default fp(configPlugin, { name: "config" });
```

---

## Database Integration

Type-safe database access is one of the biggest advantages of server-side TypeScript. Three popular ORMs each take a different approach.

### TypeORM

TypeORM uses **decorators** to define entities and supports both the Active Record and Data Mapper patterns.

```typescript
import {
  Entity,
  PrimaryGeneratedColumn,
  Column,
  CreateDateColumn,
  UpdateDateColumn,
  ManyToOne,
  OneToMany,
} from "typeorm";

@Entity("users")
export class User {
  @PrimaryGeneratedColumn("uuid")
  id!: string;

  @Column({ length: 100 })
  name!: string;

  @Column({ unique: true })
  email!: string;

  @Column({ type: "enum", enum: ["admin", "user"], default: "user" })
  role!: "admin" | "user";

  @OneToMany(() => Post, (post) => post.author)
  posts!: Post[];

  @CreateDateColumn()
  createdAt!: Date;

  @UpdateDateColumn()
  updatedAt!: Date;
}

@Entity("posts")
export class Post {
  @PrimaryGeneratedColumn("uuid")
  id!: string;

  @Column()
  title!: string;

  @Column("text")
  content!: string;

  @ManyToOne(() => User, (user) => user.posts)
  author!: User;

  @CreateDateColumn()
  createdAt!: Date;
}
```

### Prisma

Prisma generates a fully typed client from a schema file. The generated types stay in sync with your database automatically.

```typescript
// prisma/schema.prisma defines the models, then Prisma generates the client.
// After running `npx prisma generate`, use the client:

import { PrismaClient, User, Post, Prisma } from "@prisma/client";

const prisma = new PrismaClient();

// Create a user — the input type is fully inferred
async function createUser(
  data: Prisma.UserCreateInput
): Promise<User> {
  return prisma.user.create({ data });
}

// Query with relations — return type includes nested posts
async function getUserWithPosts(id: string) {
  return prisma.user.findUnique({
    where: { id },
    include: { posts: true },
  });
}
// Return type: (User & { posts: Post[] }) | null

// Type-safe filtering and pagination
async function searchUsers(params: {
  query?: string;
  role?: "admin" | "user";
  skip?: number;
  take?: number;
}) {
  const { query, role, skip = 0, take = 20 } = params;

  const where: Prisma.UserWhereInput = {
    ...(role && { role }),
    ...(query && {
      OR: [
        { name: { contains: query, mode: "insensitive" } },
        { email: { contains: query, mode: "insensitive" } },
      ],
    }),
  };

  const [users, total] = await Promise.all([
    prisma.user.findMany({ where, skip, take, orderBy: { createdAt: "desc" } }),
    prisma.user.count({ where }),
  ]);

  return { users, total, page: Math.floor(skip / take) + 1 };
}
```

### Drizzle ORM

Drizzle takes a **TypeScript-first** approach — you define your schema in TypeScript and the types flow through queries automatically.

```typescript
import { pgTable, uuid, varchar, text, timestamp, pgEnum } from "drizzle-orm/pg-core";
import { relations } from "drizzle-orm";
import { drizzle } from "drizzle-orm/node-postgres";
import { eq } from "drizzle-orm";
import { Pool } from "pg";

// Define schema in TypeScript
export const roleEnum = pgEnum("role", ["admin", "user"]);

export const users = pgTable("users", {
  id: uuid("id").defaultRandom().primaryKey(),
  name: varchar("name", { length: 100 }).notNull(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  role: roleEnum("role").default("user").notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export const posts = pgTable("posts", {
  id: uuid("id").defaultRandom().primaryKey(),
  title: varchar("title", { length: 255 }).notNull(),
  content: text("content").notNull(),
  authorId: uuid("author_id")
    .references(() => users.id)
    .notNull(),
  createdAt: timestamp("created_at").defaultNow().notNull(),
});

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

// Usage — types are inferred from the schema
const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle(pool);

async function findUserByEmail(email: string) {
  const result = await db.select().from(users).where(eq(users.email, email));
  return result[0] ?? null;
  // Return type: InferSelectModel<typeof users> | null
}
```

---

## API Development

### REST API Patterns

Structure a REST API with clear separation between routing, validation, and business logic.

```typescript
import { Router, Request, Response, NextFunction } from "express";
import { z } from "zod";

// Zod schema for request validation
const CreateProductSchema = z.object({
  name: z.string().min(1).max(200),
  price: z.number().positive(),
  category: z.enum(["electronics", "clothing", "books", "food"]),
  tags: z.array(z.string()).optional(),
});

type CreateProductInput = z.infer<typeof CreateProductSchema>;

// Validation middleware factory
function validate<T>(schema: z.ZodSchema<T>) {
  return (req: Request, _res: Response, next: NextFunction): void => {
    const result = schema.safeParse(req.body);
    if (!result.success) {
      res.status(400).json({
        error: "Validation failed",
        details: result.error.flatten().fieldErrors,
      });
      return;
    }
    req.body = result.data;
    next();
  };
}

const router = Router();

router.post(
  "/products",
  validate(CreateProductSchema),
  async (req: Request, res: Response) => {
    const product: CreateProductInput = req.body;
    // product.name, product.price, product.category are all typed
    res.status(201).json({ id: crypto.randomUUID(), ...product });
  }
);

export default router;
```

### GraphQL with TypeScript

Type-safe GraphQL servers benefit from code-generation tools that keep resolvers in sync with the schema.

```typescript
import { buildSchema } from "type-graphql";
import { Resolver, Query, Mutation, Arg, ObjectType, Field, ID } from "type-graphql";

// Define GraphQL types with decorators
@ObjectType()
class Book {
  @Field(() => ID)
  id!: string;

  @Field()
  title!: string;

  @Field()
  author!: string;

  @Field()
  publishedYear!: number;
}

// Resolver with fully typed arguments and return values
@Resolver(Book)
class BookResolver {
  private books: Book[] = [];

  @Query(() => [Book])
  async books(): Promise<Book[]> {
    return this.books;
  }

  @Query(() => Book, { nullable: true })
  async book(@Arg("id") id: string): Promise<Book | undefined> {
    return this.books.find((b) => b.id === id);
  }

  @Mutation(() => Book)
  async addBook(
    @Arg("title") title: string,
    @Arg("author") author: string,
    @Arg("publishedYear") publishedYear: number
  ): Promise<Book> {
    const book: Book = {
      id: crypto.randomUUID(),
      title,
      author,
      publishedYear,
    };
    this.books.push(book);
    return book;
  }
}

// Build the schema
async function createSchema() {
  return buildSchema({
    resolvers: [BookResolver],
  });
}
```

---

## Authentication and Middleware Patterns

### JWT Authentication

Implement type-safe JWT authentication with a middleware that decodes tokens and attaches the user payload to the request.

```typescript
import { Request, Response, NextFunction } from "express";
import jwt from "jsonwebtoken";

interface JwtPayload {
  userId: string;
  email: string;
  role: "admin" | "user";
  iat: number;
  exp: number;
}

declare global {
  namespace Express {
    interface Request {
      user?: JwtPayload;
    }
  }
}

function authenticateToken(
  req: Request,
  res: Response,
  next: NextFunction
): void {
  const authHeader = req.headers.authorization;
  const token = authHeader?.startsWith("Bearer ")
    ? authHeader.slice(7)
    : undefined;

  if (!token) {
    res.status(401).json({ error: "Authentication required" });
    return;
  }

  try {
    const secret = process.env.JWT_SECRET;
    if (!secret) {
      throw new Error("JWT_SECRET is not configured");
    }
    const payload = jwt.verify(token, secret) as JwtPayload;
    req.user = payload;
    next();
  } catch {
    res.status(403).json({ error: "Invalid or expired token" });
  }
}

// Generate a token
function generateToken(user: { id: string; email: string; role: "admin" | "user" }): string {
  const secret = process.env.JWT_SECRET;
  if (!secret) {
    throw new Error("JWT_SECRET is not configured");
  }
  return jwt.sign(
    { userId: user.id, email: user.email, role: user.role },
    secret,
    { expiresIn: "24h" }
  );
}

export { authenticateToken, generateToken };
```

### Role-Based Access Control

Build a reusable authorization middleware that checks the user's role before granting access.

```typescript
import { Request, Response, NextFunction } from "express";

type Role = "admin" | "user";

// Higher-order function that returns middleware for role checking
function requireRole(...allowedRoles: Role[]) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const user = req.user;

    if (!user) {
      res.status(401).json({ error: "Authentication required" });
      return;
    }

    if (!allowedRoles.includes(user.role)) {
      res.status(403).json({
        error: "Insufficient permissions",
        required: allowedRoles,
        current: user.role,
      });
      return;
    }

    next();
  };
}

// Usage:
// router.delete("/users/:id", authenticateToken, requireRole("admin"), deleteUser);
// router.get("/profile", authenticateToken, requireRole("admin", "user"), getProfile);

export { requireRole };
```

---

## Error Handling

### Custom Error Classes

Define a hierarchy of error classes that carry HTTP status codes and structured error information.

```typescript
// Base application error
export class AppError extends Error {
  constructor(
    message: string,
    public readonly statusCode: number,
    public readonly code: string,
    public readonly details?: Record<string, unknown>
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super(`${resource} with ID ${id} not found`, 404, "NOT_FOUND", {
      resource,
      id,
    });
  }
}

export class ValidationError extends AppError {
  constructor(errors: Record<string, string[]>) {
    super("Validation failed", 400, "VALIDATION_ERROR", { errors });
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = "Authentication required") {
    super(message, 401, "UNAUTHORIZED");
  }
}

export class ForbiddenError extends AppError {
  constructor(message = "Insufficient permissions") {
    super(message, 403, "FORBIDDEN");
  }
}

export class ConflictError extends AppError {
  constructor(message: string) {
    super(message, 409, "CONFLICT");
  }
}
```

### Global Error Middleware

A centralized error handler catches all errors and returns a consistent JSON response.

```typescript
import { Request, Response, NextFunction } from "express";
import { AppError } from "./errors";

interface ErrorResponse {
  error: {
    code: string;
    message: string;
    details?: Record<string, unknown>;
  };
}

function globalErrorHandler(
  err: Error,
  _req: Request,
  res: Response<ErrorResponse>,
  _next: NextFunction
): void {
  // Log the full error for debugging
  console.error(`[ERROR] ${err.name}: ${err.message}`, {
    stack: err.stack,
  });

  if (err instanceof AppError) {
    res.status(err.statusCode).json({
      error: {
        code: err.code,
        message: err.message,
        details: err.details,
      },
    });
    return;
  }

  // Unknown errors — return a generic 500 response
  res.status(500).json({
    error: {
      code: "INTERNAL_SERVER_ERROR",
      message: "An unexpected error occurred",
    },
  });
}

// Async handler wrapper — catches promise rejections and forwards them
function asyncHandler(
  fn: (req: Request, res: Response, next: NextFunction) => Promise<void>
) {
  return (req: Request, res: Response, next: NextFunction): void => {
    fn(req, res, next).catch(next);
  };
}

export { globalErrorHandler, asyncHandler };
```

---

## Environment Configuration

### Type-Safe Environment Variables

Use a validation library to parse and type environment variables at startup. This prevents runtime errors from missing or malformed configuration.

```typescript
import { z } from "zod";

const envSchema = z.object({
  NODE_ENV: z.enum(["development", "production", "test"]).default("development"),
  PORT: z.coerce.number().int().positive().default(3000),
  DATABASE_URL: z.string().url(),
  JWT_SECRET: z.string().min(32),
  REDIS_URL: z.string().url().optional(),
  CORS_ORIGINS: z
    .string()
    .transform((val) => val.split(",").map((s) => s.trim()))
    .default("http://localhost:3000"),
  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
});

// Validate at startup — throws with clear errors if anything is missing
function loadConfig() {
  const result = envSchema.safeParse(process.env);

  if (!result.success) {
    const formatted = result.error.flatten().fieldErrors;
    console.error("❌ Invalid environment variables:", formatted);
    process.exit(1);
  }

  return result.data;
}

// Export a singleton config object
export const config = loadConfig();

// TypeScript now knows every field and its type:
// config.PORT        → number
// config.NODE_ENV    → "development" | "production" | "test"
// config.DATABASE_URL → string
// config.CORS_ORIGINS → string[]
```

---

## Deployment and Build Considerations

### Build Pipeline

A production build compiles TypeScript to JavaScript, strips type information, and produces optimized output.

```typescript
// tsconfig.build.json — extends base config with production overrides
{
  "extends": "./tsconfig.json",
  "compilerOptions": {
    "sourceMap": false,
    "declaration": false,
    "declarationMap": false,
    "removeComments": true
  },
  "exclude": [
    "node_modules",
    "dist",
    "**/*.test.ts",
    "**/*.spec.ts",
    "test/"
  ]
}
```

A typical build script in `package.json`:

```typescript
{
  "scripts": {
    "build": "tsc -p tsconfig.build.json",
    "start": "node dist/index.js",
    "start:prod": "NODE_ENV=production node dist/index.js"
  }
}
```

### Docker for Node.js TypeScript Apps

Use a **multi-stage Docker build** to keep the production image lean — the TypeScript compiler and dev dependencies are discarded after the build stage.

```typescript
// Dockerfile (shown as code for reference)
// Stage 1: Build
// FROM node:20-alpine AS builder
// WORKDIR /app
// COPY package*.json ./
// RUN npm ci
// COPY tsconfig*.json ./
// COPY src/ ./src/
// RUN npm run build
//
// Stage 2: Production
// FROM node:20-alpine
// WORKDIR /app
// COPY package*.json ./
// RUN npm ci --omit=dev
// COPY --from=builder /app/dist ./dist
// EXPOSE 3000
// CMD ["node", "dist/index.js"]
```

Key considerations for production deployments:

- **Health checks** — expose a `/health` endpoint for orchestrators like Kubernetes
- **Graceful shutdown** — handle `SIGTERM` to close database connections and finish in-flight requests
- **Structured logging** — use JSON logs that integrate with log aggregation tools

```typescript
import express from "express";

const app = express();

// Health check endpoint
app.get("/health", (_req, res) => {
  res.json({ status: "healthy", uptime: process.uptime() });
});

const server = app.listen(3000, () => {
  console.log("Server started on port 3000");
});

// Graceful shutdown
function shutdown(signal: string) {
  console.log(`Received ${signal}. Shutting down gracefully...`);
  server.close(() => {
    console.log("HTTP server closed");
    process.exit(0);
  });

  // Force exit after 10 seconds if graceful shutdown stalls
  setTimeout(() => {
    console.error("Forced shutdown after timeout");
    process.exit(1);
  }, 10_000);
}

process.on("SIGTERM", () => shutdown("SIGTERM"));
process.on("SIGINT", () => shutdown("SIGINT"));
```

---

## Next Steps

Continue your TypeScript learning journey:

| File | Topic | Description |
|---|---|---|
| [00-OVERVIEW.md](00-OVERVIEW.md) | TypeScript Fundamentals | Core language features, basic types, and project setup |
| [01-TYPE-SYSTEM.md](01-TYPE-SYSTEM.md) | Type System | Generics, utility types, conditional types, and type inference |
| [02-PATTERNS.md](02-PATTERNS.md) | Design Patterns | Common design patterns implemented in TypeScript |
| [04-FRONTEND-FRAMEWORKS.md](04-FRONTEND-FRAMEWORKS.md) | Frontend Frameworks | React, Angular, and Vue with TypeScript |
| [05-TESTING.md](05-TESTING.md) | Testing | Unit testing, integration testing, and mocking in TypeScript |
| [06-TOOLING.md](06-TOOLING.md) | Tooling | Build tools, linters, and developer experience |
| [07-BEST-PRACTICES.md](07-BEST-PRACTICES.md) | Best Practices | Coding standards and recommended approaches |
| [08-ANTI-PATTERNS.md](08-ANTI-PATTERNS.md) | Anti-Patterns | Common mistakes and how to avoid them |
| [LEARNING-PATH.md](LEARNING-PATH.md) | Learning Path | Guided progression through all TypeScript topics |

---

## Version History

| Version | Date | Changes |
|---------|------|---------|
| 1.0 | 2025 | Initial Node.js and Backend TypeScript documentation |
