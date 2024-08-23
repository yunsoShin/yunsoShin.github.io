### 오늘의 개발 기록: NestJS와 TypeORM을 사용한 MySQL 데이터베이스 연결 및 설정

오늘은 NestJS와 TypeORM을 활용하여 MySQL 데이터베이스에 연결하고, 환경 변수에 따라 설정을 다르게 적용하는 작업을 진행했습니다. 이 과정에서 환경에 따른 유연한 설정과 데이터베이스 관리의 중요성을 다시 한번 느낄 수 있었습니다. 이번 포스팅에서는 오늘 진행한 작업의 주요 내용을 정리해보겠습니다.

#### 1. 환경 변수와 설정 관리

우선, `ConfigModule`과 `ConfigService`를 사용하여 환경 변수를 안전하게 관리하는 구조를 만들었습니다. NestJS의 `ConfigModule`을 글로벌로 설정하여, 모든 모듈에서 환경 변수를 쉽게 접근할 수 있도록 했습니다. 이를 통해 MySQL 데이터베이스 연결 정보를 안전하게 관리할 수 있었습니다.

```typescript
@Module({
  imports: [
    ConfigModule.forRoot({
      isGlobal: true
    }),
    TypeOrmModule.forRootAsync({
      imports: [ConfigModule],
      inject: [ConfigService],
      useFactory: (configService: ConfigService) => ({
        type: "mysql",
        host: configService.get("DB_HOST") as string,
        port: parseInt(configService.get("DB_PORT") || "3306", 10),
        username: configService.get("DB_USER") as string,
        password: configService.get("DB_PASSWORD") as string,
        database: configService.get("DB_NAME") as string,
        synchronize: configService.get("IS_DEV") === "true",
        autoLoadEntities: configService.get("IS_DEV") === "true"
      })
    }),
    PostsModule
  ],
  controllers: [AppController],
  providers: [AppService]
})
export class AppModule implements OnModuleInit {
  private readonly logger = new Logger(AppModule.name);

  constructor(private connection: Connection) {}

  async onModuleInit() {
    try {
      if (this.connection.isConnected) {
        this.logger.log("Database connection established successfully.");
      } else {
        this.logger.error("Database connection failed.");
      }
    } catch (error) {
      this.logger.error(
        "Error while checking database connection.",
        error.stack
      );
    }
  }
}
```

#### 2. `synchronize`와 `autoLoadEntities` 옵션의 차이점

NestJS에서 TypeORM을 사용할 때 중요한 설정 중 하나는 `synchronize`와 `autoLoadEntities` 옵션입니다. 이 두 옵션의 역할을 명확히 이해하는 것이 데이터베이스 관리에서 매우 중요합니다.

- **`synchronize`**: 엔티티(Entity)와 데이터베이스 테이블 간의 동기화를 자동으로 처리합니다. 개발 환경에서는 매우 유용하지만, 프로덕션 환경에서는 예상치 못한 데이터 손실을 방지하기 위해 이 옵션을 `false`로 설정하는 것이 좋습니다.

- **`autoLoadEntities`**: 애플리케이션에서 정의된 모든 엔티티를 자동으로 로드합니다. 이 옵션은 개발 중에 모든 엔티티를 자동으로 로드할 수 있어 편리하지만, 프로덕션 환경에서는 상황에 따라 조절할 필요가 있습니다.

#### 3. 모듈화된 엔티티 관리

이번 작업에서는 각 모듈에서 엔티티를 관리하도록 구조를 설계했습니다. 예를 들어, `PostsModule`은 자신의 엔티티를 직접 관리합니다. 이를 통해 모듈 간의 의존성을 줄이고, 코드의 유지보수성을 높일 수 있었습니다.

```typescript
import { Module } from "@nestjs/common";
import { TypeOrmModule } from "@nestjs/typeorm";
import { PostsService } from "./posts.service";
import { PostsController } from "./posts.controller";
import { Post } from "./entities/post.entity";

@Module({
  imports: [TypeOrmModule.forFeature([Post])],
  controllers: [PostsController],
  providers: [PostsService]
})
export class PostsModule {}
```

#### 4. 데이터베이스 연결 상태 확인 및 로깅

`OnModuleInit` 인터페이스를 구현하여 데이터베이스 연결 상태를 확인하고, 이를 로깅하는 방식도 추가했습니다. 이 부분은 애플리케이션이 시작될 때 데이터베이스 연결이 성공적으로 이루어졌는지 확인하고, 문제가 있을 경우 적절한 조치를 취할 수 있게 해줍니다.

```typescript
async onModuleInit() {
  try {
    if (this.connection.isConnected) {
      this.logger.log('Database connection established successfully.');
    } else {
      this.logger.error('Database connection failed.');
    }
  } catch (error) {
    this.logger.error('Error while checking database connection.', error.stack);
  }
}
```

#### 5. 배운 점 및 개선 사항

이번 작업을 통해 환경에 따라 유연하게 설정을 관리하는 것이 얼마나 중요한지 다시 한번 깨달았습니다. 특히, 프로덕션 환경에서는 자동 동기화를 피하고, 마이그레이션을 통한 스키마 관리를 더욱 강화할 필요가 있습니다. 또한, 로깅과 에러 처리를 통해 시스템의 신뢰성을 높이는 것도 중요하다는 점을 다시 한번 확인했습니다.

앞으로의 작업에서는 데이터베이스 연결 실패 시 재시도 로직을 추가하거나, 환경 변수의 관리 방식을 더욱 효율적으로 개선해 나갈수도 있을것같습니다.

다음계획으로는 연결한 데이터베이스를 바탕으로 작성한 블로그 글을 어떻게 저장하고 사진 및 동영상을 제공할것인지의 기획과 , 페이지별로의 트래픽분석 (하루 방문자수등등)을 고려한 post 테이블의 DBA를 해보고
이를 insert하는 API와 컨트롤러를 제작해보고 배포합니다

---
