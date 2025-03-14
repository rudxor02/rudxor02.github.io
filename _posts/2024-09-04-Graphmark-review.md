---
layout: post
type: tags
title: "[Retro] 두 달만에 혼자 서비스 출시하고 접은 후기"
date: 2024-09-04 15:00:00 +0900
comments: true
toc: true
tags: retro
---

사실 Graphmark 프로젝트 끝난지는 2달이 지났지만, 재충전 핑계로 이제야 회고글을 작성한다.

## 시작하기 앞서…

Graphmark가 먼저 뭐하는 서비스였는지부터 보여주면 좋을 것 같아서 소개 영상 첨부한다.

<video width="320" height="240" controls>
  <source src="/assets/videos/post/2024-09-04-Graphmark-review-graphmark-video.mp4" type="video/mp4">
  Your browser does not support the video tag.
</video>

## Graphmark 시작 배경

대략 4월 중순부터 회사에서 간단히 말해서 llm을 사용하는 b2c 제품을 혼자 만들어 보는 기회를 가졌다. 목표는 llm이기 때문에 가능한 서비스로 수익을 내는 것이다. 말은 정말 간단한데, 그냥 막막했다… 시장조사한답시고 llm 제품들 들여다 보면 죄다 바이럴성같고, 왜 저건 성공하고 이건 실패했는지도 모르겠고, 뭐랄까 그냥 온실 밖으로 나와버린 화초가 된 기분이었다. 나름의 문제를 뾰족하게 잘 정의하고 했겠지만, 그때는 그런걸 보는 시각이 좀 부족했고, 지금도 한참 부족한 것 같다.

어쨌든 이야기 안의 등장인물들의 페르소나를 자동으로 만들어준다거나, 챗봇이 중심이 된 쇼핑몰 등 여러가지 서비스를 생각해 보았지만, 결국 나와 가까운 문제를 푸는 게 맞겠다 싶어서, 그리고 확실히 보이는 문제 같아서 웹서핑 중에 마주치는 수많은 정보들을 어떻게어떻게 편하게 정리해주는 서비스를 기획하기로 했다.

처음에는 (ai업계 팔로업한답시고 x를 깔아서 팔로우 많이 해놓고 있었음) sns상에서 매일 다 못 읽는 피드들을 요약해서 키워드 중심으로 시각화해주는 서비스를 기획했었는데, 저작권 문제가 좀 두려웠다. 그래서 사용자가 직접 인풋을 넣게 하면 책임이 좀 덜하지 않을까 싶어서 북마크한 웹페이지들을 knowledge graph로 시각화해주는 Graphmark를 생각했다.

## MVP까지…

5월 9일에 시작해서 5월 20일에 mvp 출시했었다.

### 문제 정의

```text
사람들은 북마크에 뭘 저장했는지 까먹습니다. 까먹기 때문에 다시 찾으려는 시도조차 못합니다.
```

### 솔루션

```text
기존의 북마크를 좀 더 visualizing해서 대체합니다.
웹서핑을 하다가 관심있게 보는 페이지에 마킹을 하면, 자동으로 엔티티를 뽑아낸다면 페이지를 하나하나 넣으면서 점점 더 발전하는 그래프를 볼 수 있을 것입니다.
새 탭을 켰을 때 이러한 그래프가 보인다면, 나만의 지식 포털 사이트를 만들 수 있을 것입니다.
chrome extension 형태로 제공합니다.
```

### 랜딩 페이지 제작

![image.png](/assets/images/post/2024-09-04-Graphmark-review-graphmark-planning.png)

framer로 제작했다. 로고랑 이것저것 끄적였던 흔적들…

![image.png](/assets/images/post/2024-09-04-Graphmark-review-graphmark-product-hunt.png)

랜딩 페이지 빠르게 만들고 나서 프로덕트헌트에 내놓았었다. 반응은 미적지근했다. waitlist 20명인가 30명 받고, upvote 8명인가? 받았었는데, 그때 내 생각은 몇표가 나오든 무조건 go였다. 지금 돌아보면 이정도 지표가 나온다면 안 하는게 맞는 것 같다.

