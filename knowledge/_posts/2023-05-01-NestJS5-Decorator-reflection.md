---
layout: post
type: tags
title: "[NestJS 5편] Decorator, reflection"
date: 2023-05-01 14:30:59 +0900
comments: true
toc: true
tags: nestjs typescript
---


NestJS는 데코레이터와 reflection 기능을 기반으로 동작한다. 서비스로 등록하고 싶은 클래스에 `Injectable` 데코레이터를 달아주고, 모듈로 등록하고 싶은 클래스에 `Module` 데코레이터를 달아줘야만 제대로 동작하기 때문이다.

그럼 데코레이터와 reflection이 뭔지, 이들이 어떻게 쓰이는지 살펴보자.

## Decorator

데코레이터는 익숙한 사람이 많을 것이다. 말 그대로 객체를 꾸며주는? 기능이며 다른 언어에서도 많으니 굳이 자세히 설명하지 않겠다. [[링크]](https://www.typescriptlang.org/docs/handbook/decorators.html)

다만 typescript에서의 데코레이터 기능은 실험적인 기능이라 정말 만약에 그럴일은 없겠지만 없어질 수도 있다. *(진짜 근본없는 언어인거 같다 ㅋㅋ)* NestJS에서는 class decorator와 parameter decorator가 주로 쓰이므로 언급만 하고 넘어가겠다.

```tsx
const ClassDec = <T extends { new (...args: any[]): {} }>(target: T) => {
  console.log(target.name);
  return class K extends target {
    k: number;
    constructor(...options) {
      super(...options);
      this.k = 3;
    }
  };
};

const ParamDec: ParameterDecorator = <T extends { new (...args: any[]): {} }>(
  target: T,
  key: string,
  idx: number,
) => {
  console.log(`${target.name}, ${key}, ${idx}`);
};

@ClassDec
class A {
  constructor(
    // A, undefined, 0
    @ParamDec
    public readonly a: number,
  ) {}
  // undefined, method, 0
  method(@ParamDec asd: number) {}
}

console.log(new A(3)); // K { a: 3, k: 3 }
```

`target`은 클래스 그 자체가 매개변수로 넘어오는 것이며, `target`을 그냥 `object` 타입으로 받아버리면 클래스 상속을 못해서 `<T extends { new (...args: any[]): {} }>` 를 붙여봤다. 뜻은 *`T`는 클래스 타입입니다~* 라는 뜻이다.

`ClassDec`에서 또다른 클래스를 반환해주면 instantiate할 때에 반환해준 클래스 `K`로 대체가 되고, `ParamDec`에서 `target`이 내부 메소드 `method`에서는 `undefined`로 나오는데 왜 저러는지는 모르겠다. 중요하지는 않다.

## Reflection

gpt 피셜 reflection은 java에서 처음 나온 개념으로, 객체에 대한 정보(class metadata)를 저장하고 사용하는 것을 말한다고 한다. 주로 데코레이터 안에서 클래스의 metadata를 set하는 식으로 데코레이터와 세트 느낌으로 같이 쓰인다. *reflection을 하면 자기가 자기를 참조하는 일이 많은데 이 self-reference 때문에 reflection이라고 이름을 붙인 걸까? 정확한 어원은 모르겠다…*

### reflect-metadata

공식문서에서도 나오는 패키지로, typescript에서는 reflection 기능을 사용하려면 `Reflect.getMedtadata` 이런 식으로 사용해야하는데, `reflect-metadata` 패키지를 import하고 나서 사용하면 더 강력한 기능을 사용할 수 있다. 간단한 예시는 [여기](https://github.com/rbuckton/reflect-metadata) 나와있다.

### design:paramtypes

`design:paramtypes`는 metadata key들 중 그 이름이 예약된 key인데, constructor의 parameter type을 가져온다.

```tsx
import 'reflect-metadata';

const PARAMTYPES_METADATA = 'design:paramtypes';

const Empty: ClassDecorator = (target: object) => {};

class B {}

@Empty
class A {
  constructor(public readonly a: B) {}
}

console.log(Reflect.getMetadata(PARAMTYPES_METADATA, A)); // [ [ class B ] ]
```

`Empty`라는 아무것도 안 하는 데코레이터를 단 이유는 아무 데코레이터라도 달아줘야만 예약된 key를 사용할 수 있기 때문이다. *이거땜에 몇일을 날렸다…*

## In NestJS

그럼 NestJS가 위 2가지 기능을 기반으로 어떻게 동작하는지 짐작할 수 있는데, 요약하자면 다음과 같다. 

1. depedency가 존재하는 클래스*(예를 들면 `DogService`를 필요로 하는 `CatService`)*를 instantiate할 때 constructor parameter type(`DogService`)을 `design:paramtypes` metadata key로 가져온다.
2. **instance들을 저장하는 어딘가**에서 해당 타입*(`DogService`)*을 가지는 instance*(`DogService instance`)*를 가져와서 해당 클래스*(`CatService`)*를 instantiate한다.

실제 코드에서 어떻게 이걸 구현했는지는 NestJS 마지막편에서 다루도록 하겠다.

## Reference

- [ts doc- decorator](https://www.typescriptlang.org/docs/handbook/decorators.html)
- [reflect-metadata](https://github.com/rbuckton/reflect-metadata)