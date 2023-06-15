---
layout: post
title: "Windows 11, Ubuntu on WSL에 X410, IntelliJ IDEA 연동하기"
author: "Taehyeong Lee"
tags: [WSL, Ubuntu]
---
### 개요

-   `WSL2`의 등장으로 **Windows** 운영체제에서도 네이티브 환경의 리눅스 사용이 가능해졌다. `X410`을 이용하면 **WSL2 distro**에서 실행하는 모든 **GUI** 애플리케이션 또한 실행할 수 있어 강력히 추천한다. 이번 글에서는 **X410**과 `IntelliJ IDEA`를 설치하고 설정하는 방법을 소개하고자 한다.

### 사전 조건

-   **Windows 11** 운영체제에 `WSL2`가 활성화되어 있고, `Ubuntu on WSL`이 설치되어 있어야 한다. [\[관련 링크\]](https://jsonobject.tistory.com/9)

### X410 설치 및 설정

-   `X410`은 **Windows Store**에서 구매할 수 있다. [\[설치 링크\]](https://www.microsoft.com/en-us/p/x410/9nlp712zmn9q) (마침 할인 기간으로 부가세 포함 18,000원에 구매했다.)
-   구매가 완료되면 공식 홈페이지의 안내 글에 따라 설정을 완료한다. [\[안내 글 링크\]](https://x410.dev/cookbook/wsl/using-x410-with-wsl2/)

```bash
# ~/.bashrc 파일에 X410 설정 추가
$ nano ~/.bashrc
export DISPLAY=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2; exit;}'):0.0
export DISPLAY=$(ip route | grep default | awk '{print $3; exit;}'):0.0
```

### 파일 모니터링 성능 최적화

-   규모가 큰 프로젝트일수록 **IntelliJ IDEA**가 모니터링해야할 파일의 개수도 늘어난다. 아래와 같이 모니터링할 최대 파일 개수를 상향할 수 있다. [\[관련 링크\]](https://doc.networknt.com/tool/idea/settings/)

```bash
$ sudo apt-get install inotify-tools -y
$ sudo sh -c "echo \"fs.inotify.max_user_watches = 524288\" >> /etc/sysctl.conf"
$ sudo sysctl -p --system
fs.inotify.max_user_watches = 524288
```

### JetBrains Toolbox 설치
  * **IntelliJ IDEA**를 직접 설치하는 것보다는 `JetBrains Toolbox`를 설치하는 것이 여러모로 편리하다.

```bash
# JetBrains Toolbox 다운로드 및 압축 해제
$ cd ~
$ sudo apt-get install libfuse2
$ wget https://download.jetbrains.com/toolbox/jetbrains-toolbox-2.1.3.18901.tar.gz
$ sudo rm -rf /opt/toolbox
$ sudo mkdir /opt/toolbox
$ sudo tar -xzvf jetbrains-toolbox-2.1.3.18901.tar.gz -C /opt/toolbox
$ sudo ln -sf /opt/toolbox/jetbrains-toolbox-2.1.3.18901 /opt/jetbrains-toolbox

# toolbox, idea 별명 등록
$ nano ~/.bash_aliases
alias toolbox="/opt/jetbrains-toolbox/jetbrains-toolbox > /dev/null 2>&1 &"
alias idea="~/.local/share/JetBrains/Toolbox/apps/intellij-idea-ultimate/bin/idea.sh > /dev/null 2>&1 &"
alias idea.="~/.local/share/JetBrains/Toolbox/apps/intellij-idea-ultimate/bin/idea.sh . > /dev/null 2>&1 &"
```

  * **JetBrains Toolbox** 설치가 완료되면 실행하여 **IntelliJ IDEA**를 설치한다.

### IntelliJ IDEA 직접 설치
  * 아래는 **JetBrains Toolbox**을 통하지 않고 **IntelliJ IDEA**를 직접 설치하는 방법이다.

```bash
# IntelliJ IDEA 다운로드 및 압축 해제
$ cd ~
$ wget https://download.jetbrains.com/idea/ideaIU-2023.3.tar.gz
$ sudo rm -rf /opt/intellij /opt/idea
$ sudo mkdir /opt/intellij
$ sudo tar -xzvf ideaIU-2023.3.tar.gz -C /opt/intellij

# /opt/idea 심볼릭 링크 생성
$ sudo ln -sf /opt/intellij/idea-IU-233.11799.241/ /opt/idea

# idea 별명 등록
$ nano ~/.bash_aliases
alias idea="/opt/idea/bin/idea.sh > /dev/null 2>&1 &"
alias idea.="/opt/idea/bin/idea.sh . > /dev/null 2>&1 &"
```

### IntelliJ IDEA 실행

```bash
# IntelliJ IDEA 실행
$ idea

# 현재 프로젝트 디렉토리에서 IntelliJ IDEA 실행
$ idea.
```

### IntelliJ IDEA 기본 브라우저 설정

```bash
Settings → Tools → Web Browsers and Preview
- Default Browser: [Custom path]
- Custom Path: /mnt/c/Program Files/Google/Chrome/Application/chrome.exe (입력)
```

### IntelliJ IDEA 설정 초기화

```bash
$ rm -rf ~/.config/JetBrains ~/.cache/JetBrains ~/.local/share/JetBrains
```

### fcitx 한글 입력기 설치

-   **Ubuntu on WSL**은 기본 설치시 **GUI**에서 한글 입력이 지원되지 않는다. 아래와 같이 `fcitx` 한글 입력기를 설치할 수 있다.

```bash
# fcitx 한글 입력기 설치
$ sudo apt-get install fcitx fcitx-hangul fonts-noto-cjk fonts-noto-color-emoji fonts-nanum* dbus-x11 -y

# fcitx를 기본 한글 입력기로 설정
$ im-config

$ sudo sh -c "dbus-uuidgen >> /var/lib/dbus/machine-id"

# Input Method에 Hangul을 추가
$ fcitx-config-gtk3

$ nano ~/.bashrc
export QT_IM_MODULE=fcitx
export GTK_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx
export DefaultIMModule=fcitx
fcitx-autostart &>/dev/null
```

### 참고 글

-   [테크폭스 - WSL2에서 X410 사용](https://techfox.tistory.com/39)
-   [Lightning speed development on Windows 10 with WSL 2, Docker, Terminal and Jetbrains IntelliJ/PyCharm/PHPStorm](https://www.pimwiddershoven.nl/entry/lightning-speed-development-on-windows-10-with-wsl-2-docker-terminal-and-jetbrains-intellij-pycharm-phpstorm)
-   [Setup Input Method for WSL](https://patrickwu.space/2019/10/28/wsl-fcitx-setup/)
-   [WSL2에서 한글 입력 사용하기](https://sigmafelix.wordpress.com/2020/08/17/wsl2%EC%97%90%EC%84%9C-%ED%95%9C%EA%B8%80-%EC%9E%85%EB%A0%A5-%EC%82%AC%EC%9A%A9%ED%95%98%EA%B8%B0/)