### 고객 인터뷰

내가 프로젝트 진행했던 기간이 회사 전체가 진짜 바빴던 기간이었는데 감사하게도 회사 동료분들이 시간을 내주셔서 출시 전후 인터뷰를 진행했었다. 지금 돌이켜보면 출시 전은 어쩔 수 없다고 쳐도 출시 후에는 회사 내부 사람들 뿐만 아니라 외부 사람들도 더 자주 인터뷰했었으면 하는 아쉬움이 남아있다. 출시 전에는 총 12번 인터뷰했었고, 출시 후에는 6번 인터뷰했다. 너무 적었다.

### 베타 테스트

개발은 진짜 밤낮, 주말없이 급하게 했었다. 뭔가를 새롭게 안 개발 지식을 정리할 시간도 없고, 기술부채 신경쓸 시간도 없고… 솔직히 이때는 말벌 잡듯이 후다닥해서 잘 기억이 안난다 ㅋㅋㅋ…

회사 내부에서 알파 테스트 진행하고, 피드백 반영하고 바로 waitlist 등록해주신 분들께 베타 테스트 링크를 보냈다. 근데 예상보다 설치를 너무 안 해서 속상했다. waitlist에 등록한 사람들이면 당연히 대부분 메일 열어보고 설치할 줄 알았는데, 5명 정도 설치했다. 메일 열었는지 확인하는 서비스도 이걸 보내고 나서 안 게 너무 아쉬웠다. 그나마 위안이 되는 건 옆에서 나랑 비슷하게 혼자 서비스 개발하시는 동료분들 waitlist 설치율이 잘 기억이 안나지만 얼추 비슷해서 원래 이런거구나 싶었던 거였다.

## MVP 출시 후

![Untitled](/assets/images/post/2024-09-04-Graphmark-review-graphmark-geeknews.png)

waitlist의 슬픔을 뒤로한 채… 출시는 했는데 유입채널이 없어서 긱뉴스랑 해커뉴스 등 여기저기 홍보했다. 그중에 긱뉴스가 가장 반응이 좋았다! 재밌는 아이디어라고 칭찬해주시고 하루이틀 사이에 몇백명씩 들어와 주셔서 정말 기분이 좋았다. ga(google analytics)를 보니 드디어 뭔가 서비스 답게 지표가 찍히기 시작했다.

지금 생각해 보면 나는 진짜 문제를 잘 해결해서 주목받은 게 아니라, 개발자(+ 옵시디언 유저)들이 흥미로워할 만한 프로덕트를 개발해서 주목받은 게 아닐까 싶다. ‘문제를 해결하는 수단’이 주목받은 게 아니라 ‘수단’이 주목받은 느낌.

### 치명적인 실수

그러다 그 주 금요일 퇴근 후, 회사에서 어떤 이유로 로드밸런서를 바꿔야했었는데, 이게 회사 내부에서만 사용하는 (외부 트래픽을 차단하도록 security group이 설정된) 로드밸런서인줄 모르고 바꿨다가 그다음주 월요일까지 대략 2~3일 정도 서비스가 마비됐었다. 나는 vpn 켜두고 일하니까 서비스에 장애가 있는 줄 진짜 꿈에도 몰랐었다.

주말에 ga를 보니 트래픽이 말 그대로 0이라서, 진짜 암울했었다. 나는 로드 밸런서 문제인줄 몰랐으니까, 그때 몇백명이 서비스를 한번씩 쓰고 거짓말처럼 다 나간 줄 알고 있었다. chrome web store상에서는 분명히 설치하고 extension을 켜놓은 사람 수가 몇십명은 되고, Graphmark는 새 탭을 켜면 그래프가 보이는 형식이라 그 몇십명이 컴퓨터를 다 꺼두지 않는 이상 구조적으로 트래픽이 없을 수가 없는데, 둘 중 하나는 업데이트가 느린건가 하면서 보이지 않는 이슈와 주말 내내 쉐도우 복싱하고 있었다. 그 사이에 몇십명이 될 수 있었던 예비 daily user들은 다 나가고 남은 daily user들은 10~20명 정도였다.

