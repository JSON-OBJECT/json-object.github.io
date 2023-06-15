---
layout: post
title: "GitHub Pages와 Jekyll을 이용하여 무료 블로그 구축하기"
author: "Taehyeong Lee"
tags: [GitHub, Jekyll]
---
### 개요
  * **GitHub**은 `GitHub Pages`라는 이름의 무료 정적 웹사이트 호스팅 서비스를 제공한다. 유저 또는 기관 당 1개의 **GitHub Pages** 리포를 생성할 수 있고 이를 통해 별도의 호스팅 플랫폼을 운영하지 않고도 블로그 등의 텍스트 정보 중심의 사이트 구축 및 운영이 가능하다. 이번 글에서는 **GitHub Pages** 리포를 생성한 후 `Jekyll`로 마크다운 구문의 정적 웹사이트를 구축하는 방법을 정리했다.

### GitHub Pages 리포 생성
  * **GitHub**에 `{username}.github.io`라는 이름의 리포를 생성하면 자신의 **GitHub Pages** 생성이 완료된다. (주의할 점은 반드시 자신의 **username**을 포함하여 전체 이름이 일치해야 한다.) 이제 `https://{username}.github.io/`로 접속이 가능해진다.
  * 내 소유의 특정 리포를 추가로 호스팅할 수도 있다. 퍼블릭으로 공개된 특정 리포를 `gh-pages` 브랜치로 푸시하면 자동으로 빌드되어 `https://{username}.github.io/{repo-name}` 주소로 제공된다. [[관련 링크]](https://stackoverflow.com/a/40913384/17742933)

### Jekyll 설치
  * `Jekyll`을 운영체제에 설치해야 한다. 아래는 **Ubuntu**에서 **Jekyll**을 설치하는 예이다.

```bash
$ sudo apt-get install ruby-full build-essential zlib1g-dev
 
$ echo '# Install Ruby Gems to ~/gems' >> ~/.bashrc
$ echo 'export GEM_HOME="$HOME/gems"' >> ~/.bashrc
$ echo 'export PATH="$HOME/gems/bin:$PATH"' >> ~/.bashrc
$ source ~/.bashrc
 
$ gem install bundler

# Jekyll 설치
$ gem install jekyll --version="~> 4.2.0"

# kramdown 설치
$ gem install kramdown rouge
```

### Jekyll 페이지 생성
  * 이제 비어 있는 리포에 생명을 불어넣을 차례이다. 아래와 같이 `Jekyll`을 생성한다.

```bash
# GitHub Pages 리포 클로닝
$ git clone git@github.com:{username}/{username}.github.io.git
$ cd {username}.github.io.git

# Jekyll 페이지 생성
$ jekyll new --skip-bundle .

# GitHub Pages 플러그인 활성화
$ nano Gemfile
# 1. 주석 처리
# gem "jekyll", "~> 4.2.2"
# 2. 내용 추가
gem "github-pages", "~> 228", group: :jekyll_plugins

$ Jekyll 페이지 빌드
$ bundle install
```

  * 빌드된 내용을 **master** 브랜치에 머지하면 **GitHub Pages**에 의해 자동으로 빌드되어 수 분 뒤에 내용이 반영된다.

### _config.yml
  * `_config.yml` 파일은 환경 설정 및 플러그인 설정을 담고 있다. 이 파일을 아래와 같이 수정할 수 있다. 

```bash
title: JSON-OBJECT Software Engineering Blog
email: jsonobject@gmail.com
description: >-
  Professional Senior Backend Engineer. Specializing in high volume traffic and distributed processing with Kotlin and Spring Boot as core technologies.
github_username: JSON-OBJECT

# Hacker 테마 적용, 기본값은 minma
theme: jekyll-theme-hacker

plugins:
  - jekyll-feed

# kramdown 구문 활성화
markdown: kramdown
kramdown:
  input: GFM
  syntax_highlighter: rouge

# Pagination 활성화
paginate: 5
paginate_path: "/posts/:num/"
permalink: "/:title.html"
```

### index.html
  * 루트 디렉토리에 관문 역할을 하는 `index.html` 파일을 아래와 같이 작성한다.

````
```
---
layout: default
---
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="utf-8">
    <title>JSON-OBJECT Software Engineering Blog</title>
  </head>
  <body>

    <h1>Recent Posts</h1>
      <ul>
        {% for post in paginator.posts %}
        <li>
          <h2><a href="{{ post.url | prepend: site.baseurl | replace: '//', '/' }}">{{ post.title }}</a></h2>
          <time datetime="{{ post.date | date_to_xmlschema }}">{{ post.date | date_to_string }}</time>
          <p>{{ post.content | strip_html | truncatewords:50 }}</p>
        </li>
        {% endfor %}
      </ul>

<div class="pagination">
  {% if paginator.previous_page %}
    <a href="{{ paginator.previous_page_path }}" class="previous">
      Previous
    </a>
  {% else %}
    <span class="previous">Previous</span>
  {% endif %}
  <span class="page_number ">
    Page: {{ paginator.page }} of {{ paginator.total_pages }}
  </span>
  {% if paginator.next_page %}
    <a href="{{ paginator.next_page_path }}" class="next">Next</a>
  {% else %}
    <span class="next ">Next</span>
  {% endif %}
</div>

  </body>
</html>
```
````

### 참고 글
  * [Github 블로그 검색엔진에 등록하기 (구글/네이버/다음)](https://yenarue.github.io/tip/2020/04/30/Search-SEO/)
