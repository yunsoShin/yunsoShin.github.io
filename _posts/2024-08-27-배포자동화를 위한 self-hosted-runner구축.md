### **GitHub Actions Self-Hosted Runner 구성 및 배포 자동화**

현대 소프트웨어 개발에서는 자동화된 배포 프로세스가 필수적입니다. 이 블로그 포스트에서는 로컬 호스트 머신에서 SSH 키를 생성하고, 이를 사용하여 도커 컨테이너 내에서 GitHub Actions Self-Hosted Runner를 구성하는 방법을 알아보겠습니다. 최종적으로 이 Runner를 통해 도커 컨테이너에서 호스트 머신으로 SSH 접근을 하여 배포 작업을 자동화하는 방법도 다룰 것입니다.

---

## **1. SSH 키 생성 및 설정**

먼저, 도커 컨테이너에서 사용할 SSH 키를 호스트 머신에서 생성해야 합니다. 이 SSH 키는 도커 컨테이너에서 호스트 머신으로의 원격 접근을 허용하는 데 사용됩니다.

### **1.1. SSH 키 생성**

아래 명령어를 사용하여 호스트 머신에 SSH 키를 생성합니다:

```bash
ssh-keygen -t rsa -b 4096 -f ~/.ssh/host_rsa -N ""
chmod 600 ~/.ssh/host_rsa
```

- `~/.ssh/host_rsa`: 생성된 개인 키의 경로입니다.
- `~/.ssh/host_rsa.pub`: 생성된 공개 키의 경로입니다.

### **1.2. 공개 키를 authorized_keys에 추가**

생성된 공개 키를 `authorized_keys`에 추가하여 호스트 머신이 SSH 연결을 허용하도록 설정합니다:

```bash
cat ~/.ssh/host_rsa.pub >> ~/.ssh/authorized_keys
```

이제 호스트 머신에 SSH 연결을 허용하는 설정이 완료되었습니다.

---

## **2. 도커 컨테이너 구성**

이제 GitHub Actions Self-Hosted Runner를 실행할 도커 컨테이너를 구성할 차례입니다. 컨테이너는 호스트 머신의 SSH 키를 사용하여 원격 접근을 수행할 수 있습니다.

### **2.1. 도커 설치 (필요한 경우)**

호스트 머신에 Docker가 설치되어 있지 않은 경우 아래 명령어로 Docker를 설치할 수 있습니다:

```bash
sudo apt-get update
sudo apt-get install -y docker.io
```

### **2.2. 도커 이미지 생성**

도커 이미지를 생성하기 위해 Dockerfile을 작성합니다. 이 Dockerfile은 Self-Hosted Runner를 설정하고, 호스트 머신에 SSH로 접근할 수 있는 환경을 구성합니다:

```Dockerfile
FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive

RUN apt-get update && \
    apt-get install -y curl jq git build-essential && \
    apt-get clean

RUN useradd -m runner

ENV RUNNER_VERSION=2.296.0
ENV RUNNER_URL=https://github.com/actions/runner/releases/download/v${RUNNER_VERSION}/actions-runner-linux-x64-${RUNNER_VERSION}.tar.gz

RUN mkdir /runner
WORKDIR /runner
RUN curl -o actions-runner-linux-x64.tar.gz -L ${RUNNER_URL} && \
    tar xzf actions-runner-linux-x64.tar.gz && \
    rm -f actions-runner-linux-x64.tar.gz

RUN ./bin/installdependencies.sh

COPY entrypoint.sh /runner/
RUN chmod +x /runner/entrypoint.sh

RUN chown -R runner:runner /runner

USER runner

ENTRYPOINT ["/runner/entrypoint.sh"]
```

### **2.3. Entry Point Script 작성**

도커 컨테이너 내에서 Runner를 실행하기 위한 스크립트를 작성합니다. 이 스크립트는 Runner를 설정하고 GitHub Actions에서 사용할 수 있도록 합니다:

```bash
#!/bin/bash

if [ -z "$GITHUB_REPOSITORY" ] || [ -z "$GITHUB_RUNNER_TOKEN" ]; then
  echo "GITHUB_REPOSITORY 및 GITHUB_RUNNER_TOKEN 환경 변수가 설정되지 않았습니다."
  exit 1
fi

./config.sh --url https://github.com/${GITHUB_REPOSITORY} --token ${GITHUB_RUNNER_TOKEN} --unattended --replace
./run.sh
```

