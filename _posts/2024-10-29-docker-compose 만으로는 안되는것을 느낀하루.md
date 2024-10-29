# Docker Compose의 한계와 Kubernetes의 필요성: 실제 경험을 통해 얻은 교훈

## 들어가며

개발 초기 단계에서 **Docker Compose**는 여러 컨테이너를 손쉽게 관리할 수 있는 훌륭한 도구입니다. 그러나 프로젝트의 규모가 커지고, 서비스 간 의존성이 복잡해지면 Docker Compose만으로는 그 한계에 부딪히게 됩니다. 이번 글에서는 제가 실제로 겪은 Docker Compose의 복잡성 문제와 **Kubernetes**로 전환하게 된 이유를 공유하고자 합니다.

## Docker Compose로 구성한 복잡한 서비스들

프로젝트에서는 다음과 같은 다양한 서비스를 Docker Compose로 관리하고 있었습니다:

- **서비스 레이어**: `backend`, `mysql`
- **메시지 브로커 레이어**: `kafka`, `zookeeper`
- **인메모리 캐싱 DB 레이어**: `redis`
- **로깅 레이어**: ELK 스택 (`elasticsearch`, `logstash`, `kibana`)
- **CI/CD 레이어**: `jenkins`

아래는 이 모든 서비스를 하나의 `docker-compose.yml` 파일로 구성한 전체 코드입니다.

```yaml
version: "3.8"

services:
  backend:
    build:
      context: ../backend
      dockerfile: ../docker/dockerfiles/dockerfile.backend
    container_name: ${DOMAIN_NAME}_backend
    ports:
      - "7778:7778"
    environment:
      - DB_HOST=mysql # mysql 컨테이너 이름
      - DB_PORT=3306
      - DB_USER=${DB_USER}
      - DB_PASSWORD=${DB_PASSWORD}
      - DB_NAME=${DB_NAME}
      - IS_DEV=${IS_DEV}
      - JWT_SECRET=${JWT_SECRET}
      - BACKEND_PORT=${BACKEND_PORT}
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      mysql:
        condition: service_healthy
      redis:
        condition: service_healthy
    networks:
      - mynetwork
    restart: always

  mysql:
    build:
      context: ./
      dockerfile: ./dockerfiles/dockerfile.mysql
    container_name: ${DOMAIN_NAME}_mysql
    environment:
      - MYSQL_ROOT_PASSWORD=${DB_PASSWORD}
      - MYSQL_DATABASE=${DB_NAME}
    ports:
      - "6654:3306"
    volumes:
      - db_starter_data:/var/lib/mysql
      - ./mysql/my.cnf:/etc/mysql/my.cnf
      - ./mysql/init.sql:/docker-entrypoint-initdb.d/init.sql
    healthcheck:
      test:
        [
          "CMD-SHELL",
          "mysqladmin ping -h mysql -u root -p${MYSQL_ROOT_PASSWORD} || exit 1"
        ]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 40s
    networks:
      - mynetwork

  elasticsearch:
    image: docker.elastic.co/elasticsearch/elasticsearch:8.9.0
    container_name: ${DOMAIN_NAME}_elasticsearch
    environment:
      - discovery.type=single-node
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    networks:
      - mynetwork
    volumes:
      - es_starter_data:/usr/share/elasticsearch/data

  logstash:
    image: docker.elastic.co/logstash/logstash:8.9.0
    container_name: ${DOMAIN_NAME}_logstash
    ports:
      - "5044:5044"
    volumes:
      - ./elk/logstash.conf:/usr/share/logstash/pipeline/logstash.conf
      - ./elk/pipelines.yml:/usr/share/logstash/config/pipelines.yml
    networks:
      - mynetwork
    depends_on:
      - elasticsearch

  kibana:
    image: docker.elastic.co/kibana/kibana:8.9.0
    container_name: ${DOMAIN_NAME}_kibana
    ports:
      - "5601:5601"
    environment:
      - ELASTICSEARCH_HOSTS=http://elasticsearch:9200
      - xpack.security.enabled=false
    networks:
      - mynetwork
    depends_on:
      - elasticsearch

  jenkins:
    image: jenkins/jenkins:lts
    container_name: ${DOMAIN_NAME}_jenkins
    ports:
      - "8080:8080"
      - "50000:50000"
    volumes:
      - ./jenkins_starter_home:/var/jenkins_home
    networks:
      - mynetwork
    environment:
      - JAVA_OPTS=-Djenkins.install.runSetupWizard=false
    restart: always
    depends_on:
      - mysql
      - elasticsearch

  zookeeper:
    image: confluentinc/cp-zookeeper:latest
    container_name: ${DOMAIN_NAME}_zookeeper
    ports:
      - "2181:2181"
    environment:
      - ZOOKEEPER_CLIENT_PORT=2181
    networks:
      - mynetwork
    restart: always

  kafka:
    image: wurstmeister/kafka:latest
    container_name: ${DOMAIN_NAME}_kafka
    ports:
      - "9092:9092"
    environment:
      - KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092
      - KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
    depends_on:
      - zookeeper
    networks:
      - mynetwork
    restart: always

  redis:
    image: redis:latest
    container_name: ${DOMAIN_NAME}_redis
    ports:
      - "6379:6379"
    networks:
      - mynetwork
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 30s
      timeout: 10s
      retries: 5
      start_period: 10s
    restart: always

volumes:
  db_starter_data:
  es_starter_data:
  jenkins_starter_home:

networks:
  mynetwork:
    driver: bridge
```

