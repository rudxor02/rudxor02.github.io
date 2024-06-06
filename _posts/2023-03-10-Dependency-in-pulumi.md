---
layout: post
type: tags
title: "[Pulumi] Dependency in pulumi"
date: 2023-03-10 23:30:55 +0900
comments: true
toc: true
tags: pulumi
---

오늘(230307) `pulumi destroy`를 해서 회사에서 동작하는 개발용 서비스를 중지시켰다. pulumi cli가 내부적으로 어떻게 동작하는지 몰라서 일어난 일이다. 원래 있던 리소스의 파라미터 값을 편하게 가져오기 위해 `pulumi import`를 사용해서 코드를 짰으나 이게 pulumi 서버와 연동되는 줄 몰랐다.

이때는 단순히 터미널상에서 정보를 가져와서 코드만 짜주는 기능으로만 생각했었다. 그래서 `pulumi up`을 했는데 동작이 이상한걸 느껴서 _어 그럼 삭제했다가 다시 만들어야겠다 → ???_

`pulumi import` 를 하고 나서 리소스에 대한 metadata를 지워주는 명령어인 `pulumi stack delete <urn>` 를 해줬어야 했는데 안 그래줘서 실제 import한 리소스들을 갖고 동작하고 있었던 거였다

## Resource metadata

