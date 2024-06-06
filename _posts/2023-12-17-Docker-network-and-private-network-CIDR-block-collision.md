---
layout: post
type: tags
title: "[Docker] Docker network and private network CIDR block collision"
date: 2023-12-17 21:00:00 +0900
comments: true
toc: true
tags: aws vpc vpc-peering docker
---

오늘은 1~2주 전에 docker + vpn을 사용하면서 겪었던 네트워크 이슈에 대해서 잊기 전에 써보려고 한다.

## 문제 상황

1. aws에서, vpc1과 vpc2 사이에 vpc peering을 해놓은 상태이다.
2. vpc1에 ec2로 서버 프로그램을 하나 띄웠고,
3. vpc2에서 ecs로 클라이언트 컨테이너를 하나 띄웠다.
4. 컨테이너에서는 ec2의 private ip 주소로 접근한다.
5. 자꾸 startup health check과정에서 timeout error가 났다. 그런데 public ip 주소로 하면 접속이 잘 된다.
    1. 참고로 vpc2에 있던 다른 컨테이너들은 vpc1에 있는 다른 ec2에 접속이 가능한 상태였다
6. 그래서 결국 환경변수를 public ip로 해놓고 몇 주 동안 켜놨다 ㅠㅠ

오랜만에 여유가 좀 돼서 근무시간에 위 버그를 해결하려 했다.

아무리 생각해도 ec2의 security group도 vpc2 대역을 허가하고 있고, 5.a에서 보다시피 잘 되는데 방금 띄운 ec2만 안된다는 게 너무너무 이상했다. vpc peering도, sg도 정상 작동한다는 뜻이었고, 이 밖에 timeout 에러가 날 이유가 전혀 떠오르지 않았기 때문이다. (timeout error는 주로 ip 주소로 호스트를 못 찾을 때 혹은 포트가 방화벽에 막혀 있을 때 발생하기 때문에…)

## 해결

진~짜 진짜 이상해서 몇 시간 동안 오만 데 다 뒤져봤다. 결국 route table을 봐야겠다 싶어서 해당 ec2에 접속해서 아래 명령어를 쳐봤더니…

```bash
$ ip route
default via 172.31.0.1 dev enX0 proto dhcp src 172.31.3.106 metric 512 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
172.20.0.0/24 dev br-*** proto kernel scope link src 172.20.0.1
...
```

docker private ip 주소와 상대방 vpc의 대역이 겹치는 것이었다… (docker0, br-*** 둘이 겹침) 그래서 docker-compose.yml에 아래와 같이 docker private network의 ip대역을 명시해줬더니 잘 됐다.

```latex
networks:
  my-network:
    driver: bridge
    ipam:
      driver: default
      config:
        - subnet: "172.19.0.0/24"
          gateway: "172.19.0.1"
```

```bash
$ ip route
default via 172.31.0.1 dev enX0 proto dhcp src 172.31.3.106 metric 512 
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 
172.19.0.0/24 dev br-*** proto kernel scope link src 172.19.0.1 
...
```

## 결론

private ip 대역을 사용할 땐 항상 다른 부분이랑 겹치는 게 없는 지 조심하자…
