---
layout: post
type: tags
title: "How pulumi works"
date: 2023-03-23 23:30:55 +0900
comments: true
toc: true
tags: pulumi infrastructure
---

*[여기](https://www.pulumi.com/docs/intro/concepts/how-pulumi-works/#how-does-pulumi-work) 공식 문서 내용이 잘 돼있어 이 내용 중심으로 작성해봤다. 사실 플랫폼 사용자가 적어 구글링 해도 잘 안나와서 공식 문서만 믿고 가야한다. 다행히 aws처럼 공식 문서가 끔찍하게 돼있지는 않아서 볼 만하다.*

## Pulumi란

IaC(Infrastructure as Code) 플랫폼 중 하나이다. sdk가 잘 돼있어 새로운 언어를 배울 필요가 없으며, 오픈 소스이다. ([vs terraform](https://www.pulumi.com/docs/intro/vs/terraform/#differences))

## Pulumi architecture

![pulumi_architecture](/assets/images/post/2023-03-23-How-pulumi-works-20230404221127.png)

### Project, Stack, State

`project`는 제일 큰 단위이며, `stack`은 `project` 내에 존재하는 배포 단위(dev, prod, test, …), `state`는 stack의 리소스의 metadata이다. (예를 들면 s3 bucket이 만들어졌는지, 만들어졌으면 이름은 뭔지 등등)

하나의 폴더 안에 하나의 `project`가 존재하고 `project` 안에 여러 개의 `stack`이 존재하고, 각각 `state`를 가진다. 각 stack이 만들어지면 project root path에 `Pulumi.<stackname>.yaml` 이름으로 파일이 만들어진다. 이 파일 안에는 환경변수나 aws region 등 `stack` 별 설정이 들어간다. (참고로 이 파일은 `state`와 다르다. 나중에 `state`가 어떻게 저장되는 지 설명함)

### Language host

`language executor`와 `language runtime`으로 구성되며, 간단히 말하면 `language executor`는 해당 pulumi sdk가 쓰인 언어를 실행(javascript, python, …)시키는 runtime 혹은 binary file이며, `language runtime`은  pulumi sdk 패키지라고 보면 된다. (node: `@pulumi/pulumi`, python: `pulumi`)

### Deployment engine

`language host`로부터 자원 관련한 요청들을 받아 현재 자원 상태(`state`)를 보고 상황에 따라 `resource provider`(얘가 pulumi 안에서 실제로 자원을 생성하는 마지막 단계)에게 요청을 전달하는 역할을 한다.

### Resource provider

`resource plugin`과 `sdk`로 구성되며, `sdk`는 pulumi에서 실제 자원을 생성하기 위해 사용하는 패키지이고 (node: `@pulumi/aws`, python: `pulumi_aws`), `resource plugin`은 `sdk`가 클라우드에 실제 요청을 보내기 위해 사용하는 binary file이다.

## Pulumi up을 하면…

```jsx
// index.js

import * as pulumi from '@pulumi/pulumi';
import * as aws from '@pulumi/aws';

const bucket = new aws.s3.Bucket('test');
```

1. pulumi cli가 node `language host` 실행
2. node `language host`가 프로그램(위 code) 실행
    1. aws.s3.Bucket 객체를 구성하고 `deployment engine`에 요청을 보내고 프로그램 실행을 계속한다.
    2. 이때 `new aws.s3.Bucket(...)`은 실제로 instance를 반환하는 게 아니고 생성 요청만 보내는 것임
    3. `language host`는 계속해서 코드를 실행하게 되고 다른 자원들에 관한 요청을 동시에 (비동기적으로) 보내게 됨
3. `deployment engine`은 요청을 받아서
    1. `pulumi up` 을 실행했을 때 리소스들의 `state`가 last `state`와 바뀌지 않았으면 딱히 하는 일이 없다. 이전과 다른 사항이 있어야만 동작한다.
    2. 위 코드를 처음 실행하는 경우 `state`도, 실제 resource도 새로 create한다.
        1. 가능한 동작은 create, update, replace, delete 정도이며, replace는 기본적으로 down time을 없게 하기 위해서 새로운 리소스를 create하고 그 다음 원래 있던 리소스를 delete한다.
    3. 이때 `deployment engine`은 aws에 직접 요청을 보내는 게 아니라 aws `resource provider` 안에 있는 aws `resource plugin`에 요청을 하게 되고, 이는 요청을 수행하기 위해 aws sdk를 사용한다.
    4. create 작업을 마치면 `state` file에 기록
4. node `language host` 종료
5. `deployment engine`, `resource provider` 종료

## 좀 더 알아보면

자원이 다른 자원을 참조할 때 create, delete 순서를 어떻게 정하는 지 등의 dependency problem을 pulumi가 해결한 방식은 먼가 되게 편리하면서도 단점이 있는데, [다른 글](/insight/2023/03/10/Dependency-in-pulumi)에서 적어 보겠다.

`stack`은 각각 `state`라고 불리는 metadata를 가지는데 `state`를 저장하는 백엔드로 pulumi service(app.pulumi.com)에서 관리할 수도 있고 로컬 혹은 다른 서버에서 관리할 수도 있다. 방금 말한 백엔드는 service 버전이고, self-manage 버전도 지원하는데 이는 aws s3나 google cloud storage에서 사용하는 object 형식의 metadata이다.

metadata가 pulumi service에 저장되면 (`state` 백엔드로 pulumi service를 이용하면) 얻는 이점은 다음과 같다.

- history of `state`
- encrypted `state`
- managed encryption and key management for `secret` (`secret`은 아까 말한 `Pulumi.<stack-name>.yaml`에 저장되는데 stack의 환경변수라고 보면 된다,)
- concurrent `state` locking (팀 단위로 운영하면 꼭 필요한 기능)
- etc…

몇 주 써보니까 그냥 제공하는 pulumi service 쓰는 게 정신건강에 제일 이롭다…

pulumi service는 2개의 endpoint(api.pulumi.com, app.pulumi.com)로 구성돼 있는데, pulumi cli는 api.pulumi.com의 api와 cloud provider의 api를 적절히 조합한 것이라서 pulumi service 자체는 client의 cloud credential을 알 방법이 없다. pulumi cli는 그저 aws api 같은 것들을 부르는 역할만 하기 때문이다.

이것도 pulumi 얘네들이 강조하는 기능 중 하나이다. cloud credential이 로컬에만 설정돼있으면 서버에 전송되지 않고 안전하게 동작할 수 있다.

## Reference

- [pulumi architecture doc](https://www.pulumi.com/docs/intro/concepts/)
- [pulumi aws tutorial](https://www.pulumi.com/docs/get-started/aws/begin/#before-you-begin)
