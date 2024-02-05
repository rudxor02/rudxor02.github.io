---
layout: post
type: tags
title: "[ECS + Celery] Cancel consuming before deploying"
date: 2023-12-19 21:00:00 +0900
comments: true
toc: true
tags: aws ecs celery deployment
---

[celery](https://docs.celeryq.dev/en/stable/)는 한마디로 queue를 중간에 두고 백그라운드 작업을 실행할 수 있게 하는 파이썬 라이브러리다. python, 백그라운드 작업 하면 딱 떠오르는 유명한 라이브러리라서 이쪽에 관심이 있던, 혹은 프로젝트를 진행하셨던 분들이라면 아마 다 아실 거라 생각한다. 회사 프로젝트에서 celery를 사용할 일이 있어서 celery worker를 ecs로 배포했는데, 이와 관련하여 고민했던 과정들을 적어보고자 한다.

# 문제 상황

worker가 실행하는 job 중에는 오래 걸리는 작업들이 있기 마련이다. 애초에 서버가 http 연결을 작업 요청을 받고 작업을 끝낸 다음 결과를 보내기까지 오래 열어두는 것이 부담돼서 celery 같은 걸 사용하는 거니까. 적게는 몇십 초 안에 끝날 것이고 많게는 5분, 10분 걸리는 작업도 있을 것이다. 문제는 ecs에서 배포할 때 만약에 worker에서 작업을 실행하고 있다면 **최대 2분까지 작업을 끝내기를 기다리는 게 가능하다**는 것이다. 이게 무슨 말이냐면

## ECS stop timeout

ecs task definition 파라미터에는 [stop timeout](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#ContainerDefinition-stopTimeout)이라는 게 있다. stop timeout이란, ecs에서는 task를 종료할 때 graceful shutdown을 하는데, task가 알아서 종료되는 걸 최대로 기다리는 시간이다. 좀 더 자세하게 설명하자면 task 종료를 하는 순간부터 container 안에 켜져 있는 프로세스에 `SIGTERM` 시그널을 전달하는데, 하던 것 까지만 마무리하고 더 이상 일을 하지 말고 프로세스를 종료하라는 뜻이다. stop timeout이 지났는데도 프로세스가 종료되지 않으면 `SIGKILL` 시그널을 전달하고, 프로세스는 작업의 진행 여부와 상관 없이 강제종료된다.

> If the parameter isn't specified, then the default value of 30 seconds is used. The maximum value is 120 seconds.

문제는 이게 최대 2분까지밖에 허용이 안 된다는 것이다. (max 120 - 당연히 나같은 사람이 많은 것 같고, ecs 레포 가보면 [관련 이슈](https://github.com/aws/containers-roadmap/issues/256#issuecomment-1549434318)들이 잔뜩 있다.) ecs 재배포도 결국 새로운 버전의 task를 켜고, 현재 버전의 task를 끄는 작업이므로 만약 celery worker에서 2분이 넘게 걸릴 만한 작업을 진행하고 있다가 바로 배포를 하게 되면 작업이 중간에 강제종료된다.

하필 우리 프로젝트에서 진행하는 작업들 중에 2분 넘게 걸리는 게 조금 있어서… ecs를 eks로 옮겨야 하나…? 라는 고민을 했었다. eks에는 아무래도 컨테이너를 직접 관리하기 때문에 이런 옵션이 있을 것 같았기 때문이다. (k8s를 안 써봤지만 [terminationGracePeriodSeconds](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#pod-termination)가 해당 부분인 듯) migration 사례도 많고 ecs와 eks는 애초에 비슷한 게 많기 때문에 크~게 어려울 것 같지는 않았지만, 1~2주 정도는 잡고 가야할 것 같은데 이거 하나만을 위해서 migration을 하기에는 살짝 부담이 됐었다.

# 해결

다행히 celery에서 [cancel-consumer](https://docs.celeryq.dev/en/stable/userguide/workers.html#queues-canceling-consumers)란 커맨드를 간신히 찾아냈다. queue name을 명시해서 명령어를 실행하면 해당 queue를 바라보고 있는 worker가 queue에서 작업을 가져오는 걸 중지한다. 그래서 일정 시간이 지나면 프로세스는 켜져 있고 작업을 실행하지 않는 유령 worker상태가 된다. 요거 cli로 하면 잘 안되길래 (깃헙에도 이슈가 있는데 더 들여다보기 싫어서 그냥 celery 라이브러리 사용해서 실행 했더니 다행히 잘 됐다) 커맨드로 만들어서 cd pipeline에 끼워넣었다.

# 결론

worker에서 작업을 더이상 받지 않게 하는 기능은 뭔가 있을 만한? 기능이라고 생각했지만, docs도 좀 삐리하게 생기고 라이브러리의 기능을 못 믿어서 에이 이런거 없겠지~ 하면서 하마터면 eks로 옮길 뻔했는데 다행히 검색이 돼서 나의 눈에 보였다.

- 수동
  - worker의 상태를 주기적으로 polling해서 task를 실행 안할 때 worker를 끄는 스크립트 실행
    - scalable하지 않은 방법. 이게 가능하려면 한동안 아무 작업이 들어오지 않아야 한다. 그러나 우리는 1분마다 트래픽이 들어오는 게 아니었고 분명히 작업이 아무것도 없이 비는 시간이 있었기 때문에 가능하긴? 했다.
  - 작업을 받는 쪽, 즉 worker에게 작업을 전달하는 쪽을 먼저 queue로 쌓지 않게 함
    - 이건 queue에 있던 모든 task들이 끝날 때까지 기다려야 한다. queue에 작업이 100개 쌓여있으면 worker가 100개 다 처리할 때까지 기다려야 하는 방법. 그러나 위와 같은 이유로 이것도 그 당시만 생각하면 가능한 해결책이었음.
- 코드
  - cancel-consumer를 cd pipeline 중간에 실행
- 인프라
  - eks로 옮겨서 stop timeout 좀 더 큰 값으로 설정

똑같은 문제를 위 3가지 관점에서 다르게 해결할 수 있었던 문제였는데 2번째 방법이 우리 상황에 잘 맞았었다. (1번째는 좀…) 각 계층에서 가능한 옵션이 뭐가 있다는 걸 아는 건 의사결정에 강력한 도움이 되는 것 같다. 라이브러리를 못 믿고 포기하지 말고 docs 진~짜 꼼꼼히 읽어봐야게씀.

# References

1. [celery docs](https://docs.celeryq.dev/en/stable/)
2. [ecs stop timeout](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task_definition_parameters.html#ContainerDefinition-stopTimeout)
3. [github issue - extend stop timeout max value](https://github.com/aws/containers-roadmap/issues/256#issuecomment-1549434318)
4. [sigterm vs sigkill](https://komodor.com/learn/what-is-sigkill-signal-9-fast-termination-of-linux-containers/)
5. [celery cancel-consumer](https://docs.celeryq.dev/en/stable/userguide/workers.html#queues-canceling-consumers)