`pulumi import`를 하면 내부적으로 pulumi 서버와 어떤 동작을 하는지 알아야겠다는 생각이 들었다. (사실 모르고 쓴 게 더 문제…) 그래서 내부 구조를 알아야겠다 싶어서 찾다 찾다 pulumi가 서버랑 어떻게 통신하는지 [여기](https://www.pulumi.com/docs/intro/concepts/state/)서 설명해주는 글을 찾았다.

위 문서를 읽다가 metadata가 정확히 어떻게 저장되는지 알고싶어 `pulumi stack export` 결과를 파일로 저장해서 뜯어봤다. 리소스 하나당 대략적으로 이렇게 저장되는 듯 하다. (참고로 리소스 종류는 aws subnet)

```json
{
 "urn": "pulumi 안에서의 urn",
 "custom": true,
 "id": "aws 상의 resource id",
 "type": "pulumi aws 패키지 상의 클래스명",
 "inputs": {
     "__defaults": [
    "이거는 pulumi aws 소스코드 안에 명시적으로 지정돼서 나오는건지 잘 모르겠다.",
    ...
   ],
   ...
  },
 "outputs": {...},
 "parent": "stack urn",
 "dependencies": [
     "이 리소스와 dependency 관계를 가지는 다른 리소스의 urn",
   "내부적으로는 이렇게 dependency가 등록되는 듯 하다"
 ],
 "provider": "pulumi provider urn",
 "propertyDependencies": {
     "availabilityZone": null,
     "cidrBlock": null,
     "mapPublicIpOnLaunch": null,
     "privateDnsHostnameTypeOnLaunch": null,
     "vpcId": [
         "vpc urn"
    ]
 }
}
```

`propertyDependencies` 를 잘 보면 inpurtArgs의 property만을 뽑아온 것임을 알 수 있는데, 리소스를 생성할 때 내부적으로 이렇게 `dependencies`를 명시해 주는 듯 하다. 그래서 `pulumi destroy` 를 하게 되면 dependency가 없는 것부터 먼저 지우는 방식으로 추측된다.

## Dependency in pulumi

pulumi 플랫폼이 초기에는 특정 리소스만을 삭제하는 기능이 없었다고 한다. ([공식문서](https://www.pulumi.com/docs/reference/cli/pulumi_destroy/)에도 `pulumi destroy` 가 전체 리소스를 지우는 명령어라고 나오고 특정 리소스를 지우는 기능은 flag option을 자세히 보아야 나온다. _있는지도 이거 쓰면서 알았음._ 뭔가 주요 기능은 아닌 느낌?)

특정 리소스만을 삭제하는 기능은 처음 레포가 만들어지고 3년이나 지나고서야 [기능이 추가](https://github.com/pulumi/pulumi/pull/3244)됐다. 그것도 issue 내용을 읽어보면 알겠지만 먼가? 임시방편으로 한 느낌이 강하게 든다. delete 작업을 하면 resource replace로 인식을 해서 delete-before-replace 라는 동작을 응용해서 삭제작업을 진행한다. 특정 리소스 삭제기능을 새로 만든 게 아니라 다른 기능 위에 얹은 기능이라는 뜻

심지어 [초기 버전](https://github.com/pulumi/pulumi/blob/master/pkg/resource/deploy/step_generator.go#L947)에서는 dependency check 안하고 마구잡이로 delete request를 보냈다고 한다…_(`pulumi destroy` 가 특정 리소스 삭제를 한동안 지원하지 않았던 배경인듯 하다. 애초에 이 기능을 빼놓고 플랫폼을 설계했는데 나중에 추가하려니 골치가 아팠던 것)_

그래도 정말 편리한 기능인데 (dependency를 명시적으로 선언 안해줘도 pulumi는 잘 돌아가기 때문) 어떻게 보면 독이 된다. resource delete를 할 때 dependency check를 내부적으로 해줘야 하기 때문이고 _(다른 얘기긴 한데 ),_ [소스 코드 주석](https://github.com/pulumi/pulumi/blob/master/pkg/resource/deploy/step_generator.go#L1738)을 보면 dependency problem을 완전히 풀었다고 하기는 어렵다. resource들의 inputArg만을 보고 참조하냐 안하냐를 판단해서 해결하는 방법이기 때문… 코드를 완전히 보지는 않았지만 아마 이때 `propertyDependencies` property가 사용되는 듯 하다

실제로 다음과 같은 리소스가 삭제가 안됐던 경험이 있다.

```tsx
const testDefaultSG= new aws.ec2.DefaultSecurityGroup(
  `test-default-sg`,
  {
  ...,
    ingress: [
      {
        fromPort: 0,
        protocol: '-1',
        securityGroups: [
     someSecurityGroup.id
    ],
        self: true,
        toPort: 0,
      },
      ...
    ],
    ...
  }
);
```

metadata의 `propertyDependencies` property에 `someSecurityGroup.urn`이 있어서 `pulumi destroy` 를 실행해서 리소스를 지우려고 하면 2개 리소스 중 어느 하나 삭제되지 않고 무한정 응답만 기다리는 상태가 된다.

이를 해결하려면 aws console에서 위에 나온`sampleDefaultSecurityGroup` 의 inbound rule을 먼저 지워줘야 삭제가 된다. 그전까지는 stack의 metadata 상에서 보면 해당 rule은 관리 대상이 아니기 때문에 삭제를 할 수 없고, 따라서 2개 리소스 어느 것 하나 지울 수 없다. 해보지는 않았지만 `securityGroupRule` 리소스를 바깥에서 한번 정의해서 inbound rule에 넣어주면 아마 삭제가 될 듯 하다.

저렇게 rule 안에서 security group을 참조한다면 자동으로 따로 `securityGroupRule` 리소스를 만들어서 등록해줘야 하는데 inputArg만을 체크해서 `propertyDependencies` 에 리소스를 등록만 해두는 방식을 사용해서 이러한 문제가 발생한 듯 하다.

> 결론은 원래 존재하는 클라우드의 자원들의 파라미터만을 가져오고 싶다면 `pulumi import` 후 `pulumi stack delete <urn>`을 잘 해주자… 그리고 dependency 잘 신경쓰며 작업하자

## Reference

- [pulumi repo](https://github.com/pulumi/pulumi)
- [pulumi backend](https://www.pulumi.com/docs/intro/concepts/state/)
- [pulumi cli destroy](https://www.pulumi.com/docs/reference/cli/pulumi_destroy/)
