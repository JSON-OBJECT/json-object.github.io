---
layout: post
title: "Windows 11, Ubuntu on WSL 설치하기"
author: "Taehyeong Lee"
tags: [WSL, Ubuntu]
---
### 개요

-   `WSL2`는 **Windows 10/11** 운영체제에서 가상 머신 또는 듀얼 부트 없이 리눅스 운영체제를 사용 가능하게 해주는 공식 도구이다. **Microsoft**에서는 **WSL 2** 기반으로 네이티브하게 작동하는 여러 벤더의 리눅스 운영체제를 **Windows Store**에서 무료로 제공하고 있다. 이를 통해 개발자는 번거로운 과정이나 성능 저하 없이 네이티브하게 리눅스 기반의 개발 환경을 구축할 수 있다.

### 특징

-   `WSL 2` 아키텍쳐는 경량의 **VM** 위에 마이크로소프트에서 직접 제작한 리눅스 커널을 기반으로 여러 리눅스 벤더들과 공식 협력하여 **WSL distro**을 제공하기 때문에 **Windows** 생태계에서는 역대 가장 뛰어난 성능의 리눅스 환경을 제공한다.
-   위와 같은 이유로 **Cygwin**과 같이 최대한 리눅스와 비슷하게 흉내낸 것이 아닌 진짜 리눅스 쉘에서 완전히 동일하게 바이너리를 이용할 수 있다. (특히 개발자들이 환영할만한 부분으로 막강한 리눅스 생태계에서 생산성을 높이는 수많은 툴을 사용할 수 있다. 마이크로소프트에서도 공식적으로 개발자를 위해 **WSL 2**를 설계했다고 밝히고 있다.)
-   **Windows**와 **WSL distro**간의 파일 시스템에 대한 상호 접근이 가능하다. **Windows**에서는 `\\wsl$`을, **WSL distro**에서는 `/mnt` 경로로 접근할 수 있다. (어떤 경우던지 상대 파일 시스템에 접근하면 **I/O** 퍼포먼스가 떨어진다. **WSL distro** 내에서 **ext4** 파일 시스템에 접근할 때 최고의 성능을 보여준다. 그래서 내 경우, **IDE**와 소스 코드를 모두 **WSL distro**에 설치한다.)

### 사전 조건

