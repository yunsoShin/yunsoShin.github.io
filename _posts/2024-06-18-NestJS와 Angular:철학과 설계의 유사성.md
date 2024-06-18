---
layout: post
title: "NestJS와 Angular: 철학과 설계의 유사성"
date: 2024-06-18
categories: [NestJS, Angular]
tags: [web framework, typescript]
---

NestJS는 Node.js를 위한 진보된 웹 프레임워크로, Angular의 철학과 디자인 패턴에 깊은 영향을 받아 개발되었습니다. 이는 두 프레임워크가 구조화된 모듈 시스템, 의존성 주입, 데코레이터 사용 등에서 많은 유사성을 가지게 합니다. 아래에서는 NestJS와 Angular의 주요 개념을 비교하면서, 코드 예제를 통해 구체적으로 설명해 보겠습니다.

## 목차

1. [모듈 시스템](#모듈-시스템)
2. [의존성 주입 (Dependency Injection)](#의존성-주입-dependency-injection)
3. [데코레이터 사용](#데코레이터-사용)
4. [파이프 (Pipes)](#파이프-pipes)
5. [가드 (Guards)](#가드-guards)
6. [인터셉터 (Interceptors)](#인터셉터-interceptors)
7. [미들웨어 (Middleware)](#미들웨어-middleware)
8. [라우팅 (Routing)](#라우팅-routing)
9. [테스트 (Testing)](#테스트-testing)
10. [커스텀 데코레이터 (Custom Decorators)](#커스텀-데코레이터-custom-decorators)
11. [라이프사이클 훅 (Lifecycle Hooks)](#라이프사이클-훅-lifecycle-hooks)
12. [결론](#결론)

## 모듈 시스템

### Angular

Angular에서는 모듈이 `@NgModule` 데코레이터로 정의됩니다. 모듈은 컴포넌트, 서비스, 다른 모듈 등을 그룹화하여 애플리케이션을 구조화합니다.

```typescript
// Angular의 AppModule
import { NgModule } from "@angular/core";
import { BrowserModule } from "@angular/platform-browser";
import { AppComponent } from "./app.component";

@NgModule({
  declarations: [AppComponent],
  imports: [BrowserModule],
  providers: [],
  bootstrap: [AppComponent]
})
export class AppModule {}
```

### NestJS

NestJS에서도 모듈이 `@Module` 데코레이터로 정의되며, 컨트롤러, 서비스, 다른 모듈 등을 포함합니다.

```typescript
// NestJS의 AppModule
import { Module } from "@nestjs/common";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";

@Module({
  imports: [],
  controllers: [AppController],
  providers: [AppService]
})
export class AppModule {}
```

## 의존성 주입 (Dependency Injection)

### Angular

Angular는 의존성 주입을 통해 컴포넌트나 서비스에 필요한 의존성을 주입합니다. `@Injectable` 데코레이터를 사용하여 서비스를 정의합니다.

```typescript
// Angular의 Service
import { Injectable } from "@angular/core";

@Injectable({
  providedIn: "root"
})
export class DataService {
  constructor() {}
  getData() {
    return "Hello Angular";
  }
}

// Angular의 Component
import { Component } from "@angular/core";
import { DataService } from "./data.service";

@Component({
  selector: "app-root",
  template: `{{ data }}`
})
export class AppComponent {
  data: string;
  constructor(private dataService: DataService) {
    this.data = this.dataService.getData();
  }
}
```

### NestJS

NestJS는 Angular와 유사하게 `@Injectable` 데코레이터를 사용하여 서비스를 정의하고, 의존성을 주입합니다.

```typescript
// NestJS의 Service
import { Injectable } from "@nestjs/common";

@Injectable()
export class AppService {
  getHello(): string {
    return "Hello NestJS";
  }
}

// NestJS의 Controller
import { Controller, Get } from "@nestjs/common";
import { AppService } from "./app.service";

@Controller()
export class AppController {
  constructor(private readonly appService: AppService) {}

  @Get()
  getHello(): string {
    return this.appService.getHello();
  }
}
```

## 데코레이터 사용

### Angular

Angular는 데코레이터를 사용하여 컴포넌트, 디렉티브, 파이프, 서비스 등을 정의합니다.

```typescript
// Angular의 Component 데코레이터
import { Component } from "@angular/core";

@Component({
  selector: "app-root",
  templateUrl: "./app.component.html",
  styleUrls: ["./app.component.css"]
})
export class AppComponent {
  title = "app";
}
```

### NestJS

NestJS는 데코레이터를 사용하여 컨트롤러, 라우트 핸들러, 모듈 등을 정의합니다.

```typescript
// NestJS의 Controller 데코레이터
import { Controller, Get } from "@nestjs/common";

@Controller("cats")
export class CatsController {
  @Get()
  findAll(): string {
    return "This action returns all cats";
  }
}
```

## 파이프 (Pipes)

### Angular

Angular에서 파이프는 데이터를 변환하는 데 사용됩니다. 예를 들어, 날짜 형식을 변환하거나 텍스트를 소문자로 바꾸는 데 사용됩니다.

```typescript
// Angular의 파이프
import { Pipe, PipeTransform } from "@angular/core";

@Pipe({
  name: "capitalize"
})
export class CapitalizePipe implements PipeTransform {
  transform(value: string): string {
    return value.charAt(0).toUpperCase() + value.slice(1);
  }
}
```

### NestJS

NestJS에서도 파이프는 유효성 검사와 변환을 위해 사용됩니다. 예를 들어, 요청 데이터를 검증하거나 변환하는 데 사용됩니다.

```typescript
// NestJS의 파이프
import {
  PipeTransform,
  Injectable,
  ArgumentMetadata,
  BadRequestException
} from "@nestjs/common";

@Injectable()
export class ParseIntPipe implements PipeTransform<string, number> {
  transform(value: string, metadata: ArgumentMetadata): number {
    const val = parseInt(value, 10);
    if (isNaN(val)) {
      throw new BadRequestException("Validation failed");
    }
    return val;
  }
}
```

## 가드 (Guards)

### Angular

Angular에서는 라우트 가드를 사용하여 특정 경로에 접근하기 전에 사용자의 인증 상태를 확인할 수 있습니다.

```typescript
// Angular의 AuthGuard
import { Injectable } from "@angular/core";
import { CanActivate, Router } from "@angular/router";
import { AuthService } from "./auth.service";

@Injectable({
  providedIn: "root"
})
export class AuthGuard implements CanActivate {
  constructor(private authService: AuthService, private router: Router) {}

  canActivate(): boolean {
    if (this.authService.isLoggedIn()) {
      return true;
    } else {
      this.router.navigate(["login"]);
      return false;
    }
  }
}
```

### NestJS

NestJS에서도 가드는 요청이 처리되기 전에 특정 조건을 확인하는 데 사용됩니다. 주로 인증 및 권한 부여를 위해 사용됩니다.

```typescript
// NestJS의 AuthGuard
import { Injectable, CanActivate, ExecutionContext } from "@nestjs/common";
import { Observable } from "rxjs";

@Injectable()
export class AuthGuard implements CanActivate {
  canActivate(
    context: ExecutionContext
  ): boolean | Promise<boolean> | Observable<boolean> {
    const request = context.switchToHttp().getRequest();
    return validateRequest(request);
  }
}

function validateRequest(request: any): boolean {
  // 인증 로직을 여기에 추가
  return true;
}
```

## 인터셉터 (Interceptors)

### Angular

Angular에서 HTTP 인터셉터는 모든 HTTP 요청 및 응답을 가로채서 처리할 수 있습니다. 예를 들어, 공통 헤더를 추가하거나 오류를 처리하는 데 사용됩니다.

```typescript
// Angular의 HTTP 인터셉터
import { Injectable } from "@angular/core";
import {
  HttpEvent,
  HttpInterceptor,
  HttpHandler,
  HttpRequest
} from "@angular/common/http";
import { Observable } from "rxjs";

@Injectable()
export class AuthInterceptor implements HttpInterceptor {
  intercept(
    req: HttpRequest<any>,
    next: HttpHandler
  ): Observable<HttpEvent<any>> {
    const authToken = "my-auth-token";
    const authReq = req.clone({
      headers: req.headers.set("Authorization", `Bearer ${authToken}`)
    });
    return next.handle(authReq);
  }
}
```

### NestJS

NestJS에서 인터셉터는 요청 및 응답을 가로채서 처리할 수 있습니다. 예를 들어, 로깅, 캐싱, 응답 데이터 변환 등을 처리할 수 있습니다.

```typescript
// NestJS의 인터셉터
import {
  Injectable,
  NestInterceptor,
  ExecutionContext,
  CallHandler
} from "@nestjs/common";
import { Observable } from "rxjs";
import { map } from "rxjs/operators";

@Injectable()
export class TransformInterceptor implements NestInterceptor {
  intercept(context: ExecutionContext, next: CallHandler): Observable<any> {
    return next.handle().pipe(map((data) => ({ data })));
  }
}
```

## 미들웨어 (Middleware)

### Angular

Angular는 미들웨어 개념이 없지만, 서비스나 다른 기법을 사용하여 비슷한 기능을 구현할 수 있습니다.

### NestJS

NestJS에서는 미들웨어를 사용하여 요청 처리 전에 특정 작업을 수행할 수 있습니다.

```typescript
// NestJS의 미들웨어
import { Injectable, NestMiddleware } from "@nestjs/common";
import { Request, Response, NextFunction } from "express";

@Injectable()
export class LoggerMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    console.log(`Request...`);
    next();
  }
}
```

## 라우팅 (Routing)

### Angular

Angular는 `@angular/router` 모듈을 통해 클라이언트 사이드 라우팅을 지원합니다. 이를 통해 URL 경로에 따라 컴포넌트를 로드할 수 있습니다.

```typescript
// Angular의 라우팅 설정
import { NgModule } from "@angular/core";
import { RouterModule, Routes } from "@angular/router";
import { HomeComponent } from "./home/home.component";
import { AboutComponent } from "./about/about.component";

const routes: Routes = [
  { path: "", component: HomeComponent },
  { path: "about", component: AboutComponent }
];

@NgModule({
  imports: [RouterModule.forRoot(routes)],
  exports: [RouterModule]
})
export class AppRoutingModule {}
```

### NestJS

NestJS는 `@nestjs/common` 모듈을 통해 서버 사이드 라우팅을 지원합니다. 컨트롤러를 통해 URL 경로를 정의하고 핸들러를 지정할 수 있습니다.

```typescript
// NestJS의 라우팅 설정
import { Controller, Get } from "@nestjs/common";

@Controller("home")
export class HomeController {
  @Get()
  getHome(): string {
    return "Welcome to Home";
  }
}

@Controller("about")
export class AboutController {
  @Get()
  getAbout(): string {
    return "About Us";
  }
}
```

## 테스트 (Testing)

### Angular

Angular는 Jasmine과 Karma를 통해 테스트 프레임워크를 제공합니다. 이를 통해 컴포넌트, 서비스 등을 단위 테스트할 수 있습니다.

```typescript
// Angular의 테스트 예제
import { TestBed } from "@angular/core/testing";
import { AppComponent } from "./app.component";

describe("AppComponent", () => {
  beforeEach(async () => {
    await TestBed.configureTestingModule({
      declarations: [AppComponent]
    }).compileComponents();
  });

  it("should create the app", () => {
    const fixture = TestBed.createComponent(AppComponent);
    const app = fixture.componentInstance;
    expect(app).toBeTruthy();
  });
});
```

### NestJS

NestJS는 Jest를 통해 테스트 프레임워크를 제공합니다. 이를 통해 컨트롤러, 서비스 등을 단위 테스트할 수 있습니다.

```typescript
// NestJS의 테스트 예제
import { Test, TestingModule } from "@nestjs/testing";
import { AppController } from "./app.controller";
import { AppService } from "./app.service";

describe("AppController", () => {
  let appController: AppController;

  beforeEach(async () => {
    const app: TestingModule = await Test.createTestingModule({
      controllers: [AppController],
      providers: [AppService]
    }).compile();

    appController = app.get<AppController>(AppController);
  });

  it('should return "Hello World!"', () => {
    expect(appController.getHello()).toBe("Hello World!");
  });
});
```

## 커스텀 데코레이터 (Custom Decorators)

### Angular

Angular에서는 커스텀 데코레이터를 사용하여 클래스, 메서드, 속성에 메타데이터를 추가할 수 있습니다.

```typescript
// Angular의 커스텀 데코레이터
function Log(target: any, propertyKey: string, descriptor: PropertyDescriptor) {
  const originalMethod = descriptor.value;
  descriptor.value = function (...args: any[]) {
    console.log(
      `Method ${propertyKey} called with args: ${JSON.stringify(args)}`
    );
    return originalMethod.apply(this, args);
  };
  return descriptor;
}

class ExampleService {
  @Log
  exampleMethod(arg1: string, arg2: number) {
    return `${arg1} - ${arg2}`;
  }
}
```

### NestJS

NestJS에서도 커스텀 데코레이터를 사용하여 라우트 핸들러에 메타데이터를 추가할 수 있습니다.

```typescript
// NestJS의 커스텀 데코레이터
import { SetMetadata } from "@nestjs/common";

export const Roles = (...roles: string[]) => SetMetadata("roles", roles);

import { Controller, Get } from "@nestjs/common";
import { Roles } from "./roles.decorator";

@Controller("users")
export class UserController {
  @Get()
  @Roles("admin")
  findAll() {
    return "This route is restricted to admin roles";
  }
}
```

## 라이프사이클 훅 (Lifecycle Hooks)

### Angular

Angular는 컴포넌트의 생명주기 동안 특정 시점에 호출되는 여러 라이프사이클 훅을 제공합니다.

```typescript
// Angular의 라이프사이클 훅
import { Component, OnInit, OnDestroy } from "@angular/core";

@Component({
  selector: "app-example",
  template: `<p>Example component</p>`
})
export class ExampleComponent implements OnInit, OnDestroy {
  ngOnInit() {
    console.log("Component initialized");
  }

  ngOnDestroy() {
    console.log("Component destroyed");
  }
}
```

### NestJS

NestJS는 모듈, 컨트롤러, 프로바이더 등의 생명주기 동안 특정 시점에 호출되는 여러 라이프사이클 훅을 제공합니다.

```typescript
// NestJS의 라이프사이클 훅
import { Injectable, OnModuleInit, OnModuleDestroy } from "@nestjs/common";

@Injectable()
export class ExampleService implements OnModuleInit, OnModuleDestroy {
  onModuleInit() {
    console.log("Module initialized");
  }

  onModuleDestroy() {
    console.log("Module destroyed");
  }
}
```

## 결론

NestJS는 Angular의 디자인 패턴을 채택하여, 엔터프라이즈 애플리케이션을 위한 견고한 아키텍처를 제공합니다. 모듈 시스템, 의존성 주입, 데코레이터의 사용 등에서 Angular와 많은 유사성을 보여줍니다. 이를 통해 개발자들은 Angular와 유사한 방식으로 NestJS 애플리케이션을 구조화하고, 쉽게 유지 보수할 수 있습니다.

## 추가 설명

### 환경

**Angular**: 클라이언트 사이드 프레임워크로, 브라우저에서 동작하는 애플리케이션을 만듭니다. 주로 컴포넌트 기반으로 UI를 구축합니다.

**NestJS**: 서버 사이드 프레임워크로, Node.js 환경에서 동작하는 백엔드 애플리케이션을 만듭니다. 주로 컨트롤러, 서비스 기반으로 비즈니스 로직을 구축합니다.

### 의존성 등록

**Angular**: 서비스는 `providedIn` 속성을 통해 모듈, 컴포넌트, 루트 등 여러 레벨에 등록될 수 있습니다.

```typescript
@Injectable({
  providedIn: "root"
})
export class DataService {}
```

**NestJS**: 서비스는 `@Module` 데코레이터의 `providers` 배열에 명시적으로 등록됩니다.

```typescript
@Module({
  providers: [AppService]
})
export class AppModule {}
```

### 사용 방식

**Angular**: 컴포넌트에서 서비스 주입.

```typescript
constructor(private dataService: DataService) {}
```

**NestJS**: 컨트롤러나 다른 서비스에서 서비스 주입.

```typescript
constructor(private readonly appService: AppService) {}
```