하 진짜 이거때문에 멘탈이 너무 안 좋았었다. 무슨 백엔드 개발자가 이런걸 실수하냐 하면서 자책도 많이 했다. 후에 서비스 지표가 안 좋아도 이거때문에 그런건지 내가 서비스에 쓸모없는 기능을 넣은건지 구별이 안됐다. 모수가 너무 적게 남아서 가설을 세우고 실험을 해도 판단이 잘 안 섰다. 그래도 어떡하겠어… 다시 모아야지 하면서 울며 겨자먹기 식으로 계속 홍보하고 다녔다.

그 뒤로 여러 번의 개선을 했다. 감사하게도 회사 내부에 잘 쓰고 계시는 분이 계셔서 그분이랑 많이 대화하면서 기능도 추가하고 많이 고쳤다. 그 뒤로 얼마 지나지 않아 목표했던 지표를 달성하지 못하게 되면서 서비스를 그만두었다.

## 나는 왜 그만두었을까?

사실 solopreneur들 글을 읽다 보면 길게 고민하지 말고 일단 시작해라. 그중에 안되는 거 버리고 잘되는 거 밀고가라. 라는 식의 교훈이 웬만하면 있다. 그래 그건 알겠는데… 프로젝트할 때 제일 어렵고 힘빠지게 만들었던 건 어떨때 포기하고 다른 길을 찾아봐야 하는지 진짜 모르겠다는 점이었다. 그래서 매 단계마다 지표가 안좋았는데 밀고 나갔던 결과로 이어진 것 같다. 터무니 없더라도 인터넷에서라도 기준을 잘 찾아보고 잘 세웠어야 했다. 기준을 명확히 세우지 않았고, 세우더라도 근거가 빈약했기 때문에 기준을 달성못했을 때 지금까지 해온 게 아까워서 칼같은 결정을 내리지 못했다.

피보팅을 한다면 어떻게든 온몸을 비틀어서 할 수 있었겠지만, 멘탈도 안 좋았고, 자신감도 잃어서 회사 업무 시간에 자신없는 방향으로 바꿔서 일해보겠다고 말할 마음도 없었다. 그리고 그때 나에게 지표가 말해주고 있었다.

## 여기서 배운 게 있다면

지금 돌이켜보면 llm만이 가능한 서비스가 목표라서 내가 너무 새로운 걸 하려고 했던 것 같다. 기존 제품에다가 llm을 덧붙이는 식으로 했다면 사용자도 ai 기능에 잘 적응하면서 제품을 사용할 수 있었을 것 같은데, 익숙하지 않은 그래프를 들이밀어서 매력이 떨어진 건 아닐까? 라는 생각도 든다. 시장&수요 조사도 대충대충 했으며, 갑자기 홍대병이 도져서 내가 좋아하는 폰트로 도배를 했었다. 가설 검증 방식도 빈약했다. 그리고 가장 중요한 건 나에게 가까운 문제를 골랐지만, 내가 고객은 아니었다.

전문적인 수준은 아니지만 혼자 기획, 디자인, 프론트, 백, 홍보, 인터뷰, 리서치 다 한 것도 처음이라 여러가지 찾아서 적용하는 게 힘들었지만 그만큼 재밌었고 값진 경험을 한 것 같다. 항상 그때의 내가 최선이라고 생각하는 방향으로 서비스를 개발해왔고 그 과정이 재밌었기 때문에 후회는 없지만 다시 한다면 다르게 할 것 같은 포인트가 있고 서비스 운영 측면에서 실수한 부분도 있어서 아쉬움은 많이 남는다.

시장은 차갑다. 남의 돈 빌어먹고 사는 게 쉬운 게 아니다. 이 2가지 말을 뼈저리게 느낀 두 달이었다. 앞으로 내가 서비스를 또 만들게 된다면 go or die를 판단할 때 이 프로젝트의 각 단계별 지표를 기준으로 삼아야 할 것 같다.
