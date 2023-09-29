---
layout: post
type: tags
title: "[AWS] Vpn with service discovery and vpc peering"
date: 2023-09-21 21:00:00 +0900
comments: true
toc: true
tags: aws openvpn vpn vpc ecs service-discovery vpc-peering
---

*오늘은 vpn을 aws에서 사용할 때 service discovery로 ecs container에 접근할 수 있도록 ~~삽질~~세팅했던 경험을 공유하고자 한다. (짧음)*

## 문제 상황

pulumi 관련 태스크를 처음 맡으신 흰수염고래님의 업무를 도와드리다가 발생한 일이다.

1. vpc1에 openvpn 서버를 ec2로 구성해놓은 상태이다
    1. vpc1의 ip대역은 192.168.0.0/16라고 하자.
2. vpc2에 ecs 서비스를 띄워놨고, 웹 콘솔 안에 중요한 정보가 담겨있어 vpc를 켜야 접속할 수 있도록 세팅이 필요하다.
    1. vpc2의 ip대역은 172.16.0.0/12라고 하자.
    2. vpc1과 vpc2는 vpc peering으로 연결이 돼있다.
        1. 여기서 vpc perring을 설명하자면, 원래는 서로 다른 vpc끼리는 말 그대로 가상 **사설** 네트워크이므로, 통신이 불가능하지만 ip대역이 다른 vpc에 한해서 vpc peering을 할 수 있다. 그러면 vpc1의 192.168.0.5의 호스트와 172.16.0.4의 호스트는 통신이 가능해진다. ([참고1](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html))
    3. 이때 서비스 보안그룹의 ingress rule에는 웹 콘솔 포트에 192.168.0.0/16 ip대역을 허용해놨다. 즉, vpc1에서 접속할 수 있도록 설정해놨다.
3. ecs 서비스 컨테이너는 켜질 때마다 다른 public ip가 할당이 되어, service discovery 설정을 해놨다.
    1. 설정해놓은 도메인을 `example.internal`이라고 하자.
    2. service dicovery란 ecs 서비스에 사설 도메인을 연결할 수 있는 기능이다. 만약에 서비스 용량이 3이라서 task 1, 2, 3을 띄워서 172.16.0.6, 172.16.0.7, 172.16.0.8의 ip를 각각 가진다고 하면 `example.internal`로 접속하면 저 셋 중 하나로 도메인주소가 ip주소로 resolve된다. ([참고2](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)) 원래는 없었으나 load balancer를 매번 연결하는 게 불편하다고 느꼈는지 비교적 최근(2018)에 나온 기능이다. ([참고3](https://aws.amazon.com/ko/blogs/korea/amazon-ecs-service-discovery/))

이제 vpn을 켜면 나의 맥북은 가상으로 vpc1 안에 있는 상태가 되므로, vpc2에 있는 서비스에 접속을 할 수 있어야 한다! 오잉 근데 크롬을 켜고 ip로 접속하면 접속이 되는데… `example.internal` 도메인으로 접속하면 접속이 안된다.

## 해결

1시간 정도 나랑 흰수염고래님이랑 *아니 진짜 이상하네…* 하면서 머리를 싸맸지만 답이 나오질 않았다. 그러자 옆에 있던 카멜레온님이 *음 진짜 이상하네요…* 라며 같이 고민해주셨고, vpc1에 띄워놨던 ec2에 ssh로 직접 접속해서 `example.internal`로 http 요청을 보냈는데 연결이 됐다. 카멜레온님의 추측은 *vpn을 켜도 dns주소가 달라서 사설 도메인에 묶인 ip주소를 못 가져오는 것이 아닐까요?* 였고… 그 말을 듣고 ovpn 설정 파일에 dns 주소를 추가하니 그제서야 도메인 접속이 됐다 (킹멜레온 ㄷㄷㄷ…)

```text
dhcp-option DNS 192.168.0.2
```

(aws vpc내부 dns 서버는 ip대역의 가장 첫번째 주소 + 2를 한 값이다)

vpc1에 띄워놓은 ec2에는 당연히 dns 서버 주소가 저 주소로 연결이 돼있을 거지만, 로컬에서는 vpn을 켜도 dns 서버 주소가 변경되지 않으므로 8.8.8.8에서 `example.internal`이 누구인지 찾고 있던 것이었다.

## 느낀점

이전까지는 사설 도메인을 쓸 일이 없으니까 dns 주소의 중요함을 몰랐었고 (설정 안해줘도 잘 찾아가니까), 설정해본 적도 없었지만 이렇게 사설 도메인 주소를 써야 할 상황이라면 dns 주소가 중요하다는 것을 알게 되었다 ㅎㅎ

## References

1. [what is vpc peering](https://docs.aws.amazon.com/vpc/latest/peering/what-is-vpc-peering.html)
2. [what is service discovery](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/service-discovery.html)
3. [service discovery feature added in 2018](https://aws.amazon.com/ko/blogs/korea/amazon-ecs-service-discovery/)
