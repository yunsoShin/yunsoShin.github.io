---
layout: post
title: "NestJS에서 API 생성 시 테스트 코드 작성 가이드"
date: 2024-08-25 10:00:00 +0900
categories: [NestJS, Testing, TypeScript]
tags: [NestJS, 테스트, Mock, 유닛 테스트, 통합 테스트]
---

### NestJS에서 API 생성 시 테스트 코드 작성 가이드

NestJS를 사용해 API를 개발할 때, 테스트 코드 작성은 필수적입니다. 특히, 모듈화된 구조와 의존성 주입(DI) 방식을 사용하는 NestJS에서는 테스트 코드 작성이 비교적 수월하지만, 초기 설정 과정에서 몇 가지 중요한 사항을 고려해야 합니다. 이 글에서는 API 생성 후 테스트 코드를 작성하기 위한 기본 설정과 Mock 설정 방법, 그리고 유의사항을 다룹니다.

#### 배경

저는 개인 블로그를 만들어 포스팅된 글들을 관리하고, 방문자들의 반응을 추적하고 싶었습니다. 특히, 각 글이 얼마나 많은 조회수를 기록했는지 알고 싶었습니다.

NestJS는 모듈화된 구조와 의존성 주입(DI) 방식을 통해 유지보수성과 확장성을 높여주는 훌륭한 프레임워크입니다. 하지만 개발 과정에서 가장 중요한 부분 중 하나는 **테스트 코드** 작성이었습니다. 서비스와 컨트롤러가 기대한 대로 동작하는지 확인하고, 추후 수정 및 확장 시 발생할 수 있는 문제를 미리 방지하기 위해서는 철저한 테스트가 필수적이었습니다.

이 글에서는 제가 블로그 글의 조회수를 추적하는 API를 개발하며 사용한 테스트 코드 작성법을 소개합니다. API 생성 후 테스트 코드를 작성하기 위한 기본 설정과 Mock 설정 방법, 그리고 유의사항을 다루며, 여러분도 이와 같은 상황에서 쉽게 적용할 수 있도록 안내합니다.

<!-- more -->

#### 1. 초기 API 설정

API를 개발할 때 먼저 리소스(Resource)를 생성합니다. NestJS CLI에서 `resource` 커맨드를 사용하면 컨트롤러(Controller), 서비스(Service), 그리고 관련된 모든 파일들을 자동으로 생성할 수 있습니다.

```bash
nest g resource posts
```

이 명령어를 통해 `posts`라는 이름의 리소스가 생성되며, 여기에 컨트롤러와 서비스가 포함됩니다. 이후, `PostsController`와 `PostsService`에서 기본적인 API 엔드포인트와 서비스 메소드를 작성할 수 있습니다.

생성된 파일들은 프로젝트 구조에서 다음과 같이 나타나게 됩니다:

```plaintext
src/
└── posts/
    ├── dto/
    ├── entities/
    │   └── post.entity.ts
    ├── posts.controller.spec.ts
    ├── posts.controller.ts
    ├── posts.module.ts
    ├── posts.service.spec.ts
    └── posts.service.ts
```

이 폴더 구조는 각각의 파일이 어떤 역할을 하는지 명확하게 나눠줍니다. `entities` 폴더에는 데이터베이스와 연관된 엔티티가, `dto` 폴더에는 데이터 전송 객체(DTO)가 위치하게 됩니다. 이 구조는 프로젝트의 유지보수성과 가독성을 높여줍니다.

#### 2. Post 엔티티 정의

우선, 블로그 글에 대한 정보를 담고 있는 `Post` 엔티티를 정의합니다. 이 엔티티는 데이터베이스에 저장되는 테이블 구조를 정의하며, 블로그 글의 제목, 내용, 조회수, 작성 일자 등을 포함합니다.

