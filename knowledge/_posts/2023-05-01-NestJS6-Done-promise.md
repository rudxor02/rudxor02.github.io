---
layout: post
type: tags
title: "[NestJS 6편] Done promise"
date: 2023-05-01 14:31:00 +0900
comments: true
toc: true
tags: nestjs typescript javascript
---


NestJS에서 소스코드 보다가 정말 신선했던 부분이 있었는데, 이게 NestJS를 개발한 사람이 혼자 이 방법을 생각해내서 사용한건지, 아니면 많이 쓰이는 방법인데 필자만 몰랐던 건지 모르겠지만 아무튼 되게 똑똑한 방법이고 활용성 높은 코드라고 느껴서 이렇게 따로 글을 써봤다. *나중에 알게 된 사실인데 테스트 패키지인 jest에서도 똑같아 보이는 걸 봤다. [[링크]](https://jestjs.io/docs/asynchronous#callbacks) done promise는 여러 곳에서 많이 쓰이는 것 같다.*

os를 공부하다 보면 여러 개의 프로세스가 하나의 공유 자원에 접근해야 할 때 mutual exclusion이라는 개념을 적용할 때가 있는데, 이 알고리즘은 다음과 같은 특징을 가지고 있다. *(필자가 추상적으로 생각하는 걸 그대로 적은 것이므로 절대 정확한 게 아니다!)*

1. 하나의 프로세스만이 공유 자원에 접근할 수 있다.
2. 공유 자원에 접근중인 프로세스가 작업을 끝내면 끝났다는 것을 알려주는 신호를 보낸다.
3. 기다리는 프로세스들 중 하나의 프로세스가 공유자원에 접근할 수 있는 기회를 받게 되고 `2`로 다시 돌아간다.

바로 밑에서 언급할 done promise는 처음 봤을 때 mutual exclusion 개념과 레벨은 다르지만 `2`번을 typescript 코드로 구현했다는 느낌을 많이 받았다. NestJS에서는 dependency가 있는 클래스를 instantiate하기 위해서는 당연하게도 우선 dependency들이 instantiate되는 걸 기다려야 하는데, 이때 done promise가 쓰였다. 이제 done promise가 무엇인지 알아보자.

## Done promise

done promise와 비교하기 위해 일반적으로 promise에서 `resolve`를 사용하는 방법을 살펴보겠다.

```tsx
const a = new Promise<unknown>((resolve) => {
  setTimeout(() => {
    console.log('3 seconds passed');
    resolve(3);
  }, 3000);
});

a.then((value: unknown) => {
  console.log(`done, received ${value}`);
});
```

위 코드를 실행하면 3초 뒤에 콘솔이 찍히게 된다. done promise는 `resolve`를 promise 정의하는 부분에서 사용하지 않고 외부에서 사용할 수 있도록 `done`이라는 변수에 저장한다. 

```tsx
let done: (value: unknown) => void;

const a = new Promise<unknown>((resolve) => {
  done = resolve;
  // setTimeout(() => {
  //   console.log('3 seconds passed');
  //   resolve(3);
  // }, 3000);
});

setTimeout(() => {
  console.log('3 seconds passed');
  done(3);
}, 3000);

a.then((value: unknown) => {
  console.log(`done, received ${value}`);
});
```

지금은 기다리는 애가 하나여서 `resolve`를 외부로 뺀 게 별 쓸모가 없어보이는데, 아래 코드를 보자.

```tsx
let done: (value: unknown) => void;

const a = new Promise<unknown>((resolve) => {
  done = resolve;
});

const AWaitsaPromise = async (a: Promise<unknown>) => {
  // waiting a done
  const value = await a;
  console.log(`done, A received ${value}`);
  // keep doing something
};

const BWaitsaPromise = async (a: Promise<unknown>) => {
  // waiting a done
  const value = await a;
  console.log(`done, B received ${value}`);
  // keep doing something
};

const CResolvesaPromiseAfter3Seconds = () => {
  setTimeout(() => {
    console.log('3 seconds passed');
    done(3);
  }, 3000);
};

AWaitsaPromise(a);
BWaitsaPromise(a);
CResolvesaPromiseAfter3Seconds();
```

**`C`가 `a`라는 공유자원에 작업중이고 `A`와 `B`가 이를 기다리고 그 다응에 어떤 코드를 실행해야 하는 상황**이라고 생각하면 위 코드가 이해하기 쉬울 것이다. 실행하면  다음과 같은 결과가 나온다.

```bash
3 seconds passed
done, A received 3
done, B received 3
```

## Reference

- [js korean doc - 일반적인 promise 예시](https://developer.mozilla.org/ko/docs/Web/JavaScript/Reference/Global_Objects/Promise#%EC%98%88%EC%A0%9C)
- [jest doc - testing asynchronous code](https://jestjs.io/docs/asynchronous)