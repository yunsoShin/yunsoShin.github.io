### Docker Compose: 개요 및 구현 방법

Docker Compose는 여러 컨테이너를 정의하고 실행하는 데 사용되는 도구입니다. 특히, 여러 서비스가 상호 작용하는 복잡한 애플리케이션의 경우 유용합니다. Docker Compose를 사용하면 여러 컨테이너로 이루어진 애플리케이션을 쉽게 정의하고 배포할 수 있습니다.

이 글에서는 Docker Compose의 기본 개념과 함께 이를 활용하여 애플리케이션을 구축하고 배포하는 방법을 단계별로 설명하겠습니다.

---

## 1. Docker Compose란?

Docker Compose는 `docker-compose.yml`이라는 파일을 사용하여 애플리케이션의 서비스, 네트워크, 볼륨 등을 정의합니다. `docker-compose.yml` 파일에 정의된 내용을 바탕으로 `docker-compose up` 명령어를 통해 모든 컨테이너를 일괄적으로 실행할 수 있습니다.

### 주요 기능

- **멀티 컨테이너 애플리케이션**: 복잡한 애플리케이션을 여러 컨테이너로 나누어 각각의 역할을 분리할 수 있습니다.
- **환경 설정 관리**: 환경 변수, 네트워크 설정 등을 손쉽게 관리할 수 있습니다.
- **서비스 조정**: 여러 서비스 간의 의존성 관리 및 자동 시작/종료가 가능합니다.
- **개발 환경 구축**: 일관된 개발 환경을 유지할 수 있어 협업에 유리합니다.

---

## 2. Docker Compose 설치

### 설치 방법 (Linux/MacOS):

Docker가 이미 설치되어 있다면 Docker Compose도 함께 설치되어 있을 가능성이 높습니다. 그러나 최신 버전을 설치하거나 확인하려면 다음 명령어를 사용할 수 있습니다.

```bash
# Linux 설치 명령어
sudo curl -L "https://github.com/docker/compose/releases/download/$(curl -s https://api.github.com/repos/docker/compose/releases/latest | grep -oP '(?<=tag_name": ")[^"]*')" -o /usr/local/bin/docker-compose

# 실행 권한 부여
sudo chmod +x /usr/local/bin/docker-compose

# 설치 확인
docker-compose --version
```

### 설치 방법 (Windows):

Windows에서는 Docker Desktop을 설치하면 Docker Compose가 자동으로 포함됩니다.

---

## 3. Docker Compose 구성 파일 작성 (`docker-compose.yml`)

Docker Compose의 핵심은 `docker-compose.yml` 파일입니다. 이 파일에 모든 서비스, 볼륨, 네트워크 설정을 정의합니다.

### 예제: 간단한 웹 애플리케이션

아래는 간단한 웹 애플리케이션을 정의한 `docker-compose.yml` 파일의 예시입니다. 이 예제에서는 웹 서버(Nginx)와 데이터베이스(MySQL)를 정의합니다.

```yaml
version: "3"
services:
  web:
    image: nginx:latest
    ports:
      - "8080:80"
    volumes:
      - ./html:/usr/share/nginx/html
    depends_on:
      - db

  db:
    image: mysql:latest
    environment:
      MYSQL_ROOT_PASSWORD: example_password
      MYSQL_DATABASE: example_db
      MYSQL_USER: example_user
      MYSQL_PASSWORD: example_user_password
    volumes:
      - db_data:/var/lib/mysql

volumes:
  db_data:
```

### 구성 파일 설명:

- `version`: Docker Compose 파일의 버전입니다. 일반적으로 `3` 또는 그 이상의 버전을 사용합니다.
- `services`: 각 서비스(컨테이너)를 정의하는 블록입니다.
  - `web`: Nginx 웹 서버를 실행하는 서비스입니다.
    - `image`: 사용할 Docker 이미지를 지정합니다.
    - `ports`: 로컬 포트와 컨테이너 포트를 연결합니다.
    - `volumes`: 호스트 시스템의 디렉토리를 컨테이너 내부에 마운트합니다.
    - `depends_on`: `db` 서비스가 먼저 실행된 후 `web` 서비스가 실행되도록 설정합니다.
  - `db`: MySQL 데이터베이스를 실행하는 서비스입니다.
    - `environment`: MySQL 초기 설정을 위한 환경 변수를 정의합니다.
    - `volumes`: 데이터베이스 데이터를 지속적으로 저장할 볼륨을 설정합니다.
