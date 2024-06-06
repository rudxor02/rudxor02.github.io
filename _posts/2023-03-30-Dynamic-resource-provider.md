---
layout: post
type: tags
title: "[Pulumi] Dynamic Resource Provider"
date: 2023-03-30 23:30:55 +0900
comments: true
toc: true
tags: pulumi
---

_pulumi에는 dynamic resource provider 라는 게 존재한다. 리소스의 lifecycle을 직접 관리할 수 있는데, 여러모로 편리하고 확장성 있는 기능이다. (terraform에는 없다고 한다.) 원래 이런 게 있는지도 몰랐었는데 회사에서 좀 특별한? 상황이 생겨서 이러한 기능이 필요해서 찾다 보니 발견하게 됐다._

## Dynamic resource provider란

우선 resource provider가 뭔지부터 짚고 넘어가면, 말 그대로 pulumi에서 resource를 제공해 주는 역할을 한다. 예를 들면 다음과 같은 s3 bucket이 있다고 하자.

```jsx
import * as aws from "@pulumi/aws";

const testBucket = new aws.s3.Bucket("test-bucket", {});
```

_어? bucket 이름을 지정 안 해줬는데 어떻게 생성되는 거지?_ 라고 생각할 수 있는데 `aws.s3.Bucket` 생성자의 2번째 input에 이 글의 마지막에 나올 `bucket`이란 속성을 명시하지 않으면 자동으로 bucket 이름은 `test-bucket-a01b82` 처럼 1번째 input에 무작위 번호를 붙인 이름이 된다. [[링크]](https://www.pulumi.com/registry/packages/aws/api-docs/s3/bucket/#bucket_nodejs)

- `pulumi up` 을 처음 실행하게 되면 `provider`(여기서는 aws provider)는 `deployment engine`으로부터 이 bucket을 `create` 해달라는 요청을 받게 된다. 그러면 bucket을 생성하는 aws api를 코드 내용(이름이나 설정값 등)에 맞게 호출한다.
- 코드를 바꾸지 않고 또 실행하면 `provider`는 위 bucket이 이전 상태와 달라진 것은 없는 지 `diff`를 확인하고, 달라진 게 없음을 확인하고 아무것도 하지 않는다.
- bucket의 설정값을 바꾸고 실행하면 (예를 들면 `tags`) `provider`는 위 bucket에 대하여 `diff`를 확인하고, 달라진 게 있음을 확인하고 `update` 하는 aws api를 코드 내용에 맞게 호출한다.
- 코드를 지우고 실행하면 `provider`는 위 bucket을 삭제하는 `delete` 하는 aws api를 호출한다.
- 만약에 `test-bucket` 이라는 이름을 바꾸고 실행하면, bucket identifier (bucket 이름) 가 달라져야 한다. `update`만으로는 bucket identifier를 바꿀 수 없으므로 (aws에서 한 번 생성된 bucket은 이름 변경이 안 된다.), `provider`는 위 bucket을 `replace` 하는 동작을 한다. 먼저 다른 bucket을 `create`하고, 그 다음 원래 있던 bucket을 `delete`하는 aws api를 호출한다.

위와 같이 `provider`는 리소스가 `create`, `diff`, `update`, `delete`, `replace` 될 때 정확히 어떻게 동작해야 할 지 전체 리소스의 lifecycle을 관리해 주는 역할을 한다. 예시에서는 각 과정에서 aws provider가 aws api 중 bucket에 대한 api를 호출한다고 짐작해 볼 수 있다.

`dynamic resource provider`는 리소스의 lifecycle에 개발자가 직접 개입할 수 있게 pulumi에서 만든 개념이다. 이 글에서는 aws provider가 그랬던 것처럼 특정 리소스에 대해 lifecycle을 관리하는 걸 dynamic resource provider를 통해 직접 설정해 볼 것이다. 참고로 aws provider와 같이 패키지로 import해서 쓸 수 있는 `provider`는 그냥 `resource provider`이다. [[링크]](https://www.pulumi.com/docs/intro/concepts/resources/providers/#resource-providers)

## How to use

_참고로 이 글에서 bucket에 한해서 언급한 이름, bucketName, bucketIdentifier 다 사실은 같은 개념을 말한다. 필자가 input과 output을 구분하려고 이름을 저렇게 지은 것이다._

이 기능을 활용하면 꽤 많은 것들을 할 수가 있는데, 그 중에서 리소스의 output이 바뀔 때마다 뭔가를 하고싶다! 할 때 쓸 수 있는 코드를 작성해 보려 한다. 특정 bucket 이름이 바뀌면 로그를 출력하는 기능이다. 사실 막상 써보려니 쓸모있는게 안떠오른다…

### interface

`bucket_logger.interface.ts` 에서는 후에 쓸 interface를 정의하는데, 다른 s3 bucket의 이름을 `bucketLogger`의 `bucketIdentifier`로 넣어주고(input), 그대로 `bucketLogger`의 `bucketName`으로 전해줄 것이다.(output)

`BucketLoggerInputs` 는 실제 리소스를 정의할 때 사용하고, `BucketLoggerProviderInputs`는 provider 안에서 사용하는 interface이다. provider 안에서는 pulumi.Input이 벗겨진 상태로 사용이 가능해 값을 비교하거나 연산을 쉽게 할 수 있다.

```tsx
// bucket_logger.interface.ts

import * as pulumi from "@pulumi/pulumi";

export interface BucketLoggerInputs {
  bucketIdentifier: pulumi.Input<string>;
}

export interface BucketLoggerProviderInputs {
  bucketIdentifier: string;
}

export interface BucketLoggerOutputs {
  bucketName: pulumi.Output<string>;
}

export interface BucketLoggerProviderOutputs {
  bucketName: string;
}
```

### provider

`bucket_logger.provider.ts`에서는 provider를 정의하는데, 앞에서 말한 `diff`, `update`, `create` 등의 예약된 이름의 메소드들이 정의돼 있는 걸 볼 수 있다. [[링크]](https://www.pulumi.com/docs/intro/concepts/resources/dynamic-providers/#checkolds-news)

- `check` 메소드는 provider가 리소스를 참조할 때 항상 호출되는 함수이며, 위 링크에서 보면 후에 호출되는 메소드들에게 값을 전달해주는 역할을 한다고 나와있다. 보통은 정의 안하거나 밑에처럼 그대로 input들을 넘겨준다.
- `create` 메소드는 리소스를 생성할 때 호출되며 리소스의 id와 output을 정의할 수 있다.
- `diff` 메소드는 리소스의 상태를 검사할 때 호출되며, return 객체가 `{ changes: true }`이면 `update` 또는 `replace`를 호출한다.
  - 여기서는 안 나왔지만 `{ changes: true, replaces: [’bucketIdentifier’] }` 처럼 `replaces`에 값이 바뀐 input의 프로퍼티 이름을 적으면 `replace`가 호출된다.) 만약 return 객체가 `{ changes: false }`이면 그냥 아무것도 안 한다.
- `update` 메소드는 `diff`로부터 조건부로 호출되며, 이때 return 값으로 새롭게 바뀔 리소스의 output을 넘겨준다. 예시에서는 이때가 bucket 이름이 바뀔 때이므로, 로그를 출력해 주는 모습을 확인할 수 있다.
- `delete` 메소드는 리소스가 지워질 때 호출된다.

```tsx
// bucket_logger.provider.ts

import * as pulumi from "@pulumi/pulumi";
import {
  BucketLoggerInputs,
  BucketLoggerProviderInputs,
  BucketLoggerOutputs,
  BucketLoggerProviderOutputs,
} from "./bucket_logger.interface.ts";

export const bucketLoggerProvider: pulumi.dynamic.ResourceProvider = {
  async check(
    olds: BucketLoggerProviderOutputs,
    news: BucketLoggerProviderInputs
  ): Promise<pulumi.dynamic.CheckResult> {
    return { inputs: { ...news } };
  },

  async create(
    inputs: BucketLoggerProviderInputs
  ): Promise<pulumi.dynamic.CreateResult> {
    return {
      id: "<random-id>",
      outs: {
        bucketName: inputs.bucketIdentifier,
      },
    };
  },

  async diff(
    id: string,
    olds: BucketLoggerProviderOutputs,
    news: BucketLoggerProviderInputs
  ): Promise<pulumi.dynamic.DiffResult> {
    if (olds.bucketName !== news.bucketIdentifier) {
      return { changes: true };
    }
    return { changes: false };
  },

  async update(
    id: string,
    olds: BucketLoggerProviderOutputs,
    news: BucketLoggerProviderInputs
  ): Promise<pulumi.dynamic.UpdateResult> {
    // custom update logic
    pulumi.log.info(
      `Bucket ID Has Changed: ${olds.bucketName} -> ${news.bucketIdentifier}`
    );

    return {
      outs: {
        bucketName: news.bucketIdentifier,
      },
    };
  },

  async delete(id: string): Promise<void> {
    pulumi.log.info(`Bucket Logger ${id} Deleted`);
    return;
  },
};
```

### resource

`bucket_logger.resource.ts`에서는 `BucketLogger` 클래스를 정의하며, 생성자에서 저렇게 리소스에 기본으로 할당될 provider를 연결할 수 있다. `bucketName`을 인스턴스 변수로 지정해줌으로써 output을 할당할 수 있다.

```tsx
// bucket_logger.resource.ts

import * as pulumi from "@pulumi/pulumi";
import { bucketLoggerProvider } from "./bucket_logger.provider.ts";
import { BucketLoggerInputs } from "./bucket_logger.interface.ts";

export class BucketLogger extends pulumi.dynamic.Resource {
  public readonly bucketName!: pulumi.Output<string>;

  constructor(
    name: string,
    props: BucketLoggerInputs,
    opts?: pulumi.CustomResourceOptions
  ) {
    super(bucketLoggerProvider, name, props, opts);
  }
}
```

### instantiate

밑에는 필요한 걸 다 정의 했으니 `index.ts`에서 사용하는 모습이다. (여기서 `testBucket`에 들어가는 `bucket` 은 bucket name이다. 그리고 이 속성이 바뀌면 aws provider는 `replace` 작업을 진행한다.)

1. `testBucket`의 이름(`<some-identifier>`)이 바뀐다고 하자.
2. s3 bucket resource의 특성상 identifier가 바뀌면 `replace`를 해야 하므로, `testBucket`도 aws provider가 `replace` 작업을 진행할 것이다.
3. 이때 `testBucketLogger`의 input으로 들어가는 `bucketIdentifier`도 달라진다.
4. `bucketLoggerProvider`가 `testBucketLogger`에 대해 `update` 작업을 진행한다.
5. 더 말하자면 이때 `testBucketLogger`의 output인 `bucketName`도 바뀌게 된다. 쓸모없지만 그냥 정의해 봤다…

```tsx
// index.ts

import * as aws from "@pulumi/aws";
import { BucketLogger } from "./bucket_logger.resource.ts";

const testBucket = new aws.s3.Bucket("test-bucket", {
  bucket: "<some-identifier>",
});

const testBucketLogger = new BucketLogger("test-bucket-logger", {
  bucketIdentifier: testBucket.bucket,
});
```

당장에 생각이 안 떠올라 정말정말 쓸모없는 기능을 구현해봤지만, 필자는 pulumi의 dynamic resource provider 기능이 충분히 확장성 있고 편하게 쓰일 수 있는 기능이라고 생각한다.

## Reference

- [pulumi doc - dynamic resource provider](https://www.pulumi.com/docs/intro/concepts/resources/dynamic-providers/)
- [pulumi doc - aws s3 bucket api](https://www.pulumi.com/registry/packages/aws/api-docs/s3/bucket/#aws-s3-bucket)