## Docker Compose의 한계

### 1. 네트워크 구성의 복잡성 증가

**다양한 클러스터링 환경에서의 어려움**

5개의 레이어로 나누어 각 Compose 파일을 구성하는 작업은 처음에는 논리적이고 명확해 보였습니다. 그러나 각 레이어가 서로 다른 로컬 머신에 배포되거나, 한 머신 내에서 여러 레이어가 결합된 형태로 구성될 때 네트워크와 의존성 관리에 큰 어려움이 생겼습니다.

- **로컬 호스트 간 네트워크**: 같은 로컬 호스트에서 여러 레이어가 존재할 경우에는 `bridge` 네트워크를 통해 통신하도록 설정하면 됩니다. 그러나 서로 다른 호스트 머신에 각 레이어가 분리될 경우에는 외부 네트워크를 통해 통신해야 하므로 네트워크 설정이 복잡해졌습니다.
- **내부 통신과 외부 통신 간 전환의 어려움**: 동일한 호스트에서 실행될 때는 내부 네트워크(shared network)를 사용하여 효율적인 통신이 가능하지만, 여러 호스트에 걸쳐 있는 경우에는 외부 네트워크 통신으로 전환이 필요합니다. 이 과정에서 네트워크 설정을 유연하게 유지하기 어려웠습니다.

### 2. 환경별 네트워크 구성의 불일치

각 환경마다 네트워크 구성이 다르기 때문에 Compose 파일을 일관되게 유지하기 어려웠습니다. 예를 들어, 로컬 개발 환경에서는 모든 레이어를 하나의 머신에 올려도 문제가 없었지만, 프로덕션 환경에서는 레이어별로 여러 서버에 분산 배치해야 했습니다.

- **공유 네트워크 설정의 어려움**: Docker Compose에서는 네트워크 이름을 외부에서 생성하고 이를 공유하도록 설정해야 합니다. 그러나 네트워크 이름을 환경 변수로 설정하는 것이 불가능하여 각 환경에 맞춰 파일을 수동으로 수정해야 했습니다.
- **서비스 설정의 복잡성 증가**: 각 환경마다 서비스가 통신할 호스트나 포트가 달라 이를 관리하기 위한 추가적인 스크립트나 설정이 필요했습니다.

### 3. 서비스 의존성 관리의 복잡성