```typescript
import {
  Entity,
  Column,
  PrimaryGeneratedColumn,
  CreateDateColumn
} from "typeorm";

@Entity("post") // 테이블 이름을 스네이크 케이스로 설정
export class Post {
  @PrimaryGeneratedColumn({ name: "id" }) // 자동 생성된 기본 키 컬럼
  id: number;

  @Column({ name: "title" }) // 컬럼명을 스네이크 케이스로 지정
  title: string;

  @Column({ name: "content" })
  content: string;

  @Column({ name: "is_published", default: true }) // 컬럼명을 스네이크 케이스로 지정
  isPublished: boolean;

  @Column({ name: "views", default: 0 }) // 기본값을 0으로 설정하고 스네이크 케이스 적용
  views: number;

  @CreateDateColumn({ name: "created_at" }) // 자동 생성 시간 컬럼을 스네이크 케이스로 지정
  createdAt: Date;
}
```

`Post` 엔티티는 다음과 같은 필드를 가집니다:

- `id`: 자동 생성되는 고유 식별자
- `title`: 블로그 글의 제목
- `content`: 블로그 글의 내용
- `isPublished`: 블로그 글이 발행되었는지 여부를 나타내는 boolean 값 (기본값: true)
- `views`: 조회수 (기본값: 0)
- `createdAt`: 글이 생성된 날짜와 시간을 자동으로 기록

이 엔티티를 통해 데이터베이스 테이블이 생성되며, 이후 서비스에서 이 데이터를 관리하게 됩니다.

#### 3. 서비스 및 컨트롤러 구현

이제 블로그 글의 조회수를 증가시키는 API를 구현합니다. 이 API는 특정 글의 조회수를 1 증가시키고, 업데이트된 글 데이터를 반환합니다.

```typescript
// posts.controller.ts
import { Controller, Patch, Param } from "@nestjs/common";
import { PostsService } from "./posts.service";

@Controller("posts")
export class PostsController {
  constructor(private readonly postsService: PostsService) {}

  @Patch(":id/views")
  incrementViews(@Param("id") id: string) {
    return this.postsService.incrementViews(+id);
  }
}

// posts.service.ts
import { Injectable, NotFoundException } from "@nestjs/common";
import { InjectRepository } from "@nestjs/typeorm";
import { Repository } from "typeorm";
import { Post } from "./entities/post.entity";

@Injectable()
export class PostsService {
  constructor(
    @InjectRepository(Post)
    private readonly postsRepository: Repository<Post>
  ) {}

  async incrementViews(id: number): Promise<Post> {
    const post = await this.postsRepository.findOne({ where: { id } });
    if (!post) {
      throw new NotFoundException(`Post with id ${id} not found`);
    }
    post.views += 1;
    return this.postsRepository.save(post);
  }
}
```

#### 4. 테스트 코드 작성

NestJS에서는 `@nestjs/testing` 패키지를 사용하여 유닛 테스트와 통합 테스트를 손쉽게 작성할 수 있습니다. 여기서는 `PostsService`와 `PostsController`에 대한 유닛 테스트를 작성해보겠습니다.

##### 4.1. Mock 설정

테스트 코드를 작성할 때 실제 데이터베이스를 사용하지 않도록 Repository를 Mock으로 설정합니다. 이를 통해 테스트의 독립성을 유지하고, 빠르고 안정적인 테스트를 수행할 수 있습니다.

