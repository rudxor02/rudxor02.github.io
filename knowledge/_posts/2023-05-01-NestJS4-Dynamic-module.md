---
layout: post
type: tags
title: "[NestJS 4편] Dynamic module"
date: 2023-05-01 14:30:58 +0900
comments: true
toc: true
tags: nestjs
---


factory provider가 dynamic module에서 어떻게 쓰이는 지 알아보려면 당연히 dynamic module이 뭔지 알아야 한다. 이번 편에서도 되도록 공식문서를 번역하는 것은 지양하도록 하겠다.

## Dynamic module

그전에 우리가 쓰던 게 static module인데, 형식을 다시 살펴보자.

```tsx
// packages/common/interfaces/modules/module-metadata.interface.d.ts
export interface ModuleMetadata {
    imports?: Array<Type<any> | DynamicModule | Promise<DynamicModule> | ForwardReference>;
    controllers?: Type<any>[];
    providers?: Provider[];
    exports?: Array<DynamicModule | Promise<DynamicModule> | string | symbol 
      | Provider | ForwardReference | Abstract<any> | Function>;
}
```

`ModuleMetadata`는 `Module` 데코레이터에 들어가는 object의 타입인데, 만약 상황에 따라 달라지게 만드는 module을 만들고 싶다면 어떡할까? 앞서 봤던 custom provider와 비슷한 경우이다. 다만 provider에서 module로 옵션을 넣어줄 수 있는 레벨이 올라갔을 뿐이다. NestJS의 단위는 module이므로, factory provider처럼 module레벨에서도 dynamic하게 설정할 수 있어야 한다는 생각은 지극히 자연스럽다.

```tsx
// packages/common/interfaces/modules/dynamic-module.interface.d.ts
export interface DynamicModule extends ModuleMetadata {
    module: Type<any>;
    global?: boolean;
}
```

provider wrapper의 경우처럼 module도 dynamic module로 사용할 때에 위 형식처럼 쓸 수 있다. 예를 들면,

```tsx
{
  module: PlantModule,
  imports: [ AnimalModule ],
  providers: [ FloweService ],
}
```

위와 같이 사용하면 3편에서 봤던 `Module` 데코레이터를 붙인 `PlantModule`과 동일하게 동작한다. (그러나 static module이 아니라 dynamic module로 인식될 것이다)

