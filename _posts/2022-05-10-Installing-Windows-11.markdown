---
layout: post
title: "Windows 11, 첫 설치 방법 정리"
author: "Taehyeong Lee"
tags: [Windows]
---
### Windows 11 설치

-   **Windows 11** 설치를 위한 첫 시작은 **Windows 11 설치** 프로그램을 다운로드하는 것이다. [여기](https://www.microsoft.com/ko-kr/software-download/windows11)를 클릭하여 다운로드한다. 준비물로는 사용하지 않는 **8GB** 이상 크기의 **USB** 미디어가 필요하다. (**제품 키** 등록은 설치가 완료된 후 진행한다.)

```bash
다운로드한 MediaCreationToolW11.exe 을 실행
→ 사용할 미디어 선택: [USB 플래시 드라이브] 선택
→ (재시작 후 F2 키를 눌러 바이오스 진입하여 앞서 설치한 USB로 부팅)
→ 설치 유형을 선택하세요: [Windows만 설치(고급)] 선택
→ Windows를 설치할 위치를 지정하세요: (Windows 11을 설치할 드라이브 파티션 선택)
```

-   설치시 대상이 되는 드라이브 파티션을 지정할 수 있는데, 이 말은 한 **PC**에서 여러 개의 **Windows 11**을 운용할 수 있다는 것을 의미한다. 2개 이상의 드라이브 파티션에 서로 다른 **Windows 11**이 설치되면 자동으로 부팅 시점에 어떤 파티션으로 부팅할지를 묻는 멀티 부트 화면이 나타나게 된다. (내 경우, 개인과 회사 용도로 구분하여 드라이브 파티션을 분리하고 **Windows 11**를 설치하는 것을 선호한다.)

### Windows 11 버전 업그레이드

-   이미 **Windows 11 Home** 버전으로 정품 인증된 사용자는 **Pro** 버전으로 업그레이드하기 위해 2개의 **제품 키**가 필요하다. 하나는 **Home**에서 **Pro**로 업그레이드하기 위한 것이고, 다른 하나는 **Pro** 정품 인증을 위한 **제품 키**이다.

```bash
# 1. 업그레이드 전용 제품 키 등록
[시작]
→ [제품 키 변경]
→ [Windows 버전 업그레이드] 클릭
→ 제품 키 변경: [변경] 클릭
→ 제품 키: XXXXX-XXXXX-XXXXX-XXXXX-XXXXX (자신이 구매한 Pro 업그레이드 전용 제품 키 입력) → [다음] 클릭 → [시작] 클릭

# 2. Windows 재시작 후 2. Pro 제품 키 등록
(위와 동일한 과정으로 자신이 구매한 Pro 제품 키 입력)
```

### 유용한 유틸리티 설치

-   `Chocolatey`와 여러 유용한 유틸리티를 설치한다. [\[관련 링크\]](https://jsonobject.tistory.com/526)
-   `NexusFile`, `반디집`, `Notepad2`, `윈도우클리너`, `구라 제거기`, `모두의 프린터`를 설치한다.
-   `Windows Termianl Preview`를 설치한다. [\[관련 링크\]](https://jsonobject.tistory.com/568)
-   `Ubuntu on Windows`, `X410`, `IntelliJ IDEA`를 설치한다. [\[관련 링크 1\]](https://jsonobject.tistory.com/9) [\[관련 링크 2\]](https://jsonobject.tistory.com/596)