`depends_on` 옵션을 통해 서비스 간의 의존성을 명시할 수 있었지만, 여러 Compose 파일로 분리되면서 서로 다른 Compose 파일에 정의된 서비스 간의 의존성을 관리하기가 어려워졌습니다. 특히 클러스터 내 여러 호스트에서 각 레이어가 분리되어 실행될 때 의존성 상태를 일관성 있게 관리하는 것이 매우 복잡해졌습니다.

### 4. 서비스별 스케일링의 한계

Docker Compose는 여러 서비스를 쉽게 관리할 수 있지만, **특정 서비스에만 부하가 집중될 때** 해당 서비스의 클러스터 가용성을 늘리는 데에는 한계가 있습니다.

- **개별 서비스의 스케일링 어려움**: Docker Compose에서는 `docker-compose up --scale` 명령어를 통해 서비스의 컨테이너 수를 늘릴 수 있지만, 이는 단순히 컨테이너 수를 늘리는 것에 불과합니다. 부하 분산이나 서비스 디스커버리와 같은 고급 기능을 제공하지 않아, 추가적인 설정이 필요합니다.
- **자동화된 스케일링 미지원**: 부하에 따라 자동으로 스케일링하는 기능이 없어, 수동으로 컨테이너 수를 조절해야 하는 불편함이 있습니다.

### 5. 부분적인 가용성 향상의 어려움

**다수의 서비스 중 일부 서비스만 가용성을 늘리고자 할 때**, Docker Compose는 이에 대한 유연한 대처가 어렵습니다.

- **의존성 관리의 복잡성**: 특정 서비스만 스케일링하면, 해당 서비스와 연관된 다른 서비스들의 설정도 함께 변경해야 할 수 있어 관리가 복잡해집니다.
- **서비스 간 영향**: 하나의 Compose 파일 내에서 서비스들이 밀접하게 연결되어 있어, 일부 서비스를 변경하는 것이 다른 서비스에 영향을 줄 수 있습니다.

### 6. 네트워크 전환 및 로드밸런싱의 어려움

**다중 클러스터링 환경에서 로컬 머신에 여러 컨테이너를 두고 공유 네트워크로 관리하고 싶을 때**, 내부 네트워크의 컨테이너들이 트래픽 과부하에 시달린다면 외부 네트워크로 자연스럽게 전환하여 로드밸런싱을 하고자 합니다. 그러나 Docker Compose에서는 이러한 전략을 구현하기가 어렵습니다.

- **동적인 네트워크 설정의 부재**: Docker Compose는 정적인 네트워크 설정만을 지원하며, 트래픽 상황에 따라 네트워크 구성을 동적으로 변경하는 기능이 없습니다.
- **로드밸런싱 기능의 한계**: 기본적으로 로드밸런싱을 지원하지 않아, 외부 도구나 추가적인 설정이 필요합니다.

## Kubernetes의 필요성 인식

이러한 문제들을 해결하기 위해 **Kubernetes**를 도입하게 되었습니다. Kubernetes는 복잡한 마이크로서비스 아키텍처를 관리하고 확장할 수 있는 강력한 기능을 제공합니다.

### Kubernetes의 주요 이점

#### 1. 유연한 네트워킹

- **자동화된 네트워크 구성**: Kubernetes에서는 모든 파드가 클러스터 내에서 일관된 네트워크를 공유하므로, 서비스 간 통신을 위해 별도의 네트워크 설정이 필요하지 않습니다.
- **서비스 디스커버리**: Kubernetes의 Service 리소스를 통해 동적으로 생성되는 파드의 IP를 추적하지 않고도 안정적인 서비스 엔드포인트를 제공합니다.

#### 2. 환경별 일관성 제공

- **매니페스트 파일 관리**: YAML 매니페스트 파일을 통해 환경에 관계없이 동일한 설정으로 배포가 가능합니다.
- **ConfigMap과 Secret**: 환경별 설정 값을 관리하기 용이하며, 이는 환경 변수로 쉽게 주입할 수 있습니다.