근데 아까 module레벨에서 factory provider처럼 dynamic하게 설정할 수 있어야 한다고 했는데, 그러러면 module 클래스의 static 메소드로 factory option을 전달하고, factory provider에 다시 전달하면 된다. 코드로 살펴보자. [[3편 코드 참고]](/knowledge/2023/05/01/NestJS3-Custom-provider.html#usefactory)

```tsx
import {
  DynamicModule,
  FactoryProvider,
  Injectable,
  Module,
} from '@nestjs/common';

interface CatServiceOption {
  headLeg: string;
}

// @Injectable()
// class CatServiceOptionProvider {
//   head = 'head';
//   leg = 'leg';
//   arm = 'arm';
// }

@Injectable()
class CatService {
  headLeg = '';
  constructor(private readonly catServiceOption: CatServiceOption) {
    this.headLeg = catServiceOption.headLeg;
  }
}

// const catServiceFactoryProvider: FactoryProvider = {
//   provide: CatService,
//   useFactory: (option: CatServiceOptionProvider) => {
//     const headLeg = option.head + option.leg;
//     return new CatService({ headLeg });
//   },
//   inject: [CatServiceOptionProvider],
// };

// @Module({
//   providers: [CatServiceOptionProvider, catServiceFactoryProvider],
// })
// class CatModule {}

class CatModule {
  static create(head = 'head', leg = 'leg'): DynamicModule {
    const catServiceFactoryProvider: FactoryProvider = {
      provide: CatService,
      useFactory: () => {
        const headLeg = head + leg;
        return new CatService({ headLeg });
      },
    };

    return {
      module: CatModule,
      providers: [catServiceFactoryProvider],
    };
  }
}
```

이전 코드(주석)에서는 factory option을 구성하는 걸 `CatServiceOptionProvider`가 했었는데, 위 코드처럼 바뀌면 module 레벨에서 option을 전달해서 `CatModule.create`를 하게 되면 똑같은 결과가 나타난다. 심지어 사용하기도 더 편해졌다.

예시 말고 실제로 어떻게 사용되는지 살펴보면,

```tsx
import { ConfigModule } from '@nestjs/config';

ConfigModule.forRoot({ envFilePath: './config/.env' })
```

이처럼 `forRoot`를 사용하면 `ConfigModule`이 해당 path에서 환경변수를 가져온다.

`forRoot`, `create` 같은 static 메소드 이름은 그냥 의미상 붙인 이름일 뿐이고, 직접 구현한다면 `DynamicModule`만 반환하면 어느것이든 상관없다. 다만 이름에는 어느 정도 표준이 존재하는데, `register`, `forRoot`, `forFeature` 정도가 있다. (참고로 `registerAsync`, `forRootAsync`, `forFeatureAsync`는 `Promise<DynamicModule>`을 반환한다. 비동기적으로 동작한다는 의미에서 뒤에 Async를 붙인다.)

### register

`register`는 방금 `create`처럼 부를때 option마다 다른 dynamic module을 생성하고 싶을 때 사용한다.

```tsx
CatModule.register(head: 'head1');

...

@Module({imports: [ CatModule.register(head: 'head1') ]})
class SomeModule{}

// another cat module!
@Module({imports: [ CatModule.register(head: 'head2') ]})
class AnotherModule{}
```

근데 여기서 provider token(예를 들면 `provide: CatService`)을 사용할때처럼 `module: CatModule`이 `CatModule.forRoot`의 반환값에 포함돼있으므로 *`import: [ CatModule ]` 하면 안되는걸까* 라는 생각이 들 것이다. 안 된다. 방금 말한 것은 static module로 인식되어 module token이 다르게 생성되기 때문이다.

참고로 같은 클래스에서 나온 dynamic module이라도 metadata가 다르면 module token이 다르게 생성된다. metadata 정보가 포함된 string을 hashing한 값이 token이 되기 때문이다. 그럼 `module: CatModule` 이건 왜해주는거냐는 생각이 들 것인데 이건 token을 위한 게 아니라 다른걸 위한 것이다. [[7편 참고]](/insight/2023/05/01/NestJS7-How-tokens-are-managed.html#how-tokens-are-managed-1) *방금 말한 의문이 1편부터 8편까지의 모든 내용을 알아보게 된 계기다*

### forRoot

`forRoot`는 해당 모듈이 dynamic module로서 한 번 설정되고, 그 설정이 다른 곳에서 여러번 쓰일 일이 있을 때에 사용한다. 제대로 이해하려면 `forFeature`와 같이 봐야 한다.

### forFeature

`forFeature`는 `forRoot`에서 사용한 설정을 이용해서 필요한 기능을 위해 dynamic module을 새로 만들 때 사용한다. 말로 하면 너무 어려우니까 코드로 보자.

```tsx
import { Cat } from './cat.entity';
import { Dog } from './dog.entity';

TypeOrmModule.forRoot({
      type: 'mysql',
      host: 'localhost',
      port: 3306,
      username: 'root',
      password: 'root',
      database: 'test',
      entities: [ Cat, Dog ],
      synchronize: true,
    }),

...

@Module({
  imports: [ TypeOrmModule.forFeature([ Cat ]) ],
  providers: [ CatService ],
  controllers: [ CatController ],
})
class CatModule {}

@Module({
  imports: [ TypeOrmModule.forFeature([ Dog ]) ],
  providers: [ DogService ],
  controllers: [ DogController ],
})
class DogModule {}
```

module에서 `TypeOrmModule.forFeature([ Cat ])`을 import하면 그 module에 `Cat` entity가 저장된 테이블에 접근할 수 있게 해주는 provider가 주입된다. 그 provider가 뭔지, service는 그걸 어떻게 사용하는 지 등의 자세한 내용은 공식문서 참고 바란다. [[링크]](https://docs.nestjs.com/techniques/database#repository-pattern)

`CatService`는 `Cat` entity가 저장된 테이블에 접근해야 하고, `DogService`는 `Dog` entity가 저장된 테이블에 접근해야 한다고 하자. 아까처럼 `register`를 그대로 가져와서 똑같은 걸 여러 번 import시키는 것도 가능은 하겠지만 비효율적이다. 코드도 길어지고 (module에 넣어줄 때마다 일일이 db host, username 등을 명시해 줘야 한다) 서비스에서 필요없는 테이블까지 접근할 수 있기 때문이다.

이 문제점은 db 설정만을 위한 `forRoot`를 따로 두고, import만을 위한 `forFeature`를 따로 두게 되면 해결된다. 이때 `forFeature`는 `forRoot`의 설정값으로 생성한 module을 사용자가 바라는 기능에 맞게 살짝 바꿔서 가져오는 역할을 한다.

이전 편에서 봤던 코드가 이해될 것이다. [[3편 코드 참고]](/knowledge/2023/05/01/NestJS3-Custom-provider.html#example)

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

이렇게 하면 내부 어딘가에 있을 factory provider에게 저기 있는 `useFactory`와 `inject`가 그대로 전달되어 저 설정값대로 db에 연결한다.

## TypeOrmModule

사용자 입장에서 위와 같이 `TypeOrmModule`을 사용하는 건 쉽다. 그런데 이게 내부적으로 어떻게 가능한걸까? 필요한 entity를 넣어주면 그 entity가 저장된 테이블에 접근하는 provider를 주입해준다니… 우리가 이때까지 알아본 걸로 구조를 설명할 수 있다. 필자가 추상적으로 설명한 것이니 정확한 구조와 다를 수 있다.

- `TypeOrmModule`은 core module이 존재하는데, 이곳에 모든 게 담긴다.
- core module에는 각 entity마다 저장된 테이블에 접근할 수 있는 provider가 하나씩 존재한다.
- `forRoot`를 부르게 되면 위 2개가 모두 구성되고 그냥 dynamic module을 반환한다. (forRoot는 설정용)
- `forFeature`를 부르게 되면 core module의 provider 중에서 필요한 provider만 감싸서 dynamic module로 반환한다.

직접 보니 생각보다 구조가 간단해서 그냥 소스코드 들고 와봤다. [[링크]](https://github.com/nestjs/typeorm) `forRoot`와 `forFeature`의 반환값만 보기 바란다.

```tsx
// lib/typeorm.module.ts

@Module({})
export class TypeOrmModule {
  static forRoot(options?: TypeOrmModuleOptions): DynamicModule {
    return {
      module: TypeOrmModule,
      imports: [TypeOrmCoreModule.forRoot(options)],
    };
  }

  static forFeature(
    entities: EntityClassOrSchema[] = [],
    dataSource:
      | DataSource
      | DataSourceOptions
      | string = DEFAULT_DATA_SOURCE_NAME,
  ): DynamicModule {
    const providers = createTypeOrmProviders(entities, dataSource);
    EntitiesMetadataStorage.addEntitiesByDataSource(dataSource, [...entities]);
    return {
      module: TypeOrmModule,
      providers: providers,
      exports: providers,
    };
  }

 ...

}
```

저기 `createTypeOrmProviders`에 `entities`를 전달해주면 해당 entity에 접근할 수 있는 `providers`를 가져와 export해주는 dynamic module을 반환하는 걸 볼 수 있다.

다음 코드도 형식만 보자. `useFactory`를 사용해서 provider를 동적으로 생성하는 것을 볼 수 있다.

```tsx
// lib/typeorm.providers.ts

export function createTypeOrmProviders(
  entities?: EntityClassOrSchema[],
  dataSource?: DataSource | DataSourceOptions | string,
): Provider[] {
  return (entities || []).map((entity) => ({
    provide: getRepositoryToken(entity, dataSource),
    useFactory: (dataSource: DataSource) => {
      const enitityMetadata = dataSource.entityMetadatas.find((meta) => meta.target === entity)
      const isTreeEntity = typeof enitityMetadata?.treeType !== 'undefined'
      return isTreeEntity 
        ? dataSource.getTreeRepository(entity)
        : dataSource.options.type === 'mongodb'
          ? dataSource.getMongoRepository(entity)
          : dataSource.getRepository(entity);
    },
    inject: [getDataSourceToken(dataSource)],  
    targetEntitySchema: getMetadataArgsStorage().tables.find(
      (item) => item.target === entity,
    ),
  }));
}
```

## Reference

- [nest doc - dynamic module](https://docs.nestjs.com/fundamentals/dynamic-modules)
- [nest doc - typeorm](https://docs.nestjs.com/techniques/database)
- [nest doc - config](https://docs.nestjs.com/techniques/configuration)
