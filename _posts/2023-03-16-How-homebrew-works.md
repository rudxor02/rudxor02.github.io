---
layout: post
type: tags
title: "[Homebrew] How Homebrew Works"
date: 2023-03-16 23:30:55 +0900
comments: true
toc: true
tags: homebrew
---

_회사 다니면서 mac을 처음 써봐서 homebrew를 설치하다가, 문득 os마다 하나씩은 존재하는 패키지 관리자는 어떻게 동작하는지 궁금했던 게 기억나서 회사에서 동일 주제로 세미나를 했던 걸 정리하고자 한다._

## Homebrew란

mac os에서의 주된 패키지(docker, node, yarn, pulumi 같은 것들) 관리자이다.

## How homebrew works

homebrew는 온라인 저장소(주로 github)에 저장된 패키지를 받아와 로컬에서 binary file로 컴파일해서 사용자가 패키지를 터미널에서 cli로 사용할 수 있게 연결해 주는 역할을 한다.

```bash
$ brew install pulumi

$ where pulumi
#/opt/homebrew/bin/pulumi

$ ls /opt/homebrew/bin
#pulumi, node, yarn, ...

$ brew install docker

$ where docker
#/usr/local/bin

$ ls /usr/local/bin
#docker, ... (여기에는 homebrew 뿐만 아니라 다른 binary 파일들도 존재)
```

## Terminology

_양조과정에서 모티브를 따온듯 하다._

- Formula _(양조방식)_
  - package definition을 말한다. 즉 package가 어떻게 구성되는 지가 Formula에 나와있다.
  - homebrew-core 레포에 저장된 패키지 설치를 해주는 ruby 파일을 Formula라고 칭하며, ruby 파일의 이름은 패키지 이름과 같다. ruby class 이름이기도 하다.
    - `homebrew-core/Formula/<package-name>.rb`
    - reference는 [여기](https://rubydoc.brew.sh/Formula.html)서 확인할 수 있다.
  - `brew install <package-name>` 을 하게 되면 Formula를 받아와서 해당 Formula에 적힌 내용대로 패키지 설치를 진행한다.
  - [여기](https://formulae.brew.sh/formula/)서 homebrew에서 공식적으로 지원하는 Formula 목록들을 찾을 수 있다.
  - ex) npm, yarn, pulumi
- Keg _(작은 통)_
  - Installation prefix of formula
  - 쉽게 말해 패키지의 버전이다.
- Cellar _(지하실)_
  - Keg가 설치되는 로컬 디렉토리
- Cask _(통)_
  - Formula와 비슷한 개념으로, package definition이다. Formula와 다른 점은 mac os 어플리케이션 형식으로 설치, 지원이 된다 _(launchpad에서 찾을 수 있다)_
  - homebrew-cask 레포에 저장된 패키지 설치를 해주는 ruby 파일 이름은 패키지 이름과 같다.
    - `homebrew-cask/Casks/<package-name>.rb`
  - `brew install --cask <package-name>` 을 통해서 설치할 수 있다.
  - ex) docker
- Tap _(맥주 탭)_
  - package’s git repository
- Bottle _(맥주병)_

  - pre-compiled binary file = pre-built Keg
  - bottle이 필요없는 패키지도 있다.
  - os별로 존재한다.
    ![bottle](/assets/images/post/2023-03-16-How-homebrew-works-20230402231139.png)

  - _왜 필요한지 추측을 해보자면 컴파일 하는 데_
    - _시간이 오래 걸려서?_
    - _컴파일 도구가 무거워서 그냥 배포자가 컴파일하는 게 여러모로 편해서?_
    - _공유하기 쉬워서?_

## How homebrew works (with terminology)

- `brew install <package-name>`을 실행하면…

  - homebrew resolves Formula in homebrew-core repository with `<package-name>`
  - homebrew는 Formula를 따라서 Cellar에 Keg를 설치한다. 위치는 `/opt/homebrew/Cellar/<package-name>/<package-version>/` 이며, package binary file이 이곳에 위치한다.
  - homebrew symlinks `/opt/homebrew/bin/<package-name>` into Keg
    ![symlink](/assets/images/post/2023-03-16-How-homebrew-works-20230402231656.png)

- package를 실행하면…
  - $PATH (includes `/opt/homebrew/bin`) → `/opt/homebrew/bin/<package-name>` → `/opt/homebrew/Cellar/<package-name>/<package-version>/<path-to-package-binary-file>` 을 따라서 package binary file이 실행된다.
- homebrew에서 지원하는 패키지들은 패키지 개발자가 homebrew-core 레포에 pull request를 해서 패키지 등록 과정을 거친 상태이다.
  - [여기](https://github.com/Homebrew/homebrew-core/pull/30711)에 pulumi 라는 패키지를 homebrew에 새로 등록하기 위한 pull request가 있다.
  - version update도 마찬가지이다. ex) [pulumi : version update to 3.58.0](https://github.com/Homebrew/homebrew-core/pull/125761)
- 꼭 homebrew-core 레포에 pull request 과정을 거친 공식적인 패키지를 사용 안하더라도 `brew tap` 을 통해서 custom package를 만들고 배포할 수 있다. [여기](https://jldlaughlin.medium.com/how-does-homebrew-work-starring-rust-94ae5aa24552) 블로그에 튜토리얼 느낌으로 잘 나와있음.

## Reference

- [homebrew doc](https://docs.brew.sh/)
- [ruby formula interface doc](https://rubydoc.brew.sh/Formula.html)
- [brew tap turotial (블로그)](https://jldlaughlin.medium.com/how-does-homebrew-work-starring-rust-94ae5aa24552)