#### 3. 자동화된 의존성 및 상태 관리

- **헬스 체크와 레디니스 프로브**: 서비스의 상태를 지속적으로 모니터링하여 자동으로 재시작하거나 트래픽을 조절합니다.
- **의존성 관리**: Init Containers나 PostStart 훅을 통해 서비스 간의 의존성을 관리할 수 있습니다.

#### 4. 서비스별 스케일링의 유연성

- **자동 스케일링 지원**: Kubernetes의 HPA(Horizontal Pod Autoscaler)를 통해 부하에 따라 특정 서비스의 파드(Pod) 수를 자동으로 조절할 수 있습니다.
- **개별 서비스 관리 용이**: Deployment 리소스를 사용하여 서비스별로 독립적인 스케일링이 가능합니다.

#### 5. 부분적인 가용성 향상

- **선택적 스케일링**: 필요한 서비스만 선택적으로 스케일링할 수 있어, 리소스 활용의 효율성을 높일 수 있습니다.
- **의존성 분리**: 서비스 간 의존성을 명확하게 정의하고 분리하여, 한 서비스의 변경이 다른 서비스에 미치는 영향을 최소화합니다.

#### 6. 네트워크 전환 및 로드밸런싱

- **서비스 디스커버리 및 로드밸런싱 내장**: Kubernetes의 Service 리소스는 클러스터 내에서 자동으로 로드밸런싱을 제공하며, DNS를 통해 서비스 디스커버리를 지원합니다.
- **동적인 네트워크 관리**: 네트워크 정책(Network Policy)을 사용하여 트래픽을 동적으로 관리하고, 필요에 따라 외부로의 네트워크 전환이 가능합니다.
- **Ingress 컨트롤러**: 외부 트래픽에 대한 로드밸런싱과 라우팅을 관리하여, 내부 및 외부 네트워크 간의 유연한 전환을 지원합니다.

## Kubernetes 도입 후의 변화

Kubernetes를 도입한 후, 우리는 다음과 같은 이점을 얻을 수 있었습니다.

- **운영 효율성 향상**: 자동화된 배포와 스케일링으로 운영 부담이 크게 감소했습니다.
- **높은 가용성 확보**: 헬스 체크와 자동 복구 기능을 통해 서비스의 안정성이 향상되었습니다.
- **리소스 최적화**: 필요한 서비스에만 리소스를 집중 투입하여 비용 효율성을 높였습니다.
- **유연한 네트워크 구성**: 네트워크 정책과 Ingress를 활용하여 복잡한 네트워크 환경에서도 유연한 트래픽 관리가 가능해졌습니다.

## 결론

Docker Compose는 초기 개발 단계에서 여러 컨테이너를 쉽게 관리할 수 있는 훌륭한 도구입니다. 그러나 서비스 규모가 커지고 복잡성이 증가하면 Docker Compose만으로는 한계에 부딪히게 됩니다. 특히, 서비스별 스케일링, 부분적인 가용성 향상, 네트워크 전환 및 로드밸런싱과 같은 복잡한 요구 사항을 처리하기 위해서는 Kubernetes와 같은 컨테이너 오케스트레이션 도구의 도입이 필요합니다.

실제 경험을 통해, Kubernetes는 복잡한 서비스 아키텍처를 효과적으로 관리하고 확장할 수 있는 강력한 기능을 제공한다는 것을 알게 되었습니다. 초기 설정과 학습 곡선이 다소 가파를 수 있지만, 장기적으로 볼 때 그 이점은 투자할 가치가 충분하다고 생각합니다.

---

Docker Compose의 한계를 직접 경험하면서, 복잡한 서비스 환경에서 Kubernetes의 필요성을 절실히 느꼈습니다. 이 글이 비슷한 고민을 하고 있는 분들께 도움이 되길 바랍니다.
