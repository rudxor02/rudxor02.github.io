---
layout: post
type: tags
title: "[NestJS 7편] How tokens are managed"
date: 2023-05-01 14:31:01 +0900
comments: true
toc: true
tags: nestjs
---


이제 필요한 개념들을 다 알았으니 7편과 8편에서는 이 개념들이 NestJS 소스코드에 어떻게 적용되었는지 알아볼 것이다. 이번 7편에서는 token이 어떻게 관리되는지부터 살펴보자. *뇌피셜 주의*

## Provider token

![Module](/assets/images/post/2023-05-01-NestJS7-How-tokens-are-managed-20230501175646.png)

provider token은 `Module` 안에 _provider 안에 (타입은 `Map<InstanceToken, InstanceWrapper>`) 저장된다. `InstanceWrapper`는 provider당 하나씩 생성되며, 이에 token이 생성되어 `InstanceToken`을 key로, `InstanceWrapper`을 value로 Map에 할당된다. 전체적인 구조는 8편 참고바란다.

### type

2편에서 언급한 `InstanceToken`과 `Provider`의 type을 다시 살펴보자.

```tsx
// packages/common/interfaces/modules/injection-token.interface.ts
export type InjectionToken =
  | string
  | symbol
  | Type<any>
  | Abstract<any>
  | Function;

// packages/common/interfaces/modules/provider.interface.ts
export type Provider<T = any> =
  | Type<any>
  | ClassProvider<T>
  | ValueProvider<T>
  | FactoryProvider<T>
  | ExistingProvider<T>;
```

provider token은

- 사용자가 provider wrapper가 아닌 클래스 이름만 적는 형식으로 provider를 등록했다면 클래스를 그대로 token으로 사용하면 되고(`Type<any>`)
- provider wrapper 형식으로 등록했으면 provide 프로퍼티에 지정한 값을 그대로 가져와 할당하면 된다.

### how tokens are managed

```tsx
// packages/core/injector/module.ts
export class Module {

 ...

 public addProvider(provider: Provider, enhancerSubtype?: EnhancerSubtype) {
  if (this.isCustomProvider(provider)) {
      if (this.isEntryProvider(provider.provide)) {
        this._entryProviderKeys.add(provider.provide);
      }
      return this.addCustomProvider(provider, this._providers, enhancerSubtype);
    }
 
    this._providers.set(
      provider,
      new InstanceWrapper({
        token: provider,
        name: (provider as Type<Injectable>).name,
        metatype: provider as Type<Injectable>,
        instance: null,
        isResolved: false,
        scope: getClassScope(provider),
        durable: isDurable(provider),
        host: this,
      }),
    );

  ...

 }

 ...

 public addCustomFactory(
     provider: FactoryProvider,
     collection: Map<Function | string | symbol, InstanceWrapper>,
     enhancerSubtype?: EnhancerSubtype,
   ) {
     const {
       useFactory: factory,
       inject,
       scope,
       durable,
       provide: providerToken,
     } = provider;
 
     collection.set(
       providerToken,
       new InstanceWrapper({
         token: providerToken,
         name: (providerToken as Function)?.name || providerToken,
         metatype: factory as any,
         instance: null,
         isResolved: false,
         inject: inject || [],
         scope,
         durable,
         host: this,
         subtype: enhancerSubtype,
       }),
     );
   }

  ...

}
```

코드에서도 보이듯이

- custom provider가 아니면 그냥 provider 그 자체를 token으로 사용하고(이때 provider는 `Type<any>`)
- custom provider면 (factory provider를 예시로 가져와봤다.) `provide` 속성에서 지정한 token으로 `InstanceWrapper`에 할당하는 것을 볼 수 있다.

`addProvider`에서 `this._providers`와 `addCustomFactory`에서 `collection`이 `Map<InstanceToken, InstanceWrapper>`이다. (두 변수는 같은걸 가리킨다.)

## Module token

![Nest Container](/assets/images/post/2023-05-01-NestJS7-How-tokens-are-managed-20230501175720.png)

module은 위 그림에서 ModulesContainer에 저장되는데, 이때도 module token을 key로, Module을 value로 Map에 저장된다. (`Map<string, Module>`) token은 `ModuleTokenFactory`라는 곳에서 생성된다.

### type

module token은 string이며, DynamicModule의 타입을 다시 보자.

```tsx
// packages/common/interfaces/modules/dynamic-module.interface.ts
export interface DynamicModule extends ModuleMetadata {
  module: Type<any>;
  global?: boolean;
}

// packages/common/interfaces/modules/module-metadata.interface.ts
export interface ModuleMetadata {
  imports?: Array<
    Type<any> | DynamicModule | Promise<DynamicModule> | ForwardReference
  >;
  controllers?: Type<any>[];
  providers?: Provider[];
  exports?: Array<
    | DynamicModule
    | Promise<DynamicModule>
    | string
    | symbol
    | Provider
    | ForwardReference
    | Abstract<any>
    | Function
  >;
}
```

dynamic module metadata라고 하면 module에 할당된 위 정보들 모두를 말하는 것이다.

### how tokens are managed

```tsx
// packages/core/injector/module-token-factory.ts
export class ModuleTokenFactory {

 ...

 public create(
    metatype: Type<unknown>,
    dynamicModuleMetadata?: Partial<DynamicModule> | undefined,
  ): string {
    const moduleId = this.getModuleId(metatype);

    if (!dynamicModuleMetadata) {
      return this.getStaticModuleToken(moduleId, this.getModuleName(metatype));
    }
    const opaqueToken = {
      id: moduleId,
      module: this.getModuleName(metatype),
      dynamic: dynamicModuleMetadata,
    };
    const opaqueTokenString = this.getStringifiedOpaqueToken(opaqueToken);

    return this.hashString(opaqueTokenString);
  }

 ...

 public getStaticModuleToken(moduleId: string, moduleName: string): string {
    const key = `${moduleId}_${moduleName}`;
    if (this.moduleTokenCache.has(key)) {
      return this.moduleTokenCache.get(key);
    }

    const hash = this.hashString(key);
    this.moduleTokenCache.set(key, hash);
    return hash;
  }

 ...

}
```

`create`에서 시작해서 대상 module이

- static module이면 (클래스 이름 그대로 사용한 module) `getStaticModuleToken` 안에서 `moduleId`와 `moduleName`을 붙인 것의 hash값이 token이 된다. (이때 `moduleId`는 random string, `moduleName`은 module 클래스 이름)
- dynamic module이면 `moduleId`와 `moduleName`, `dynamicModuleMetadata`를 한 곳에 모아 통째로 stringify → hash값을 token으로 사용한다. 이렇게 되면 module wrapper에 넣어준 값들 중 하나만 달라져도 다른 token이 할당되어 새로운 instance를 생성한다.

## Reference

- [nest github](https://github.com/nestjs/nest)