- `volumes`: 서비스 간에 공유되거나 지속적인 데이터를 저장할 볼륨을 정의합니다.

---

## 4. Docker Compose 실행

### 애플리케이션 시작

`docker-compose.yml` 파일이 준비되었으면 다음 명령어로 애플리케이션을 실행할 수 있습니다.

```bash
docker-compose up
```

이 명령어는 모든 서비스(컨테이너)를 정의된 순서에 따라 시작합니다. 만약 백그라운드에서 실행하고 싶다면 `-d` 플래그를 추가합니다:

```bash
docker-compose up -d
```

### 애플리케이션 종료

실행 중인 모든 서비스를 종료하려면 다음 명령어를 사용합니다.

```bash
docker-compose down
```

이 명령어는 모든 컨테이너를 중지하고, 생성된 네트워크와 볼륨을 삭제합니다(단, 명시적으로 정의된 볼륨은 삭제되지 않습니다).

### 개별 서비스 실행 및 종료

특정 서비스만 실행하거나 종료하려면 서비스 이름을 명령어 뒤에 추가합니다:

```bash
# 특정 서비스 실행
docker-compose up web

# 특정 서비스 종료
docker-compose stop web
```

### 로그 확인

모든 서비스의 로그를 실시간으로 확인하려면 다음 명령어를 사용합니다:

```bash
docker-compose logs -f
```

---

## 5. 실제 사용 사례

### 다중 컨테이너 애플리케이션 배포

실제 환경에서는 프론트엔드, 백엔드, 데이터베이스, 캐시 서버 등을 각각의 서비스로 나누어 Docker Compose를 통해 배포할 수 있습니다.

예를 들어, Node.js 백엔드와 React 프론트엔드, Redis 캐시 서버를 사용하는 애플리케이션의 `docker-compose.yml` 파일은 다음과 같이 구성할 수 있습니다:

```yaml
version: "3"
services:
  frontend:
    build: ./frontend
    ports:
      - "3000:3000"

  backend:
    build: ./backend
    ports:
      - "5000:5000"
    depends_on:
      - redis
    environment:
      REDIS_HOST: redis
      REDIS_PORT: 6379

  redis:
    image: "redis:alpine"
    ports:
      - "6379:6379"
```

이 설정은 프론트엔드와 백엔드의 개발 환경을 독립적으로 구성하면서, Redis와 같은 서비스를 공유하여 효율적인 작업이 가능합니다.

---

## 6. Docker Compose의 고급 기능

### 스케일링 (Scaling)

Docker Compose를 사용하면 동일한 서비스를 여러 개의 인스턴스로 확장할 수 있습니다. 예를 들어, 웹 서버를 3개의 컨테이너로 확장하려면 다음 명령어를 사용합니다:

```bash
docker-compose up --scale web=3
```

이 기능은 특히 부하 분산이나 고가용성을 필요로 하는 환경에서 유용합니다.

### 환경 변수 파일 (.env)

Docker Compose는 `.env` 파일을 통해 환경 변수를 설정할 수 있습니다. 예를 들어, 다음과 같은 `.env` 파일을 작성할 수 있습니다:

```env
MYSQL_ROOT_PASSWORD=example_password
MYSQL_DATABASE=example_db
```

그 후 `docker-compose.yml` 파일에서 이를 참조할 수 있습니다:

```yaml
environment:
  - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
  - MYSQL_DATABASE=${MYSQL_DATABASE}
```

---

## 결론

Docker Compose는 다중 컨테이너 애플리케이션의 배포 및 관리에 매우 유용한 도구입니다. `docker-compose.yml` 파일을 통해 애플리케이션의 모든 측면을 정의하고, 단일 명령어로 모든 서비스를 일관되게 실행할 수 있습니다. 이를 통해 개발자는 일관된 환경을 유지하고, 복잡한 애플리케이션을 효율적으로 관리할 수 있습니다.

이제 Docker Compose를 사용하여 자신의 프로젝트에 맞는 환경을 구축해보세요. Docker Compose의 강력한 기능을 활용하면 배포, 테스트, 개발 과정이 훨씬 간단해질 것입니다.