### **2.4. 도커 이미지 빌드 및 컨테이너 실행**

위에서 작성한 Dockerfile과 entrypoint.sh 스크립트를 사용해 도커 이미지를 빌드하고, 컨테이너를 실행합니다:

```bash
cd /home/ai-server/github-runner
docker build -t github-runner .

docker run -d --name github-runner \
  -v ~/.ssh/host_rsa:/home/runner/.ssh/id_rsa \
  -v ~/.ssh/host_rsa.pub:/home/runner/.ssh/id_rsa.pub \
  -e GITHUB_REPOSITORY=레포내의 소유자이름/레포이름을 기입합니다 \
  -e GITHUB_RUNNER_TOKEN=레포내의 Settings에 들어가 액션탭에서 runner를 눌러 토큰을 가져옵니다 \
  github-runner
```

컨테이너가 정상적으로 실행되면, SSH 연결이 원활히 이루어지는지 확인합니다:

```bash
ssh-keyscan -H 192.168.0.xxxx >> /home/runner/.ssh/known_hosts
```

도커 컨테이너 내에서 SSH 설정을 확인해 볼 수 있습니다:

```bash
docker exec -it github-runner /bin/bash
ls -l /home/runner/.ssh/
```

---

## **3. 배포 스크립트 구성**

이제 배포를 자동화할 스크립트를 작성할 차례입니다. 이 스크립트는 호스트 머신에서 최신 코드를 가져와 빌드한 후, 애플리케이션을 재시작합니다.

### **3.1. 배포 스크립트 작성**

배포 작업을 수행하는 스크립트를 아래와 같이 작성합니다:

```bash
#!/bin/bash

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"
nvm use 16.14.2

cd 깃활성화된 개발 경로

CURRENT_URL=$(git remote get-url origin)
if [[ "$CURRENT_URL" == https://github.com/* ]]; then
  git remote set-url origin SSH로 접근하기 위한 깃 URL
fi

eval "$(ssh-agent -s)"
ssh-add ~/.ssh/deploy_key
#깃에 SSH로접근하기위해 사전 설정해둔 토큰 키

git pull origin main
if [ $? -ne 0 ]; then
  exit 1
fi

npm install
if [ $? -ne 0 ]; then
  exit 1
fi

npm run build
if [ $? -ne 0 ]; then
  exit 1
fi

pm2 restart ecosystem.config.js
if [ $? -ne 0 ]; then
  exit 1
fi
```

이 스크립트에 실행 권한을 부여합니다:

```bash
chmod +x /배포 스크립트 경로/deploy.sh
```

---

## **4. GitHub Actions에서 배포 트리거 설정**

GitHub Actions 워크플로우 파일을 통해 push 이벤트가 발생할 때마다 자동으로 배포 스크립트를 실행하도록 설정합니다.

### **4.1. GitHub Actions Workflow 설정**

아래와 같은 Workflow 파일을 작성하여 `main` 브랜치에 푸시될 때 자동으로 배포 작업이 실행되도록 합니다:

```yaml
name: Deploy via SSH

on:
  push:
    branches:
      - main

jobs:
  deploy:
    runs-on: self-hosted
    steps:
      - name: Checkout code
        uses: actions/checkout@v2

      - name: SSH and Execute Deploy Script
        run: |
          ssh -i /home/runner/.ssh/id_rsa -o StrictHostKeyChecking=no ${{ secrets.SSH_USER }}@${{ secrets.SSH_HOST }} 'bash /Users/sin-yunsu/Documents/GitHub/레포/deploy.sh'
        env:
          SSH_USER: ${{ secrets.SSH_USER }}
          SSH_HOST: ${{ secrets.SSH_HOST }}
```

시크릿 키의 경우는 Settings탭에서 레포단위의 환경변술를 설정 할 수 있습니다.

---

## **5. 요약**

이 포스트에서는 로컬 호스트 머신에 SSH 키를 생성하고, 도커 컨테이너 내에서 GitHub Actions Self-Hosted Runner를 실행하는 과정을 설명했습니다. 또한, 도커 컨테이너가 호스트 머신에 접근하여 배포 스크립트를 자동으로 실행할 수 있도록 설정하는 방법을 다루었습니다. 이와 같은 설정을 통해 자동화된 배포 환경을 구축할 수 있습니다.

이제 여러분의 프로젝트에도 이러한 자동화된 배포 프로세스를 도입하여 생산성을 극대화해 보세요!
