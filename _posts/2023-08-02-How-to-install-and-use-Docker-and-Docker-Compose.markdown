---
layout: post
title: "Docker, Docker Compose 설치 및 사용법 정리"
author: "Taehyeong Lee"
tags: [Docker]
---
### 개요
  * 이번 글에서는 **CentOS 7/8** 및 **Amazon Linux 2023**에서 `docker`, `docker-compose`를 설치하고 사용하는 방법을 정리했다.

### Docker 설치
  * **docker**를 사용하면 운영체제와 독립적인 이미지를 인스턴스로 올려 컨테이너로 작동시킬 수 있다. 아키텍쳐의 구성 및 확정, 배포 방법이 비약적으로 간소화된다. **CentOS 7/8** 및 **Amazon Linux 2023**에서의 설치 및 실행 방법은 아래와 같다.

```bash
# CentOS 7/8에서 Docker 설치
$ curl -fsSL https://get.docker.com/ | sh

# Amazon Linux 2023에서 Docker 설치
$ sudo dnf update
$ sudo dnf install docker -y

# Docker 서비스 시작
$ sudo systemctl start docker

# Docker 서비스 작동 상태 확인
$ sudo systemctl status docker

# Docker 서비스를 운영체제 부팅시 자동 시작하도록 설정
$ sudo systemctl enable docker

# docker 명령어를 sudo 없이 사용하기 위해 계정을 docker 그룹에 소속 (계정 재접속 필요)
$ sudo usermod -aG docker $USER
$ newgrp docker

# 설치된 Docker 버전 확인
$ docker --version
Docker version 20.10.25, build b82b9f3

# hello-world 컨테이너 실행 확인
$ docker run hello-world
Hello from Docker!
This message shows that your installation appears to be working correctly.
```

### Docker Compose 설치
  * **docker**의 단점은 애플리케이션마다 각각의 컨테이너로 독립적으로 실행된다는 것이다. 엔터프라이즈 레벨의 아키텍쳐에서는 여러 애플리케이션이 함께 실행되어 영향을 주고 받는 것이 흔하다. 이 것을 가능하게 하기 위해 **docker-compose**가 제공된다. 설치 방법은 아래와 같다.

```bash
# Docker Compose 설치
$ sudo curl -L https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m) -o /usr/local/bin/docker-compose

# Docker Compose 실행 권한 부여
$ sudo chmod +x /usr/local/bin/docker-compose

# 설치된 Docker Compose 실행 확인
$ docker-compose --version
Docker Compose version v2.20.2
```

