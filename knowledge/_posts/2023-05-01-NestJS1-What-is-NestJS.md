---
layout: post
type: tags
title: "[NestJS 1편] What is NestJS"
date: 2023-05-01 14:30:55 +0900
comments: true
toc: true
tags: nestjs oop
---


NestJS에 대해 나름 깊게? 탐구할 일이 생겨 그동안 알아봤던 것들을 공유하고자 한다. *사실 마지막 7, 8편을 쓰려고 이 많은 걸 썼다.* 

먼저 NestJS를 이해하려면 DI와 IoC를 알아야 하는데, 이 글의 인용문은 chat gpt에게 물어본 결과이다. 필자보다 말은 얘가 더 잘하니 읽으면서 참고하라고 같이 써봤다.

## Dependency injection

> Dependency Injection은 소프트웨어 공학에서 객체 지향 프로그래밍에서 일반적으로 사용되는 개념 중 하나입니다. DI는 객체 간의 의존성을 느슨하게 결합하기 위한 디자인 패턴으로, 의존성 주입이라는 방법을 사용하여 이를 구현합니다. 이는 객체가 직접 생성 및 의존성 해결을 수행하는 것이 아니라 다른 객체로부터 의존성을 받아들이는 것을 의미합니다.

```tsx
class B {}

// DI의 잘못된 예시
class A {
  constructor() {}
  a() {
    this.b = new B();
    this.b.something();
  }
}

// 옳게 된 예시 (constructor injection)
class A {
  constructor (public readonly b: B) {}
  a() {
    this.b.somthing();
  }
}
```

코드를 보면 `B`는 혼자 있어도 되는 클래스고, `A`는 동작에 있어 `B`를 필요로 한다. 즉, `A`의 dependency는 `B`이다.

그럼 `A` 클래스를 정의하는 위 2가지 방법 중 전자를 선택하게 되면 `A` 클래스가 `a` 메소드를 부를 때마다 `b`에 새로운 인스턴스를 생성하고 할당하게 될 것이다. 사실상 `B` 인스턴스를 한번만 할당하고 `something` 메소드를 계속 부르면 되는데도 말이다. 그럼 *`a` 메소드 안에서 `b`에 undefined 일때만 `B` 인스턴스를 생성하면 되는 거 아니냐* 라고 느낄 수 있는데 사람들이 그걸 하기 싫어서 DI 패턴을 채택하는 것이라고 필자는 생각한다.

사실 DI도 디자인 패턴 중 하나라 정답은 없지만, 위 방식(DI 측면에서 잘못된 예시)으로 선언하게 되면 `A`가 언제 `B` 인스턴스를 만들었는지, 몇 개 만들었는지 추적이 안 되고, `new B()`를 호출하는 데에 생각을 많이 해야한다. 지금은 아주 간단한 예시지만, 만약에 어느 클래스에서 필요로 하는 다른 클래스가 10개가 넘어가고 클래스 내 메소드 개수가 몇십 개 된다고 생각해 보면 끔찍하다. 그래서 `A`가 *아 그냥 바깥에서 `B` 인스턴스를 나한테 줘 그럼 걔 기능 쓰기만 할게* 라는 느낌으로 constructor를 통해 `B` 인스턴스**(dependency)**를 주입**(injection)**받으면 위에서 말한 문제점들이 해결된다. 다르게 말하자면 `B`를 instantiate 하는 책임을 외부로 떠넘긴 것이다.

## Inversion of control

> Inversion of Control은 DI의 개념을 구현하기 위해 사용되는 디자인 패턴입니다. ioc는 제어 역전이라고도 하며, 애플리케이션에서의 제어 흐름이 역전되는 것을 의미합니다. 즉, 객체들이 다른 객체를 직접 제어하지 않고, 대신 객체 컨테이너와 같은 외부 컴포넌트가 객체를 생성하고 관리하면서 제어의 흐름을 관리하도록 합니다. 이를 통해 코드의 재사용성과 유지 보수성이 향상됩니다.

위에서 instantiate하는 책임을 외부로 떠넘겼다고 했는데, *이 책임을 다른 객체가 받으면 그 객체는 누가 생성해주는거지? 결국 끝없는 순환 아닌가?* 라는 의문이 들 것이다. 그러면 아예 다른 객체들 관리만하는 애를 딱 한번 만들고 제어권한을 넘겨주기만 하면 해결이 될 것이다.

쉽게 말해 개발자가 `new …` 와 같은 instantiate 작업을 직접 하지 않고 하나의 외부 컴포넌트(흔히들 container라고 부르는 것 같다.)에게 다른 객체들의 제어권한**(control)**을 넘겨주는 것이다.**(inversion)** 이 글을 쓰는 시점에서 OOP에 대해 얕은 지식만 갖고 있어서 DI 같은 디자인 패턴에 대한 생각은 나중에 더 공부하고 자세하게 써 보겠다.

그럼 필요한 개념을 대충 알았으니 이제 NestJS가 뭔지 알아보자.

## NestJS

