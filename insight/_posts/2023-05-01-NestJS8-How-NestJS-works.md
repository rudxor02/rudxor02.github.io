---
layout: post
type: tags
title: "[NestJS 8편] How NestJS works"
date: 2023-05-01 14:31:02 +0900
comments: true
toc: true
tags: nestjs oop
---


자 드디어 마지막 편이다! 지금까지 알아온 것들을 바탕으로 Nest container가 어떻게 DI와 IoC를 적용하는 지 살펴보자. 정말정말 복잡하고 글도 길고 코드도 필자가 보고싶었던 부분만 본 거고 모든 내용이 필자의 뇌피셜이고  아 이런게 있구나 하는 정도로만 봐주길 바란다. 구조를 보여주는 사진도 추상적으로 마음대로 그린 것이다. ***진짜 뇌피셜 주의***

## NestJS Architecture

![Nest Factory](/assets/images/post/2023-05-01-NestJS8-How-NestJS-works-20230501175801.png)

제일 높은 레벨에서 보면 `NestFactoryStatic` 안에 `NestApplication`이 존재하며 그 안에

- `NestContainer`는 IoC 개념에서 나온 자원들에 대한 모든 권한(control)을 사용자로부터 위임(inversion)받은 container이다. [[1편 참고]](/knowledge/2023/05/01/NestJS1-What-is-NestJS.html#inversion-of-control)
- `GraphInspecor`는 후에 나올 `NestContainer`가 만든 `SerializedGraph`를 inspect하는 역할을 한다. 아마 dependency 관련해서 에러가 났을 때 사용자에게 어떤 에러가 났는지 알려주기 위해 존재하는 것 같다.
- `DependenciesScanner`는 사용자가 등록한 module, provider 등을 바탕으로 어떤 것에는 어떤 dependency가 있는지 등을 scan하는 역할을 한다.
- `InstanceLoader`와 `Injector`는 `DependenciesScanner`가 사용하는 도구? 느낌으로 실제 DI 과정은 얘네들을 거친다.

### NestContainer

![Nest Container](/assets/images/post/2023-05-01-NestJS8-How-NestJS-works-20230501175816.png)

`NestContainer` 안에

- `ModuleCompiler`는 `ModulesContainer`에 `Module`을 저장할 때에 쓰인다. 이때 module token은 `ModuleTokenFactory`에서 생성된다.
- 기본적으로 생성되는 `InternalCoreModule`이 있는데, 그 안에 `ModulesContainer` 와 `SerializedGraph` 등이 있다.

### Module

![Module](/assets/images/post/2023-05-01-NestJS8-How-NestJS-works-20230501175845.png)

`Module` 안에

- `providers`, `injectables`, `middlewares`, `controllers` 안에 `InstanceWrapper`가 저장되며, 이곳에 실제 instance들이 저장된다. `InjectionToken`과 함께 저장된다. (`Map<InjectionToken, InstanceWrapper>`)
- core provider란 것도 있는데 `Module`은 자기 부모 `Module` 그 자체이며, `ModuleRef`는 부모 `Module`을 가리키는 reference이다. (솔직히 뭐가 다른지는 모르겠다. `ModuleRef`는 자주 쓰이는 걸 봤는데 `Module`은 어디에 쓰이는지 모르겠다.) `ApplicationConfig`라는 `NestApplication`에 대한 전체적인 설정값이 각 `Module`마다 똑같이 저렇게 저장되는 것 같다.

### Instance wrapper

![Instance Wrapper](/assets/images/post/2023-05-01-NestJS8-How-NestJS-works-20230501175900.png)

`InstanceWrapper` 안에

- `values` 안에 (`Map<ContextId, InstancePerContext>`) 클래스마다 instance가 저장되는데, `ContextId`는 신경 안 써도 된다. 아마 provider의 scope를 `REQUEST`나 `TRANSIENT`로 설정했을 때에 context마다 새로운 인스턴스를 저장해야 하므로 사용하는 것 같다. [[참고]](https://docs.nestjs.com/fundamentals/injection-scopes)
  - 실제 instance는 `InstancePerContext.instance`에 저장된다.
- 그 밖에 `token`, `host`, `metatype`, `instance`가 저장돼있는 걸 볼 수 있다.
  - 특이했던 점은 `InstanceWrapper.instance` 안에는 prototype instance가 존재하는데, 실제 instance를 생성하기 전에 prototype instance를 먼저 다 생성하고, 그 다음 try-catch 구문 안에서 실제 instance를 생성하고, 실패하면 graph inspect 과정을 거쳐 사용자에게 에러를 보여주는 식인데 아마 graph를 먼저 생성하려고 prototype을 넣어주는 것 같다. (여기서 말하는 graph는 module들의 dependency를 바탕으로 생성된 `SerializedGraph`이다. 자료구조 수업시간에 배우던 edge, node 등을 가진 graph 구조다.) *진짜 확실하지 않음*

    ```tsx
    // packages/core/injector/instance-loader.ts
    export class InstanceLoader<TInjector extends Injector = Injector> {
    
     ...
    
     public async createInstancesOfDependencies(
        modules: Map<string, Module> = this.container.getModules(),
      ) {
        this.createPrototypes(modules);
    
        try {
          await this.createInstances(modules);
        } catch (err) {
          this.graphInspector.inspectModules(modules);
          this.graphInspector.registerPartial(err);
          throw err;
        }
        this.graphInspector.inspectModules(modules);
      }
    
     ...
    
    }
    ```

## Point of view

위 그림들은 필자가 전체적인 구조만 파악하기 위해 코드를 대충대충 본 부분이고, 보면서 재밌었던 부분들을 소개하고자 한다.

### never type

typescript에서 never 처음 봤을 때 *이거 어따 쓰는 거지* 라는 생각했었는데 쓰이는 거 처음 봤다. 이해하기 쉬우라고 주석도 같이 가져와봤다. [여기](https://ui.toast.com/posts/ko_20220323#%ED%83%80%EC%9D%B4%ED%95%91%EC%9D%84-%EB%B6%80%EB%B6%84%EC%A0%81%EC%9C%BC%EB%A1%9C-%ED%97%88%EC%9A%A9%ED%95%98%EC%A7%80-%EC%95%8A%EB%8A%94%EB%8B%A4)서의 의미랑 비슷하게 쓰였다.

```tsx
// packages/common/interfaces/modules/provider.interface.ts
export interface ValueProvider<T = any> {
  /**
   * Injection token
   */
  provide: InjectionToken;
  /**
   * Instance of a provider to be injected.
   */
  useValue: T;
  /**
   * This option is only available on factory providers!
   *
   * @see [Use factory](https://docs.nestjs.com/fundamentals/custom-providers#factory-providers-usefactory)
   */
  inject?: never;
}
```

### getting dependencies of class

5편에서 언급한 방법으로 dependency들을 가져오는 부분이다. [[5편 참고]](/knowledge/2023/05/01/NestJS5-Decorator-reflection.html#designparamtypes)

```tsx
// packages/core/injector/injector.ts
export class Injector{

 ...

 public getClassDependencies<T>(
     wrapper: InstanceWrapper<T>,
   ): [InjectorDependency[], number[]] {
     const ctorRef = wrapper.metatype as Type<any>;
     return [
       this.reflectConstructorParams(ctorRef),
       this.reflectOptionalParams(ctorRef),
     ];
   }
 
 ...
 
 public reflectConstructorParams<T>(type: Type<T>): any[] {
     const paramtypes = [
    // 요 부분이 constructor parameter를 가져옴
       ...(Reflect.getMetadata(PARAMTYPES_METADATA, type) || []),
     ];
     const selfParams = this.reflectSelfParams<T>(type);
 
     selfParams.forEach(({ index, param }) => (paramtypes[index] = param));
     return paramtypes;
   }

 ...

}
```

`design:paramtypes` 키를 이용해서 constructor parameter 목록을 가져오는 것을 볼 수 있다.

`type`으로 1편의 `DogService`를 넘겨줬다고 생각하면 `reflectConstructorParams`는  `[ class CatService ]` 를 반환한다. [[1편 참고]](/knowledge/2023/05/01/NestJS1-What-is-NestJS.html#dependency-injection-in-nestjs)

`reflectSelfParams`는 `Inject` 데코레이터로 따로 token을 지정해준 dependency만 `index`와 `param`(token)을 가져와서 다시 지정해 준다.

### how providers are instantiated with done promise

`InstanceWrapper`의 `InstancePerContext`에 6편에서 알아본 `donePromise`라는 익숙한 이름이 있다. NestJS에서는 dependency들의 instance가 생성될 때까지 기다려야 할 때 이 방법이 쓰인다. 우선 NestJS에서 instance들이 생성될 때 무슨 과정을 거치는 지 말해보자면

- module마다 동시에 다음을 실행한다.
  - module의 provider마다 동시에 다음을 실행한다. (`Injector.loadInstance`)
        1. provider의 dependenency 목록을 가져온다. (getting dependencies of class)
        2. 만약 dependency들 중 아직 생성되지 않은 **dependency가 있다면 instantiate될 때까지 기다렸다(`Injector.loadInstance`가 재귀적으로 호출된다.)**가 provider를 instantiate한다.

하나의 provider가 생성되는 걸 다른 여러개의 provider들이 기다려야 하는 상황이 있을 수 있다.

이때 dependency들마다 `Injector.loadInstance`가 호출되면서 **각 dependency가 instantiate될 때까지 기다리는데** 그게 이 부분이다. (`2`번)

```tsx
// packages/core/injector/injector.ts
export class Injector{

 ...

 public async resolveComponentHost<T>(
    moduleRef: Module,
    instanceWrapper: InstanceWrapper<T | Promise<T>>,
    contextId = STATIC_CONTEXT,
    inquirer?: InstanceWrapper,
  ): Promise<InstanceWrapper> {
    const inquirerId = this.getInquirerId(inquirer);
    const instanceHost = instanceWrapper.getInstanceByContextId(
      this.getContextId(contextId, instanceWrapper),
      inquirerId,
    );
    if (!instanceHost.isResolved && !instanceWrapper.forwardRef) {
      await this.loadProvider(
        instanceWrapper,
        instanceWrapper.host ?? moduleRef,
        contextId,
        inquirer,
      );
    }

  ...

 }

 ...

}
```

이건 `loadInstance` 부분이다.

```tsx
// packages/core/injector/injector.ts
export class Injector {

 ...

 public async loadInstance<T>(
    wrapper: InstanceWrapper<T>,
    collection: Map<InstanceToken, InstanceWrapper>,
    moduleRef: Module,
    contextId = STATIC_CONTEXT,
    inquirer?: InstanceWrapper,
  ) {
    const inquirerId = this.getInquirerId(inquirer);
    const instanceHost = wrapper.getInstanceByContextId(
      this.getContextId(contextId, wrapper),
      inquirerId,
    );
    if (instanceHost.isPending) {
      return instanceHost.donePromise.then((err?: unknown) => {
        if (err) {
          throw err;
        }
      });
    }
    const done = this.applyDoneHook(instanceHost);

  ...

  // instantiate

 }

 ...

 public applyDoneHook<T>(
    wrapper: InstancePerContext<T>,
  ): (err?: unknown) => void {
    let done: (err?: unknown) => void;
    wrapper.donePromise = new Promise<unknown>((resolve, reject) => {
      done = resolve;
    });
    wrapper.isPending = true;
    return done;
  }

 ...

}

```

예시 상황으로 `DogService`가 `CatService`를 기다린다고 하자. [1편에서 설명한 과정](/knowledge/2023/05/01/NestJS1-What-is-NestJS.html#dependency-injection-in-nestjs)을 다시 자세하게 풀어쓴 것이다.

- `DogService`대상으로 `loadInstance`와 `CatService`대상으로 `loadInstance`가 동시에 실행되고,
  - `CatService`는 바로 생성될 수 있다.
    - `CatService`의 `instanceHost`(타입은 `InstancePerContext`)에 `isPending`을 true로, `donePromise`를 걸어놓는다.
    - 바로 instantiate하고
    - `done` 호출해서 `CatService`가 instantiate됐음을 외부에 알린다.
  - `DogService`는 `CatService`가 실행될 때까지 기다려야 한다. (`DogService`의 dependency마다 `resolveComponentHost`가 동시에 호출된다.)
    - `DogService`의 `instanceHost`에 `isPending`을 true로, `donePromise`를 걸어놓는다.
      - `instanceWrapper`로 `CatService`를 담고 있는 `InstanceWrapper`를 넘겨주며 `loadProvider` 호출
      - `loadProvider`는 바로 `loadInstance`를 호출하는데, 여기서 dependency가 instantiate될 때까지, 즉 `donePromise`가 resolve될 때까지 (`CatService`에서 `done` 호출할 때까지) 기다린다.
    - 모든 dependency가 instantiate될 때까지 기다리면 `DogService`가 instantiate된다
    - `done`을 호출해서 `DogService`도 instantiate됐음을 외부에 알린다.

## why module token is not accessible

그럼, 왜 module token은 사용자가 지정도 못하고 접근도 못할까? 이 질문은 *왜 dependency는 module 단위로 관리되는가* 라는 의문으로 이어지는 것 같다. module token을 지정할 수 있으면 module에 token 하나를 지정해놓고 아무데서나 import하면 그만이기 때문이다. (static module은 module 클래스 이름만 쓰면 아무데서나 import할 수 있지만 dynamic module에서는 이게 불가능하다.)

질문을 *module을 없애버리고 provider를 한 번 선언하면 아무데서나 그걸 사용할 수 있게 하면 안되는 걸까* 로 바꿔보자.

### why module scope

***뇌피셜 주의***

application의 규모가 커지면 circular dependency가 생길 수 있다. 그걸 해결하려면 정말 복잡할 것이다. circular dependency를 nest doc에서도 언급하고 있고 이를 해결할 방법을 제안했지만, 최후의 수단으로 쓰라는 말투로 얘기하고 있다. 결국 DI는 circular dependency를 푸는 게 중요한 목적 중 하나이다. [[참고]](https://docs.nestjs.com/fundamentals/circular-dependency)

그림으로 살펴보자.

![global scope](/assets/images/post/2023-05-01-NestJS8-How-NestJS-works-20230502000544.png)

동그라미를 provider로 보고, 파란색 세모를 특정 역할을 하는 하나의 단위(잠시 후에 module class에서 큰 module로 묶일 애들)라고 보자. 사실상 파란색 세모는 없는 거나 마찬가지가 된다. 그때그때 provider를 찾아서 하나씩 import시켜줘야 하기 때문이다. dependency graph의 edge도 많아지게 되며 규모가 커질수록 circular dependency를 추적하기가 버거워진다.

![module scope](/assets/images/post/2023-05-01-NestJS8-How-NestJS-works-20230502000630.png)

작은 module 단위(흰색 네모)로 묶게 되면 그나마 구조를 이해하기 괜찮아진다. (파란색 세모를 하나의 큰 module이고, 흰색 네모를 TypeOrmModule의 forFeature 같은 메소드로 생성된 dynamic module이라고 생각하자.) 모든 import는 module을 통해 이루어져야 하며, dynamic module을 그때그때 만들어서 필요한 provider를 import할 수 있다. edge수도 이전보다 적어졌다.

NestJS가 Angular에서 영감을 많이 받았다고 얼핏 들은 기억이 있다. Angular에서 provider는 기본적으로 global scope이다. [[참고]](https://angular.io/guide/providers#provider-scope) 첫번째 그림과 같은 상황인 것이다. NestJS 개발자는 이 상황을 개선하려고 module scope로 만든 게 아닐까? chat gpt에게 비슷한 내용을 질문한 걸 올리며 마무리하겠다.

> Q. Angular의 provider는 기본적으로 global scope지? NestJS에서는 module scope고
>
> A. 맞습니다. Angular에서의 Provider는 기본적으로 애플리케이션 전역에서 사용 가능한 Singleton 객체로 생성됩니다. 이는 Angular의 Injector에 등록된 Provider가 애플리케이션 전역에서 공유되기 때문입니다. 하지만 이를 제한하기 위해서는 Angular의 Hierarchical Dependency Injection 시스템을 이용하여 Provider를 특정 모듈에서만 사용할 수 있도록 구성할 수 있습니다.
>
>
> 반면, NestJS에서의 Provider는 모듈 스코프로 생성됩니다. 즉, Provider는 해당 모듈에서만 사용 가능한 Singleton 객체로 생성되며, 다른 모듈에서는 해당 Provider를 직접적으로 사용할 수 없습니다. 이는 NestJS의 모듈 시스템을 통해 의존성을 관리하고, 모듈 간의 결합도를 낮추기 위함입니다. NestJS에서는 모듈마다 자체적으로 Injector를 가지며, 해당 모듈에서 등록한 Provider만이 해당 Injector에서 사용 가능합니다.
>
> 따라서, Angular과 NestJS에서의 Provider는 Scope가 다르며, 이를 이용하여 각각의 프레임워크에서 의존성을 관리하고, 모듈 간의 결합도를 조절할 수 있습니다.
>

## Reference

- [nest github](https://github.com/nestjs/nest)
- [never 타입 관련 블로그 글](https://ui.toast.com/posts/ko_20220323#%ED%83%80%EC%9D%B4%ED%95%91%EC%9D%84-%EB%B6%80%EB%B6%84%EC%A0%81%EC%9C%BC%EB%A1%9C-%ED%97%88%EC%9A%A9%ED%95%98%EC%A7%80-%EC%95%8A%EB%8A%94%EB%8B%A4)
- [angular doc - provider scope](https://angular.io/guide/providers#provider-scope)