### Docker 원격 저장소 로그인
  * 퍼블릭이 아닌 프라이빗 원격 저장소의 이미지를 풀링하고 푸시하려면 `docker login` 명령이 사전에 실행되어야 한다. [\[관련 링크\]](https://github.com/docker/for-win/issues/12355#issuecomment-642915975)

```bash
# 기존 저장된 인증 정보를 삭제
$ rm ~/.docker/config.json

# 기존 로그인되어 있다면 로그아웃
$ docker logout

# [방법 1] DockerHub 사용자는 아래와 같이 로그인
$ docker login

# [방법 2] Amazon ECR 사용자는 아래와 같이 로그인 ({region}, {ecr-uri} 파라메터를 자신의 환경에 맞게 수정)
# Login Succeeded 가 출력되면 로그인 성공
$ docker login --username AWS -p $(aws ecr get-login-password --region {region}) {ecr-uri}/
```

### Dockerfile 작성
  * 이미 제공되거나 만들어진 이미지를 그대로 사용하기도 하지만, 대부분의 배포 환경에서는 베이스 이미지를 기반으로 직접 `Dockerfile`을 작성하여 이미지를 빌드하고 컨테이너를 실행한다.

```bash
$ nano Dockerfile
FROM docker.io/jhipster/jhipster-registry:v7.1.0
EXPOSE 8761
USER root
RUN apt update -y
RUN apt install software-properties-common -y
RUN add-apt-repository ppa:deadsnakes/ppa -y
RUN apt install python3.9 -y
COPY entrypoint.sh /usr/local/bin/
ENTRYPOINT ["sh", "./entrypoint.sh"]
```

  * `EXPOSE {port}`는 해당 이미지가 리스닝해야할 포트를 사용자에게 알려주는 식별자 역할이다. 실제 물리적인 포트 개방에는 관여하지 않는다.
  * 마지막 줄은 반드시 `CMD` 또는 `ENTRYPOINT`가 작성되어야 한다. (**RUN**은 `build` 실행시 이미지에 반영되고, **CMD**, **ENTRYPOINT**는 `run` 실행시 실행된다.)
  * `entrypoint.sh`는 반드시 리눅스 방식의 **CR**로 저장되어야 한다. 아닐 경우, **dos2unix** 명령어로 변환할 수 있다. [\[관련 링크\]](https://jsonobject.tistory.com/590)

### 이미지 빌드

```bash
# 이미지 빌드
$ docker build -t {image-name}:{tag-name} .
```

### 컨테이너 실행
  * 빌드된 이미지를 기반으로 `docekr run` 명령으로 컨테이너를 실행할 수 있다. 아래와 같이 다양한 옵션을 사용할 수 있다.

```bash
# 이미지 이름으로 컨테이너 실행
# [1. 로컬 2. Docker Hub]의 순서로 컨테이너를 찾아 실행
$ docker run -d {image-name}:{tag-name}
{container-id}

# -p, 로컬 포트에 컨테이너 포트를 맵핑
$ docker run --rm -it -p {local-port}:{container-port} {image-name}:{tag-name}

# -v, 로컬 파일을 컨테이너 파일에 주입
$ docker run --rm -it -v {local-full-path}:{container-full-path} {image-name}:{tag-name}

# --entrypoint, 컨테이너 실행과 함께 bash 쉘 접속
$ docker run --rm -it --entrypoint bash {image-name}:{tag-name}

# --user, 컨테이너 실행과 함께 root 계정으로 bash 쉘 접속
$ docker run --rm -it --user root --entrypoint bash {image-name}:{tag-name}
```

  * `-d` 옵션은 컨테이너를 데몬으로 실행시킨다. 실행과 함께 실행된 컨테이너의 ID를 출력한다.

### 실행 중인 컨테이너 콘솔 접속
  * 현재 실행 중인 컨테이너의 쉘에 접속할 수 있다.

```bash
# 현재 실행 중인 컨테이너 ID 목록 조회
$ docker ps

# 현재 실행 중인 특정 컨테이너에 bash 쉘 접속
$ docker exec -it {container-id} bash

# whoami
root

# pwd
/
```

### 원격 리파지터리에서 이미지 푸시
  * 로컬에 생성된 이미지를 원격 리파지터리에 푸시하는 방법은 아래와 같다.

```bash
# 로컬에 존재하는 이미지에 대해 원격 리파지터리 푸시를 위한 태그 생성
$ docker tag foo/bar:latest xxxxx.dkr.ecr.ap-northeast-2.amazonaws.com/foo/bar

# 원격 리파지터리에 이미지를 푸시
$ docker push xxxxx.dkr.ecr.ap-northeast-2.amazonaws.com/foo/bar
```

### 로컬의 모든 이미지, 컨테이너 일괄 삭제
  * 로컬 환경에서 개발을 하다보면 더이상 사용하지 않는 이미지, 컨테이너가 계속해서 쌓이게 된다. 아래 명령을 실행하면 한 번에 일괄 삭제가 가능하다.

```bash
# 로컬에 생성 또는 실행 중인 모든 이미지, 컨테이너 제거
$ docker system prune -a
```

### docker-compose.yml 작성
  * `Docker Compose`를 이용하면 복수개의 컨테이너를 하나의 단위로 오케스트레이션할 수 있다. 아래는 **MySQL**과 **Redis**를 하나로 묶은 예이다.

```bash
$ nano docker-compose.yml
version: '3'
services:
  mysql:
    image: "public.ecr.aws/ubuntu/mysql:latest"
    environment:
      - MYSQL_USER=username
      - MYSQL_PASSWORD=password
      - MYSQL_ALLOW_EMPTY_PASSWORD=yes
      - MYSQL_DATABASE=database
      - TZ=UTC
    ports:
      - "3306:3306"
    cap_add:
      - SYS_ADMIN
    ulimits:
      nofile: 65535
    restart: always

  redis:
    image: "public.ecr.aws/ubuntu/redis:latest"
    environment:
      - ALLOW_EMPTY_PASSWORD=yes
      - TZ=UTC
    ports:
      - "6379:6379"
    restart: always
```

  * `services.{service-name}.cap_add` 옵션으로 해당 컨테이너에 **root** 유저 권한을 선택적으로 부여할 수 있다. 일반적으로 대부분의 이미지는 **root** 유저 권한으로 실행하는 것을 지양하거나 엄격하게 금지하고 있다. 이 경우 위 옵션을 통해 필요한 권한을 제한적으로 부여하여 일반 유저 계정으로도 이미지를 실행할 수 있다.
  * `services.{service-name}.ulimits` 옵션으로 해당 컨테이너의 **ulimit**를 설정할 수 있다. 위의 경우 `ulimit -n 65535` 명령과 동일한 기능의 **nofile: 65535**을 설정했다.
  * `services.{service-name}.restart: always` 옵션은 물리적인 호스트 인스턴스가 재부팅한 후에도 자동으로 해당 서비스를 실행할 수 있게 보장한다. (전제 조건으로 `sudo systemctl enable docker` 명령으로 **Docker** 서비스가 실행 중에 있고, 현재 `docker-compose up -d` 명령으로 서비스가 실행 중이어야 한다.) [[관련 링크]](https://stackoverflow.com/a/53569049/17742933)

### Docker Compose 기동

```bash
# Docker Compose 실행
$ docker-compose down && docker-compose build --pull && docker-compose up -d

# Docker Compose 로그 조회
$ docker-compose logs -f
```

### 참고 글
  * [How To Install and Use Docker on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-on-centos-7)
  * [How To Install and Use Docker Compose on CentOS 7](https://www.digitalocean.com/community/tutorials/how-to-install-and-use-docker-compose-on-centos-7)
  * [Synk - 10 best practices to build a Java container with Docker](https://snyk.io/blog/best-practices-to-build-java-containers-with-docker/)
  * [Baeldung - How To Configure Java Heap Size Inside a Docker Container](https://www.baeldung.com/ops/docker-jvm-heap-size)
  * [DigitalOcean - How To Build a Node.js Application with Docker](https://www.digitalocean.com/community/tutorials/how-to-build-a-node-js-application-with-docker)
  * [DigitalOcean - Building Optimized Containers for Kubernetes](https://www.digitalocean.com/community/tutorials/building-optimized-containers-for-kubernetes)