```typescript
// posts.service.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import { PostsService } from "./posts.service";
import { getRepositoryToken } from "@nestjs/typeorm";
import { Post } from "./entities/post.entity";
import { Repository } from "typeorm";
import { NotFoundException } from "@nestjs/common";

const mockPostRepository = {
  findOne: jest.fn(),
  save: jest.fn()
};

describe("PostsService", () => {
  let service: PostsService;
  let repository: Repository<Post>;

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      providers: [
        PostsService,
        {
          provide: getRepositoryToken(Post),
          useValue: mockPostRepository
        }
      ]
    }).compile();

    service = module.get<PostsService>(PostsService);
    repository = module.get<Repository<Post>>(getRepositoryToken(Post));
  });

  it("should increment the views of the post", async () => {
    const postId = 1;
    const post = { id: postId, views: 0 } as Post;

    mockPostRepository.findOne.mockResolvedValue(post);
    mockPostRepository.save.mockResolvedValue({ ...post, views: 1 });

    const updatedPost = await service.incrementViews(postId);

    expect(updatedPost.views).toBe(1);
    expect(mockPostRepository.findOne).toHaveBeenCalledWith({
      where: { id: postId }
    });
    expect(mockPostRepository.save).toHaveBeenCalledWith({ ...post, views: 1 });
  });

  it("should throw NotFoundException if post is not found", async () => {
    const postId = 999;

    mockPostRepository.findOne.mockResolvedValue(null);

    await expect(service.incrementViews(postId)).rejects.toThrow(
      NotFoundException
    );
  });
});
```

##### 4.2. 컨트롤러 테스트

`PostsController`에서 `PostsService`의 메소드가 올바르게 호출되는지 테스트합니다.

```typescript
// posts.controller.spec.ts
import { Test, TestingModule } from "@nestjs/testing";
import { PostsController } from "./posts.controller";
import { PostsService } from "./posts.service";

describe("PostsController", () => {
  let controller: PostsController;
  let service: PostsService;

  const mockPostsService = {
    incrementViews: jest.fn()
  };

  beforeEach(async () => {
    const module: TestingModule = await Test.createTestingModule({
      controllers: [PostsController],
      providers: [
        {
          provide: PostsService,
          useValue: mockPostsService
        }
      ]
    }).compile();

    controller = module.get<PostsController>(PostsController);
    service = module.get<PostsService>(PostsService);
  });

  it("

should call incrementViews on the service with the correct id", async () => {
    const postId = "1";
    const mockPost = { id: 1, views: 1 };

    mockPostsService.incrementViews.mockResolvedValue(mockPost);

    const result = await controller.incrementViews(postId);

    expect(result).toEqual(mockPost);
    expect(service.incrementViews).toHaveBeenCalledWith(1);
  });
});
```

#### 5. 테스트 실행

테스트 코드를 작성한 후, `npm test` 명령어를 사용하여 테스트를 실행합니다. 모든 테스트가 성공해야 하며, 실패한 테스트가 있을 경우 코드를 다시 검토하고 수정합니다.

```bash
npm test
```

#### 6. 유의사항

- **Mock 설정의 중요성**: 실제 데이터베이스에 의존하지 않도록 Mock을 잘 설정해야 합니다. 이렇게 하면 테스트가 독립적으로 수행되며, 테스트가 실패해도 데이터베이스가 오염되지 않습니다.
- **에러 핸들링 테스트**: 비정상적인 상황을 시뮬레이션하여 에러 핸들링이 제대로 이루어지는지 확인하는 테스트를 포함하는 것이 좋습니다.
- **테스트 코드의 유지보수**: 코드 변경 시 테스트 코드도 함께 업데이트해야 합니다. 테스트 코드가 실제 코드와 일치하지 않으면 의미가 없습니다.

#### 결론

NestJS에서 API를 생성할 때, 테스트 코드는 반드시 작성해야 할 중요한 요소입니다. 서비스와 컨트롤러 각각에 대해 유닛 테스트를 작성하고, 데이터베이스와의 상호작용은 Mock을 통해 처리하는 것이 좋은 접근 방식입니다. 이를 통해 안정적이고 유지보수 가능한 코드를 작성할 수 있습니다. NestJS의 유연한 테스트 모듈을 활용하여, 테스트 가능한 코드 구조를 만들고, 코드의 품질을 높여보세요.

---

이제 폴더 구조와 해당 파일의 역할을 명시하여 독자들이 프로젝트 구조를 더 쉽게 이해하고, 테스트 코드를 작성하는 방법을 명확하게 따라할 수 있도록 했습니다.
