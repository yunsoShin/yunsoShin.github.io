---
layout: post
title: "Stencil에서 정적 파일을 이용한 SEO 최적화: RSS, Sitemap, Robots.txt 설정하기"
date: 2024-05-12
categories: web-development seo stencil vercel
---

## 도입부

이 포스트에서는 Stencil 프로젝트에서 SEO를 향상시키기 위해 RSS, sitemap, 및 robots.txt 파일을 정적 파일로 추가하고, 이를 Vercel을 통해 배포하는 과정을 소개하려고 합니다.

### 배경

웹 개발에서 검색 엔진 최적화(SEO)는 매우 중요합니다. 효과적인 SEO 설정을 통해 검색 엔진에서 더 높은 순위를 얻고, 이로 인해 웹사이트의 트래픽과 가시성이 증가할 수 있습니다.

## Stencil 프로젝트 설정

Stencil은 웹 컴포넌트를 빌드하기 위한 도구이며, 다음과 같이 프로젝트를 설정합니다.

1. `assets` 디렉토리에 SEO 관련 파일을 저장합니다.
2. 기본적으로 stencil읭 빌드 경로 `www/assets/` 파일들이 해당 위치에 저장됩니다.
3. SEO폴더안의 정적, 혹은 동적으로 생성된 SEO 파일들을 vercel.json으로 라우팅처리를 해줍니다.

### 파일 구조

www/<br>
└── assets/SEO/<br>
├── sitemap.xml<br>
├── rss.xml<br>
└── robots.txt

## Vercel에서의 배포

Vercel을 사용하여 Stencil 프로젝트를 배포하는 과정은 다음과 같습니다.

### 설정 파일 (`vercel.json`)

```json
{
  "rewrites": [
    { "source": "/sitemap.xml", "destination": "/assets/SEO/sitemap.xml" },
    { "source": "/rss.xml", "destination": "/assets/SEO/rss.xml" },
    { "source": "/robots.txt", "destination": "/assets/SEO/robots.txt" }
  ],
  "headers": [
    {
      "source": "/rss.xml",
      "headers": [
        {
          "key": "Content-Type",
          "value": "application/xml; charset=utf-8"
        }
      ]
    },
    {
      "source": "/sitemap.xml",
      "headers": [
        {
          "key": "Content-Type",
          "value": "application/xml; charset=utf-8"
        }
      ]
    },
    {
      "source": "/robots.txt",
      "headers": [
        {
          "key": "Content-Type",
          "value": "text/plain; charset=utf-8"
        },
        {
          "key": "Cache-Control",
          "value": "public, max-age=86400"
        }
      ]
    }
  ]
}
```

라우팅과 파일 타입 설정
위의 vercel.json 설정을 통해 각 파일에 적절한 Content-Type 헤더를 설정하고, 캐시 제어를 위한 Cache-Control 헤더를 추가하여 SEO를 최적화할 수 있습니다.

결론
이 글에서는 Stencil과 Vercel을 사용하여 RSS, sitemap, 및 robots.txt 파일을 관리하고 SEO를 개선하는 방법을 알아보았습니다. 이 설정들은 검색 엔진에서 더 나은 순위를 얻는 데 도움이 됩니다.
