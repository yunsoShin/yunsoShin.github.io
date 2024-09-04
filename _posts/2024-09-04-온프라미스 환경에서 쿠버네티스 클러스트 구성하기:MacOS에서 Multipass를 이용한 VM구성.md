해당 블로그 글에서 Docker 설치 및 테스트 과정을 추가하여 수정해보겠습니다.

---

layout: post  
title: "온프라미스 환경에서 쿠버네티스 클러스터 구성하기: macOS에서 Multipass를 이용한 우분투 VM 설정"  
date: 2024-09-04  
categories: kubernetes  
tags: [kubernetes, VM, Docker]

---

### 온프라미스 환경에서 쿠버네티스 클러스터 구성하기: macOS에서 Multipass를 이용한 우분투 VM 설정

온프라미스 환경에서 쿠버네티스(Kubernetes) 클러스터를 구성하는 것은 클라우드 환경에서의 설정과는 다른 경험을 제공합니다. 특히, macOS 환경에서 Multipass를 활용하여 쿠버네티스 클러스터를 설정하는 과정은 실제 프로덕션 환경과 유사한 환경을 구현하는 데 큰 도움이 됩니다. 이번 글에서는 Multipass를 이용하여 Ubuntu 가상 머신(VM)을 설정하고, Docker를 설치한 후, 쿠버네티스를 설치하는 방법을 소개합니다.

#### 1. Multipass 설치

먼저, macOS에 Multipass를 설치해야 합니다. Multipass는 경량의 VM을 쉽게 생성하고 관리할 수 있도록 도와주는 도구입니다. 홈브루(Homebrew)를 사용하여 Multipass를 설치할 수 있습니다.

```bash
brew install --cask multipass
```

위 명령어를 실행하면, macOS에 Multipass가 설치됩니다. 설치가 완료되면 터미널에서 `multipass` 명령어를 사용할 수 있게 됩니다.

#### 2. Ubuntu 가상 머신 생성

쿠버네티스를 설치할 환경으로 사용할 Ubuntu VM을 생성합니다. CPU, 메모리, 디스크 용량을 지정하여 VM을 생성할 수 있습니다. 아래 명령어를 사용하여 이름이 `ubuntu-k8s`인 VM을 생성합니다.

```bash
multipass launch --name ubuntu-k8s --cpus 2 --memory 4G --disk 20G
```

- `--name ubuntu-k8s`: 생성할 VM의 이름을 지정합니다.
- `--cpus 2`: 2개의 CPU 코어를 할당합니다.
- `--memory 4G`: 4GB의 메모리를 할당합니다.
- `--disk 20G`: 20GB의 디스크 용량을 할당합니다.

명령어를 실행하면 Ubuntu가 설치된 VM이 생성되며, 이 VM은 쿠버네티스를 구성할 기본 환경이 됩니다.

#### 3. Ubuntu VM에 접속하기

VM이 생성된 후, 아래 명령어를 통해 `ubuntu-k8s` VM에 접속합니다.

```bash
multipass shell ubuntu-k8s
```

이제 Ubuntu 환경에 접속한 상태로, 여기서부터는 Docker 설치와 관련된 작업을 진행할 수 있습니다.

#### 4. Docker 설치 및 테스트

쿠버네티스를 설치하기 전에, 먼저 Docker를 설치하여 환경이 올바르게 구성되었는지 확인해야 합니다. Docker는 쿠버네티스에서 컨테이너를 관리하는 데 필수적인 도구입니다.

```bash
# 기존 도커 삭제
for pkg in docker.io docker-doc docker-compose docker-compose-v2 podman-docker containerd runc; do sudo apt-get remove $pkg; done
```

```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update
```

혹은 최신버전의 도커와 우분투를 다운로드하여 실행한다는 가정하에는 아래와 같은 명령어를 이용할 수 있습니다

```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

#### 테스트

```bash
sudo docker run hello-world
```

위 명령어를 실행하면 Docker가 올바르게 설치되었는지 확인할 수 있습니다. `hello-world` 컨테이너가 정상적으로 실행되면, Docker가 성공적으로 설치된 것입니다.

#### 5. 쿠버네티스 설치 및 설정

Docker가 정상적으로 설치된 후, 이제 쿠버네티스를 설치할 차례입니다. 먼저 필수 패키지들을 설치하고, 쿠버네티스를 설치하기 위한 설정을 진행합니다.

```bash
# 쿠버네티스 설치를 위한 필수 패키지 설치
sudo apt-get install -y apt-transport-https ca-certificates curl

# 쿠버네티스 apt 저장소 추가
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
sudo apt-add-repository "deb http://apt.kubernetes.io/ kubernetes-xenial main"

# 쿠버네티스 설치
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl

# kubelet 자동 시작 설정
sudo systemctl enable kubelet
sudo systemctl start kubelet
```

#### 6. 쿠버네티스 클러스터 초기화

이제 클러스터를 초기화합니다. 아래 명령어를 통해 마스터 노드를 초기화합니다.

```bash
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

초기화가 완료되면, 클러스터에 대한 설정을 사용자 환경으로 복사합니다.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

#### 7. 네트워크 플러그인 설치

쿠버네티스에서 Pod들이 통신할 수 있도록 네트워크 플러그인을 설치해야 합니다. 여기서는 `Weave Net`을 사용합니다.

```bash
kubectl apply -f https://git.io/weave-kube-1.6
```

#### 8. 클러스터 상태 확인

설치가 완료된 후, 다음 명령어를 통해 쿠버네티스 클러스터의 상태를 확인할 수 있습니다.

```bash
kubectl get nodes
```

모든 노드가 `Ready` 상태로 표시되면, 쿠버네티스 클러스터가 정상적으로 설정된 것입니다.

이번 글에서는 macOS에서 Multipass를 이용해 Ubuntu VM을 생성하고, Docker를 설치한 후, 쿠버네티스를 설치하는 과정을 다루었습니다. 이를 통해 온프라미스 환경에서의 쿠버네티스 설정 과정을 체험해볼 수 있습니다. 이 과정을 바탕으로 다양한 쿠버네티스 구성 및 테스트를 진행해볼 수 있을 것입니다.

---
