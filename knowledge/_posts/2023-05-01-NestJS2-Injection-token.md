---
layout: post
type: tags
title: "[NestJS 2편] Injection token"
date: 2023-05-01 14:30:56 +0900
comments: true
toc: true
tags: nestjs
---


Nest container를 통해 dependency가 관리된다는 건 알겠는데, 인스턴스를 뭘로 구분할까? 예를 들면 `CatService`의 인스턴스를 **어딘가**에 저장해놓고 `DogService`가 instantiate될 때 그걸 불러와야 하는데, 그 **어딘가**에는 `CatService` 말고 다른 인스턴스들도 있을건데, 뭘로 `CatService`의 인스턴스를 찾을까?

정답은 바로 token인데, 우선 `Inject` 데코레이터에 대해 알아보도록 하자.

## Inject decorator

`Inject` 데코레이터는 provider에 할당되는 token을 지정할 수 있는데, 다음과 같이 사용할 수 있다. [[1편 코드 참고]](/knowledge/2023/05/01/NestJS1-What-is-NestJS.html#dependency-injection-in-nestjs)

```tsx
import { Inject, Injectable, Module } from '@nestjs/common';

@Injectable()
class CatService {}

@Injectable()
class DogService {
  constructor(
    @Inject('CAT_SERVICE_TOKEN')
    public readonly catService: CatService,
  ) {}
}

@Module({
  providers: [
    {
      provide: 'CAT_SERVICE_TOKEN',
      useClass: CatService,
    }, 
    DogService,
  ],
})
class AnimalModule {}
```

우선 `AnimalModule`의 `providers`에 이상한 형식의 object가 들어온 것을 확인할 수 있다. 이는 provider wrapper라고 부르는데, `useClass`에는 provider로 등록할 클래스가 들어가고, `provide`에는 provider에 할당할 token이 들어간다. *(이름을 token이라 하면 될 것이지 왜 provide라고 해놨는지 모르겠다)* 위 코드에서는 `CatService`의 token으로  `CAT_SERVICE_TOKEN` string이 들어간 것을 확인할 수 있다.

그리고 `DogService`의 constructor parameter 중 하나인 `catService`에 붙인 `Inject` 데코레이터로 똑같은 string을 넣어주면, `DogService`가 생성될 때 `catService`에 들어갈 인스턴스를 `CAT_SERVICE_TOKEN` string token으로 찾아서 넣어준다. 다른 문자열이 들어가면 인스턴스를 찾을 수 없다고 하며 에러가 날 것이다.

그럼 여기서 한 가지 의문점이 드는데, 1편에서 썼던 코드, 즉 token을 지정하지 않으면 기본적으로는 무슨 token을 쓸까? 우선 token의 타입부터 살펴보자. *주석에 써진 경로는 nestjs/nest 레포에서 파일 위치이다. [[링크]](https://github.com/nestjs/nest)*

```tsx
// packages/common/interfaces/modules/injection-token.interface.ts
export type InjectionToken =
  | string
  | symbol
  | Type<any>
  | Abstract<any>
  | Function;
```

마지막 2개는 어느 경우에 쓰이는지 잘 모르겠다 ㅎㅎ;; 아무튼 3번째 타입으로 `Type<any>`가 들어가는데,

```tsx
// packages/common/interfaces/type.interface.ts
export interface Type<T = any> extends Function {
  new (...args: any[]): T;
}
```

쉽게 말해 클래스 그 자체를 가리키는 타입이다. 이때까지 `Module` 데코레이터의 `providers` 옵션에 클래스 이름 그대로 넣어줬던 게 기억날 것이다. 그게 `Type<any>` 타입이다. `providers`에 클래스 이름을 그대로 넣어주게 되면 아래 provider wrapper와 동일한 의미를 가진다.

```tsx
// CatService만 넣어주게 되면 다음과 같은 의미를 가진다.
{
  provide: CatService, 
  useClass: CatService,
}
```

클래스 그 자체가 provider가 되면서 token도 되는 것이다.

dependency를 주입받는 쪽에서 `Inject` 데코레이터를 안붙여줬을 때도 똑같이 그 parameter에 타입으로 할당된 클래스 그 자체를 token으로 갖는 인스턴스를 주입받는다.

요약하자면 클래스에서 dependency를 주입받을 때 `Inject` 데코레이터에 넣어준 token과 provider wrapper에서 선언한 token이 일치하는 인스턴스를 주입받게 된다.

## Module token

앞서 본 것처럼 provider의 인스턴스를 등록하는 데에 token이 쓰이듯이, module을 등록하는 데에도 token이 쓰인다. 다만 module token은 구성하는 방식이 조금 provider token과는 다른데, 이는 7편에서 소스코드와 함께 살펴보겠다. 지금은 그냥 module에도 token이 이용된다는 것만 알아두도록 하자.

### import/export

module token이 어떻게 쓰이는지 코드로 살펴보자. 1편에서 다음과 같은 코드를 봤을 것이다.

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

`FlowerService`에서 `CatService`를 사용하기 위해서 `PlantModule`이 `AnimalModule`을 import하는 모습이다. 이때 `AnimalModule`이 module token(정확히는 `${randomString}_${moduleName}` 의 hash string이 token이다)과 함께 **모듈 저장소**에 함께 등록되고, `PlantModule`에서 `AnimalModule`을 찾을 때에 이 token을 사용한다.

일반적으로 NestJS 사용자는 module token에 접근할 수 없는데, *왜 provider token과 다르게 module token은 접근할 수 없는가*는 결국 *왜 dependency가 module 단위로 관리되느냐*라는 말과 같다. 이것도 왜 그런지 나름의 이유를 생각해봤는데 나중에 8편에서 살펴보도록 하자. [[8편 참고]](/insight/2023/05/01/NestJS8-How-NestJS-works.html#why-module-token-is-not-accessible)

## Reference

- [nest doc - standard providers](https://docs.nestjs.com/fundamentals/custom-providers#standard-providers)
