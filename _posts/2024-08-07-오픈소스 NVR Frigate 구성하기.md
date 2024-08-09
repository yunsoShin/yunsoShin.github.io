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

일단 첫번째로 도커를 다운로드 받습니다 .

<a href="https://www.docker.com/" target="_blank">도커 공식홈페이지</a>

```bash
 docker --version

 docker-compose --version
```

다운로드한 도커가 잘 다운받아졌는지 확인해봅니다

<h3><a href="https://yunsoshin.github.io/posts/Docker-compose%EB%9E%80/" target="_blank">docker-compose란?</a></h3>

### 1. Docker 이미지 다운로드

Frigate를 Docker를 사용하여 설치하려면 먼저 Frigate의 Docker 이미지를 다운로드합니다. 터미널을 열고 다음 명령어를 실행하세요:

```bash
docker pull blakeblackshear/frigate:stable
```

이 명령어는 Frigate의 최신 안정 버전을 다운로드합니다.

### 2. 구성 파일 생성

Frigate는 `config.yml` 파일을 통해 다양한 설정을 관리합니다. 이 파일에는 카메라 설정, 객체 감지 설정 등이 포함됩니다. 예를 들어, 한 대의 카메라를 설정하는 기본 구성은 다음과 같습니다:

```yaml
cameras:
  front_door:
    ffmpeg:
      inputs:
        - path: rtsp://your-camera-ip/rtsp-stream-url
          roles:
            - detect
            - rtmp
    width: 1920
    height: 1080
    fps: 5
```

### 3. Docker Compose 파일 작성

Docker Compose를 사용하여 Frigate를 실행하려면 `docker-compose.yml` 파일을 작성해야 합니다. 다음은 기본적인 Docker Compose 구성의 예시입니다:

```yaml
version: "3.6"
services:
  frigate:
    container_name: frigate
    restart: unless-stopped
    image: blakeblackshear/frigate:stable
    ports:
      - "5000:5000" # Web UI
      - "1935:1935" # RTMP 서비스
    volumes:
      - /path/to/config.yml:/config/config.yml
      - /path/to/clips:/media/frigate/clips
      - /path/to/recordings:/media/frigate/recordings
      - /etc/localtime:/etc/localtime:ro
    environment:
      FRIGATE_RTSP_PASSWORD: "your_rtsp_password_here"
```

### 4. Frigate 실행

Docker Compose 파일과 구성 파일을 준비한 후, Frigate 서비스를 시작하려면 다음 명령어를 사용합니다:

```bash
docker-compose up -d
```

### 5. 웹 인터페이스 접근

Frigate가 실행되면, 웹 브라우저를 통해 `http://<your-server-ip>:5000` 주소로 접속할 수 있습니다. 여기에서 실시간 비디오 스트림을 볼 수 있고, 감지된 객체의 이벤트와 기록도 확인할 수 있습니다.

기본적으로 도커내의 환경에서 구동되는 NVR을 만나 볼 수 있을겁니다.
