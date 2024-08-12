---
layout: post
title: "Frigate 설치 및 설정 가이드 (도커 사용)"
date: 2024-08-07
categories: [NVR, Home Assistant]
tags: [Frigate, Docker, Home Automation, Surveillance]
---

Frigate는 실시간 객체 감지 기능을 제공하는 NVR(Network Video Recorder) 오픈소스 소프트웨어입니다. 홈 어시스턴트와도 통합이 가능하여, 스마트 홈 시스템에서 비디오 감시를 효율적으로 관리할 수 있게 해줍니다. Frigate는 딥러닝을 활용하여 영상에서 사람, 차량 등의 객체를 실시간으로 감지하며, 이 모든 처리는 로컬에서 수행되어 개인 정보 보호에도 유리합니다.

## Frigate 설치 및 설정 방법 (도커 사용)

### 사전 준비물

- Docker가 설치된 시스템
- Docker Compose (선택사항, Docker Compose를 사용하는 경우)
- 한 개 이상의 IP 카메라 또는 네트워크에 연결된 카메라

일단 첫번째로 도커를 다운로드 받습니다

<a href="https://www.docker.com/" target="_blank">도커 공식홈페이지</a>

```bash
 docker --version

 docker-compose --version
```

다운로드한 도커가 잘 다운받아졌는지 확인해봅니다

<h3><a href="https://yunsoshin.github.io/posts/Docker-compose%EB%9E%80/" target="_blank">docker-compose란?</a></h3>

### 1. 깃허브에서 Frigate 소스 가져오기

Frigate를 구성하시려면 일단은 깃허브에서 해당 소스를 다운로드받아야합니다.

<a href="https://github.com/blakeblackshear/frigate
" target="_blank">도커 공식홈페이지</a>

```bash
https://github.com/blakeblackshear/frigate.git
```

### 2. 디렉토리 설정

```bash
.
├── docker-compose.yml
├── config/
└── storage/
```

### 3. Docker Compose 파일 작성

Docker Compose를 사용하여 Frigate를 실행하려면 `docker-compose.yml` 파일을 작성해야 합니다. 다음은 기본적인 Docker Compose 구성의 예시입니다:

```yaml
version: "3.9"
services:
  frigate:
    container_name: frigate
    restart: unless-stopped
    image: ghcr.io/blakeblackshear/frigate:stable
    volumes:
      - ./config:/config
      - ./storage:/media/frigate
      - type: tmpfs # Optional: 1GB of memory, reduces SSD/SD Card wear
        target: /tmp/cache
        tmpfs:
          size: 1000000000
    ports:
      - "8971:8971"
      - "8554:8554" # RTSP feeds

```

### 4. Frigate 실행

Docker Compose 파일과 구성 파일을 준비한 후, Frigate 서비스를 시작하려면 다음 명령어를 사용합니다:

```bash
docker-compose up -d
```

### 5. 웹 인터페이스 접근

Frigate가 실행되면, 웹 브라우저를 통해 `https://<your-server-ip>:8971` 혹은 `https://localhost:8971`
주소로 접속할 수 있습니다. 여기에서 실시간 비디오 스트림을 볼 수 있고, 감지된 객체의 이벤트와 기록도 확인할 수 있습니다.



접속에 성고했다면 로그인 페이지가 나올텐데 , 해당 페이지의 초기설정값은
`docker logs frigate` 의 명령어로 찾을 수 있습니다


로그인에 성공한 이후 좌측하단의 설정 톱니바퀴를 눌러 Configuration editor에서 카메라 설정을 변경할 수 있습니다.
자세한 문서는 아래의 글을 확인해 주세요



기본적으로 도커내의 환경에서 구동되는 NVR을 만나 볼 수 있을겁니다.