-   현재 **Windows** 운영체제에 설치되어 있는 `Oracle VirtualBox`, `Vagrant`가 존재하다면 제거한다.
-   **Windows 10/11** 운영체제를 최신 버전으로 업데이트한다. [\[업데이트 링크\]](https://www.microsoft.com/en-us/software-download/windows10)
-   `Windows Subsystem for Linux Update`를 설치한다. [\[설치 링크\]](https://docs.microsoft.com/ko-kr/windows/wsl/install-manual#step-4---download-the-linux-kernel-update-package)

### Ubuntu on WSL 설치

-   `Ubuntu`는 데스크탑 환경을 제공하는 가장 대중적인 리눅스 운영체제이다. **Windows Store**에서 공식 제공되며 [여기](https://www.microsoft.com/store/productId/9NBLGGH4MSV6)를 클릭하여 최신 버전의 **LTS** 버전을 앱으로 설치할 수 있다. (설치 후 첫 실행시 사용할 **username**, **password**를 묻는다.) 또는 아래와 같이 콘솔에서도 설치할 수 있다.

```bash
# Ubuntu on WSL 설치
$ wsl --install -d ubuntu

# Ubuntu on WSL 버전을 WSL 2로 전환
$ wsl --set-version ubuntu 2

# Ubuntu on WSL 삭제
$ wsl --unregister Ubuntu
```

### Docker 연동

-   **Windows**에 `Docker Desktop`을 설치하면 **Ubuntu**에서도 **Docker**를 자유롭게 사용할 수 있다.

```bash
# Docker Desktop 설치 (PowerShell 관리자 권한에서 실행)
$ choco install docker-desktop -y
```

-   추가로 설치된 **Ubuntu** 콘솔 내에서 **Docker**를 이용하려면 아래 절차가 필요하다.

```bash
# Docker Desktop 실행
→ Settings
→ Resoures
→ WSL Integration
→ Enable integration with additional distros: [Ubuntu] 활성화
→ [Apply & Restart] 클릭

# IntelliJ IDEA에서 기본 Terminal 변경 설정
Settings
→ Tools
→ Terminal
→ Shell path: [ubuntu run] 입력
```

### Ubuntu on WSL 콘솔 실행

-   **Windows**에서 **Ubuntu**를 실행하는 방법은 아주 간단하다. `ubuntu` 명령어를 실행하면 된다. `Windows Terminal`이 설치되어 있다면 자동으로 인식하므로 기본 프로필로 설정해두면 편리하다.

```bash
# home 디렉토리에서 콘솔 실행
$ ubuntu

# 현재 디렉토리에서 콘솔 실행
$ ubuntu run

# 콘솔 전환 후 DNS 서버 설정
$ sudo nano /etc/wsl.conf
[network]
generateResolvConf = false

$ sudo nano /etc/resolv.conf
nameserver 8.8.8.8
nameserver 4.4.4.4

# 콘솔 전환 후 최신 업데이트 실행
$ sudo apt-get update
$ sudo apt-get upgrade -y
```

### Ubuntu on WSL 콘솔에서 Windows 애플리케이션 실행

-   거꾸로 **WSL 2** 콘솔에서 **Windows** 애플리케이션을 실행하는 것도 가능하다.

```bash
# 메모장 실행
$ powershell.exe start notepad foobar.txt

# 축약어에 등록
$ nano ~/.bash_aliases
alias cmd="powershell.exe start"

# 축약어로 실행
$ cmd notepad foobar.txt
```

### WSL 메모리 크기 변경

```bash
# WSL 종료
$ wsl --shutdown

# 메모리 크기 24GB로 변경
$ notepad "$env:USERPROFILE/.wslconfig"
[wsl2]
memory=24GB

# WSL 재시작
$ Get-Service LxssManager | Restart-Service
```

### OpenJDK 17 설치

```bash
# OpenJDK 17 설치
$ cd ~
$ wget -O- https://apt.corretto.aws/corretto.key | sudo apt-key add -
$ echo 'deb https://apt.corretto.aws stable main' | sudo tee /etc/apt/sources.list.d/corretto.list
$ sudo apt-get update
$ sudo apt-get install java-17-amazon-corretto-jdk -y

$ java --version
openjdk 17.0.7 2023-04-18 LTS

$ javac --version
javac 17.0.7
```

### Node.js 설치

-   **Ubuntu** 환경에서 **Node.js**는 `NVM`을 설치하여 편리하게 관리할 수 있다. [\[관련 링크\]](https://tecadmin.net/how-to-install-nvm-on-ubuntu-20-04/)

```bash
# NVM 설치
$ cd ~
$ curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash

# Node.js 16.13.1 설치
$ nvm install 16.13.1

$ node --version
v16.13.1

$ npm --version
8.1.2
```

### AWS CLI v2 설치

```bash
# AWS CLI v2 설치
$ cd ~
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install

$ nano ~/.bash_aliases
alias aws="/usr/local/aws-cli/v2/current/bin/aws"

$ aws --version
aws-cli/2.7.32 Python/3.9.11 Linux/5.15.57.1-microsoft-standard-WSL2 exe/x86_64.ubuntu.20 prompt/off

# AWS CLI 환경 설정
$ aws configure
AWS Access Key ID [None]: 
AWS Secret Access Key [None]: 
Default region name [None]: 
Default output format [None]:
```

### 기타 유용한 유틸리티 설치

-   아래는 **Ubuntu** 콘솔 이용시 유용한 유틸리티의 설치 예이다. [\[관련 링크\]](https://scalereal.com/devops/2020/05/15/10-cli-tools-for-developers-productivity.html)

```bash
# zsh, jq, httpie 설치
$ sudo apt-get install zsh jq httpie neofetch ranger nmap -y

# tldr 설치
$ npm install -g tldr

# autojump 설치
$ cd ~
$ git clone git://github.com/wting/autojump.git
$ cd autojump
$ ./install.py
$ source /home/jsonobject/.autojump/etc/profile.d/autojump.sh

# awslogs 설치
$ sudo apt install python3-pip -y
$ pip install awslogs
```

### 참고 글

-   [Awesome WSL - Windows Subsystem for Linux](https://github.com/sirredbeard/Awesome-WSL)
