---
layout: post
type: tags
title: "[NestJS 3편] Custom provider"
date: 2023-05-01 14:30:57 +0900
comments: true
toc: true
tags: nestjs
---


2편에서는 provider wrapper를 잠깐 살펴봤었다. 후에 나올 dynamic module을 이해하기 위해서는 factory provider를 알아야 하는데, 이를 위해 custom provider를 살펴보자.

## Custom provider

custom provider란 말 그대로 provider를 wrapper 형식으로 custom하게 등록할 수 있게 해준 것이다. wrapper 형식은 다음과 같다. 

```tsx
{
  provide: <token>,
  use~~: <provider> 
}
```

use~~ 형식에는 useValue, useClass, useExisting, useFactory 이 4개를 쓸 수 있는데, 문서에 더 자세하게 나와있으므로 여기서는 그냥 대충 보자. [[참고]](https://docs.nestjs.com/fundamentals/custom-providers)

### useValue

useValue는 말 그대로 상수값을 provider로 등록하는 것이다. mocking 등에 쓰일 수 있다.

```tsx
import { CatsService } from './cats.service';

const mockCatsService = {
  /* mock implementation
  ...
  */
};

@Module({
  imports: [CatsModule],
  providers: [
    {
      provide: CatsService,
      useValue: mockCatsService,
    },
  ],
})
export class AppModule {}
```

테스트 목적으로 CatService에 대신 mockCatService를 넣고 싶을 때 이를 사용할 수 있다. 꼭 object 형태가 아니더라도 string, 배열 등 값이면 뭐든 가능하다.

### useClass

이건 우리가 아는 일반적인 provider이다. 2편에서 말했으므로, 따로 더 말하지는 않겠다. 다만 클래스 이름 그대로 쓰지 않고 wrapper로 감싸서 전달하면 dynamic(useFactory보다는 아니지만)하게 전달할 수 있다고는 한다.

```tsx
{
  provide: ConfigService,
  useClass:
    process.env.NODE_ENV === 'development'
      ? DevelopmentConfigService
      : ProductionConfigService,
};
```

### useFactory

진짜 dynamic하게 provider를 만들 수 있다. 코드로 보면 이해가 빠르다. (factory라는 용어가 oop 디자인 패턴 중 하나인 것으로 알고 있다.)

```tsx
import { FactoryProvider, Injectable, Module } from '@nestjs/common';

interface CatServiceOption {
  headLeg: string;
}

@Injectable()
class CatServiceOptionProvider {
  head = 'head';
  leg = 'leg';
  arm = 'arm';
}

@Injectable()
class CatService {
  headLeg = '';
  constructor(private readonly catServiceOption: CatServiceOption) {
    this.headLeg = catServiceOption.headLeg;
  }
}

const catServiceFactoryProvider: FactoryProvider = {
  provide: CatService,
  useFactory: (option: CatServiceOptionProvider) => {
    const headLeg = option.head + option.leg;
    return new CatService({ headLeg });
  },
  inject: [CatServiceOptionProvider],
};

@Module({
  providers: [CatServiceOptionProvider, catServiceFactoryProvider],
})
class CatModule {}
```

*예시가 처참해서 정말 죄송합니다*

`CatService`를 구성하는 데에 `headLeg`인 `CatServiceOption`이 필요한데, 이게 `CatServiceOptionProvider`라는 다른 곳에서 등록된 provider만이 이를 구성할 수 있는 정보인 `head`와 `leg`를 가지고 있다고 해보자. 

그러면 기존의 방식으로는 `CatService`가 `CatServiceOptionProvider`를 다 받아와서 직접 `headLeg`를 만들어야 할 것이다. 근데 `head`와 `leg`만 필요한데 `arm`까지 다 데려와서 `headLeg`를 만드는 게 맞는걸까? 어떻게 보면 집에 리모콘이 어디있는지 찾으려고 시장에 나가계신 어머니를 직접 불러다가 리모콘을 찾게 시키는 것이랑 같은 꼴이다. 그냥 전화해서 어디있는지만 들으면 되는데도 말이다. 만약 필요로 하는 option이 한두개가 아니라고 생각해 보면, 정말 간단한 걸 구성하는 데에도 복잡한 dependency가 생길 것이고, 필요한 것만 주입받는다 라는 DI 개념에 어긋난다. *(사실 생성자로만 주입받기 때문에 DI가 맞긴 한데, 음… 굳이 말 안해도 방금 한 말이 무슨 의미인지 이해했을 거라 믿는다)*

공장(factory)의 의미도 뭔가를 찍어낸다는 의미에서 사용된 것이다. 만약 `headLeg`를 구성하는 방식이 코드처럼 `head + leg`가 아니라 `head + leg + leg` 처럼 바뀌어야 된다고 하면, 물건 전체(provider)를 바꾸는 게 아니라 그저 공장에서 부품(option)을 바꾸면 된다.

다시 코드로 돌아와서, Nest container는 factory provider를 `inject`에 들어간 token으로 인스턴스를 찾은 다음에 `useFactory`에 명시된 함수에 parameter로 넣어줘서 반환받은 인스턴스를 `provide`에 적힌 token으로 등록하는 식으로 처리한다. FactoryProvider의 타입을 보면 이해가 쉬울 것이다. (`OptionalFactoryDependency`는 이름 그대로 optional dependency이다.)

```tsx
// packages/common/interfaces/modules/provider.interface.d.ts
export interface FactoryProvider<T = any> {
    provide: InjectionToken;
    useFactory: (...args: any[]) => T | Promise<T>;
    inject?: Array<InjectionToken | OptionalFactoryDependency>;

    ...

}
```

### example

그럼 어느 정도 개념을 이해했으니, 이상한 예시 말고 실제로 어떻게 쓰이는 지 보면,

```tsx
import { ConfigModule } from '@nestjs/config';
import { TypeOrmModule } from '@nestjs/typeorm';

TypeOrmModule.forRootAsync({
  imports: [ConfigModule],
  useFactory: (configService: ConfigService) => ({
    type: 'mysql',
    host: configService.get('HOST'),
    port: configService.get('PORT'),
    username: configService.get('USERNAME'),
    password: configService.get('PASSWORD'),
    database: configService.get('DATABASE'),
    entities: [],
    synchronize: true,
  }),
  inject: [ConfigService],
});
```

위 코드에서 환경변수로 설정해준 database 설정값들이 `TypeOrmModule`(db에 접근하는 기능을 제공하는 module)을 만드는 데에 들어가게 된다. (`ConfigService`는 환경변수에 접근할 수 있는 NestJS에서 제공하는 기본 provider이다.)

*아니 근데 이거는 provider가 아니라 module이고 `forRootAsync`라는 이상한 것도 있고 `useFactory`랑 `inject`말고는 겹치는 게 없잖아* 라는 생각이 당연히 들 것이다. factory provider의 기능은 바로 dynamic module에서 빛을 발하는데, 4편에서 살펴보자.

## Reference

- [nest doc - custom provider](https://docs.nestjs.com/fundamentals/custom-providers)
- [nest doc - typeorm module config](https://docs.nestjs.com/techniques/database#async-configuration)