NestJS는 정말 간단하게 말해 NodeJS express(사실 express 말고 다른 것도 쓸 수 있다) 위에 DI와 IoC를 적용한 서버 패키지이며, 2016년 10월에 처음 릴리즈됐고, [Kamil Mysliwiec](https://github.com/kamilmysliwiec) 이란 사람이 거의 혼자 다 만들었다. NestJS는 decorator와 reflection 기능을 기반으로 동작한다. [[5편 참고]](/knowledge/2023/05/01/NestJS5-Decorator-reflection.html) *Typescript에서 공식적으로 decorator를 지원한 지 1년도 안돼서 릴리즈됐다. 이 방대한 걸 그 짧은 기간에 혼자 설계하고 개발했다는 게 정말 놀라웠다.*

참고로 이 다음부터는 NestJS 튜토리얼 정도는 해보고 글을 읽어보는 걸 추천한다. 그냥 공식문서 따라가면 된다. [[링크]](https://docs.nestjs.com/first-steps)

### dependency injection in NestJS

NestJS에서 dependency는 module 단위로 관리되며, module마다 provider, controller 등이 존재할 수 있다. 여기서

- cointroller는 http와 같은 요청을 받아서 service(provider 종류 중 하나)로 넘겨주는 역할을 한다.
- provider는 서비스 로직을 구현한 클래스이다. 예를 들면 controller로부터 http 요청을 받아서 db에 쿼리를 날린다던가 메일을 보낸다던가 다시 응답을 보낸다던가 등등…
- module 단위로 관리된다는 뜻은 `SomeModule`에서의 provider를 다른 `AnotherModule`에서 사용할 수 없다는 뜻. 사용하려면 export/import 작업이 필요하다.

사실 NestJS를 이용해 서버로 개발하는 입장에서는 controller가 뭔지 꼭 알아야 하지만, 필자가 다루고자 하는 주제에서는 module과 provider가 뭔지만 알면 된다. *controller는 NestJS 내부적으로 provider와 비슷하게 동작한다 아마도…*

예시 코드로 dependency가 어떻게 관리되는지 알아보자.

```tsx
import { Injectable, Module } from '@nestjs/common';

@Injectable()
class CatService {}

@Injectable()
class DogService {
  constructor(
    public readonly catService: CatService,
  ) {}
}

@Module({
  providers: [CatService, DogService],
})
class AnimalModule {}
```

`Injectable`은 provider에 다는 데코레이터이고, `Module`은 module에 다는 데코레이터이다.

`AnimalModule` 안에 `providers`로 `CatService`와 `DogService`를 등록해준 상태이고, 아까와 비슷하게 `CatService`는 혼자서도 잘 동작할 수 있고 `DogService`는 `CatService`를 dependency로 가진다.

그럼 적당한 곳(root module의 `imports` 같은 곳)에 `AnimalModule`을 등록해 두고 코드를 실행하면 모듈 안에서 일어나는 일은 다음과 같다.

1. `AnimalModule` 생성
2. `AnimalModule`에 있는 `providers`로 `CatService`와 `DogService`가 등록된 것을 확인
3. `CatService`와 `DogService`에 동시에 load작업을 함 (instantiate하기 위한 작업)
4. 이때 `DogService`는 dependency(`CatService`)가 resolve될 때까지 (instantiate 될 때까지) 기다리는 상태이며, `CatService`는 dependency가 없으므로 바로 instantiate 가능한 상태이다.
5. `CatService`가 instatiate되면 `DogService`는 dependency가 resolve되었으므로 생성자에 `CatService` 인스턴스를 주입받아 instantiate됨

이해를 돕기 위해 누가 누구를 기다린다는 둥 각 객체가 어떠한 동작을 하는 것처럼 설명해놨지만 실은 모든 과정 다 Nest container가 대신 하는 작업들이다.

### dependency scope

아까 dependency는 module 단위로 관리된다고 했었는데 코드로 알아보자.

```tsx
@Injectable()
class FlowerService {
  constructor(
    public readonly catService: CatService,
  ) {}
}

@Module({
  providers: [FlowerService],
})
class PlantModule {}
```

위 코드처럼 `PlantModule` 안에서 `FlowerService`의 dependency로 다른 모듈에 선언된 `CatService`를 등록하고 코드를 실행하면, NestJS는 `CatService` provider를 찾을 수 없다는 식의 에러 메시지를 출력한다. `FlowerService`에서 `CatService`를 쓰려면 module 관련 코드를 아래와 같이 다시 써야 한다.

```tsx
@Module({
  providers: [CatService, DogService],
	exports: [CatService],
})
class AnimalModule {}

...

@Module({
	imports: [AnimalModule],
  providers: [FlowerService],
})
class PlantModule {}
```

`PlantModule`에서 `AnimalModule`을 import하면 `AnimalModule`에서 export한 `CatService` provider를 사용할 수 있게 된다.

## Reference

- [IoC 관련 블로그 글](https://develogs.tistory.com/19)
- [nest doc](https://docs.nestjs.com/)