---
layout: post
type: tags
title: "[PostgreSQL, Manticore] semantic scholar 검색 인프라 구현"
date: 2025-05-25 14:00:00 +0900
comments: true
toc: true
tags: aws rds postgresql manticore
---

오늘은 오랜만에 기술적인 글을 쓰려고 한다. 문라이트 프로젝트하면서 100M의 저자, 200M의 논문, 2B의 인용 관계를 검색해오는 시스템을 구축했던 과정을 얘기하고자 한다.

## 문제 상황

문라이트는 특정 조건에서 semantic scholar라는 곳에서 논문을 검색해 온다. semantic scholar가 웬만한 논문들도 잘 검색되고, 인용 및 피인용 관계까지 제공해줘서 문라이트 초창기부터 신세지고 있었다.

단일 논문 검색이라면 open alex라는 곳을 이용하면 쉽게 됐는데, 인용 관계를 가져올 수 있는 곳은 semantic scholar가 유일했다. 그리고 인용관계를 가져오는 것을 포기할 수 없는 상황이었다.

semantic scholar는 api key 하나당 1 rps의 제한을 받는다. 얘네한테 api key를 더 달라고 하거나 rps를 늘려달라 했는데 답장이 안 오는 상태고, 유저는 계속해서 늘어나는 상태였다. 이미 rps 측면에서 트래픽은 한계가 다다랐고 DAU 혹은 WAU가 2배, 3배만 돼도 기능이 안정적으로 동작하지 못할 거라고 쉽게 예상할 수 있었다.

그러던 중 semantic scholar가 데이터셋을 제공한다는 사실을 알게 되었다. [데이터 어떻게 다운로드 받는지 문서](https://www.semanticscholar.org/product/api%2Ftutorial#download-full-datasets)도 잘 작성돼 있었다. 레코드 수가 글 제일 위에 말한 것처럼 엄청나게 많지만, rds에 넣는 것 뿐이라면 그렇게 오래 걸릴 것 같지도 않았다.

사실 이걸 팀원분들이랑 상의하면서 오버엔지니어링이 되는 게 아닐까 좀 고민을 했었다. 실제로 팀원분들도 걱정을 하셨다. 그러나 인용관계를 안정적으로 가져오는 유일한 방법은 이것 하나 뿐이었고, 원래 확률적으로 가져오던 인용관계를 안정적으로 가져올 수만 있다면 (외부 api라서 성공과 실패를 50대 50으로 생각하고 fallback 로직도 다 복잡하게 설계해 뒀었다. 그래서 코드도 동작도 복잡했다.) 문라이트에서 더 풍부한 기능을 제공할 수 있을 것이라 생각했다.

고려사항은 아래와 같다.

1. 논문을 제목으로 검색할 수 있어야 한다
   - 요구하는 검색 자체는 복잡하지 않다. 논문 제목(query)은 거의 원래 제목과 유사하게 들어온다. 이와 맞는 논문 하나를 찾을 수만 있으면 된다.
     - 대신 없으면 없다고 확실히 알 수 있어야 한다. 제목이 비슷한 유사한 걸 찾아오면 안된다. (false positive)
2. 논문을 검색했을 때 저자 정보와 인용 정보가 같이 와야 한다
   - 저자 정보는 문라이트 화면에서 논문 정보 보여줄 때 사용하고 있다
   - 인용 정보는 무조건 가져와야 한다. 이걸 해결하려 이 구조를 만든거니까
3. 검색 api가 평균 latency 3초 후반대로 조금 긴 편인데, 이 문제를 해결할 수 있어야 한다
   - 사실 위 수치는 검색 api 전체의 평균 latency이고, 검색에 성공했을 때 latency는 5s 정도로 더 길다.
     - rps 제한으로 오류가 나면 더 빨리 오류를 뱉고, 오류가 나지 않으면 재시도하면서 더 오래 검색한다.
4. 간단한 구조로 빨리 할 수 있어야 한다
   - 팀원분들께 말씀드릴 때 영업일 2~3일 안에 db 구축하겠다고 리소스 얼마 안 들게 하겠다고 설득했다
   - 결국 5~6일 정도 들었다… ㅜㅜ 너무 얕잡아봤다.
5. 돈이 많이 들면 안된다
   - 문라이트는 이제 막 bep를 넘겼다. 문라이트 동료분인 도토리님은 이걸 구축하는 게 돈 쓸 가치가 있는 일이라고 투자해도 된다고 하셨지만 달에 몇백씩 쓰는 건 좀 부담스럽긴 하다.
6. insert 빈도는 한두달에 한번 정도로 매우 낮고, select는 자주 해와야 한다.

## 해결 과정

### 1. 데이터 다운로드

이거부터 정말 쉽지 않았다. 회사에 on premise 서버가 있어서 거기에 모두 다운로드 받을 생각이었다. 용량은 다행히 다 합쳐서 1T가 되지 않아서 충분했지만 다운로드 속도가 문제였다. 속도가 느려서 다운로드는 db 구축 마지막 단계까지 병행했다.

author, paper, citation 별로 100MB 30개, 1GB 30개, 1GB 229개 압축 파일을 다운로드 받아야 했다.

on premise 서버의 network bandwidth는 up/down 100mbps였는데 다운로드 하나 받게 하고 실제로 측정해보니 down이 7~8mbps 정도에 그쳤다. citations 하나 다운받는데 30분 정도가 걸렸다. 여러개 동시에 해도 그닥 높아지지 않았다. 이 구린 속도로 주말 내내 다운받도록 했지만 전체 50%도 다운받지 못했다. (접속 끊겨도 명령어 실행이 되도록 tmux 사용했다)

다행히 [aria2c](https://aria2.github.io/) 명령어를 사용하니 그때부터 3~4배 정도 빨라지기 시작했다.

bandwith 체크는 아래 스크립트 켜놓고 했다.

```bash
#!/bin/bash

# ethtool eth0 으로 확인해서 1000Mbps 인지 확인

MAX_BW=12500000  # 100 Mbps = 12.5 MB/s = 12,500,000 bytes/s

echo "Monitoring eth0... (Ctrl+C to stop)"
echo "Speed based on max 100 Mbps (12.5 MB/s)"
echo

while true; do
  R1=$(awk '/eth0:/ {print $2}' /proc/net/dev)
  T1=$(awk '/eth0:/ {print $10}' /proc/net/dev)
  sleep 1
  R2=$(awk '/eth0:/ {print $2}' /proc/net/dev)
  T2=$(awk '/eth0:/ {print $10}' /proc/net/dev)

  RX_BPS=$((R2 - R1))
  TX_BPS=$((T2 - T1))

  RX_MBPS=$(awk "BEGIN {printf \"%.2f\", $RX_BPS / 125000}")
  TX_MBPS=$(awk "BEGIN {printf \"%.2f\", $TX_BPS / 125000}")

  RX_PCT=$(( RX_BPS * 100 / MAX_BW ))
  TX_PCT=$(( TX_BPS * 100 / MAX_BW ))

  echo "↓ Download: $RX_MBPS Mbps ($RX_PCT%) | ↑ Upload: $TX_MBPS Mbps ($TX_PCT%)"
done
```

### 2. 데이터 분석 & 테이블 스키마 생성

데이터 까보니 jsonl 형태로 돼있었다. python 스크립트 작성해서 필요한 데이터 컬럼만 걸러내고 테이블 스키마를 구성했다. (db는 psql 사용했다.)

author, paper, citation 3개 테이블로 구성했는데, paper가 author id를 갖고 있고, citation이 인용/피인용 paper id를 갖고 있는 형식이다.

이때 좀 불안했던 것이 ‘대부분의 rds 테이블은 몇백만개 까지가 원활하게 동작하고 그 위부터는 모른다’ 라고 알고 있어서 단일 db 인스턴스에 2B개 레코드가 잘 들어갈 지 확신이 안 섰다.

안 들어가면 샤딩이 필요할 것이고, 들어가도 select가 느리면 cache나 read replica가 필요할 것이고, 둘 중 하나라도 하게 된다면 인프라 비용은 n배가 되고 구조는 더욱 복잡해질 것이며 시간도 많이 들 것이다. gpt랑 여기서 플랜B를 세우려 씨름 많이 했었다. (다행히 그럴 일은 없었다.)

필자는 10M이 넘는 레코드를 다뤄본 적이 없었다. 검색 좀 해보니까 데이터 구조에 따라서 심하게 달라진다고 한다. 해보기 전까지는 알 수 없고 우리 테이블 구조는 컬럼 수 적고 join 많이 안 하니까 괜찮을 것 같아서 일단 진행했다.

### 3. SQL Script로 변환 & Local DB 업로드

insert 구문으로 bulk upload하는 sql 스크립트를 작성하면 엄청 느리다. 전에 pg_dump 명령어 사용하면서 안 사실이다. 그거로 테이블 덤프 따니까 아래와 같은 구조로 스크립트가 생성됐고 전에 내가 insert로 짠 스크립트는 아래보다 훨씬 느렸기 때문이다.

```sql
CREATE TABLE IF NOT EXISTS ...;

COPY {table} ({colums...}) FROM STDIN;
<row 1>
<row 2>
...
\.
```

insert 대신 copy를 쓰는 이유가 insert는 줄 하나마다 연산을 독립적으로 끝내지만 copy는 파일 다 받고 나서 테이블에 다 복사하고 한꺼번에 해서 뭐 그래가지고 오버헤드가 적다고 한다. 더 자세히는 안 찾아봤다. 아무튼 위처럼 copy sql 파일로 변환하고, 로컬로 띄워놓은 db에 일부 데이터를 테스트로 업로드했다.

여기가 제일 짜증났다. text나 text[], jsonb 필드 넣을 때 특수문자 escape 에러가 너무 많이 났기 때문이다. psql 에러 메시지도 되게 불친절했고 trial and error로 직접 escpae 로직을 짤 수밖에 없었다.

후에 rds에도 업로드하면서 citation에 있는 context라는 text[] 필드(논문에서 해당 인용 주변 text를 저장해놓는 필드)가 용량을 너무 많이 차지해서, 지금 당장 꼭 필요한 필드만 남겨둬야 했다. 컬럼 하나 있는거 없는 거 차이가 스토리지 용량 몇십GB를 좌우하니 불가피했다.

### 4. AWS RDS로 업로드

이제 authors, papers, citations가 sql로 다 변환이 됐으니 rds로 업로드해야 했다.

제일 수가 적은 authors (100M) 잘 되고, papers (200M) 넣는데 좀 뭔가 이상했다. 일단 cpu가 빌빌댔다. 100%로 잘 작업하고 있다가 어느새 쿼리가 안 끝났는데도 20%대에서 빌빌대고 있었다. 알고보니 cpu credit을 다 소모했던 것이었다.

cpu credit이란 burstable instance에 적용되는 개념인데, cpu 자원을 아껴놨다가 필요한 상황에 threshold 이상으로 가동할 때 쌓아둔 credit 소모하는 방식이다. ([참고](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances.html)) 이름 그대로 가끔씩 폭발적으로 cpu를 써야할 때 유용하다. 근데 난 이걸 몇시간이 걸리는 인덱싱 작업에 쓰려고 하니 cpu credit이 바닥난 것이고, 20%대에서 멈춰있던 것이었다.

아무튼 지금 서비스하고 있는 db가 아니니 편하게 instance type은 burstable이 아닌 다른 타입으로 바꿨다. 그랬더니 papers가 잘 들어갔다. 그런데 조금 들어가고 나니 속도가 너무 느려졌다. 스크립트 10개 정도까지는 2시간 정도 안에 잘 됐는데, 자고 일어나 보니 그동안 4~5개 밖에 안 돼있었다. 너무 느려져서 원인을 파악해보니 primary key로 지정했을 때 자동으로 생기는 index 때문에 insert가 느렸던 것이었다.

bulk upload할 때는 index를 삭제하고, 다 끝난 뒤에 index를 생성하는 것이 바람직하다. index 지우고 다시 넣어보니 1개 스크팁트 당 5분 정도로 되게 빠르게 들어갔다. 그러고 나서 id 컬럼에 인덱스를 생성했는데 30분 정도 걸렸다.

citations도 동일한 방법으로 진행했다. 들어가는 건 빠르게 잘 들어갔고 인용 paper id로 인덱스 생성할 떄 5시간이나 걸렸다. 피인용 paper id도 비슷하게 걸렸던 것 같다. 인덱싱 생성에 걸리는 시간 복잡도가 궁금해서 좀 찾아봤다. ([참고](https://www.geeksforgeeks.org/introduction-of-b-tree-2/))

search, insert 모두 logn이 걸린다. 인덱스 안 지우고 insert했을 때는 bulk insert니까 mlogn 정도가 걸려서 그렇게 오래 걸렸던 것이었다. (m은 넣을 레코드 개수, n은 이미 있는 레코드 개수)

200M 테이블 인덱스 생성에는 30분이 걸렸고, 2B 테이블 인덱스 생성에는 5~6시간 정도로, 레코드 수에 어느정도 비례해서 시간이 소요됐다. 이건 왜 그럴까? [여길](https://www.postgresql.org/docs/current/progress-reporting.html#CREATE-INDEX-PROGRESS-REPORTING) 보면 index creation phase를 볼 수 있다. 아래 query로도 확인 가능하다.

```sql
SELECT * FROM pg_stat_progress_create_index;
```

1부터 n개의 레코드를 넣는다고 하면 log1 + log2 + … + logn 이렇게니까 이것도 nlogn 정도가 든다고 볼 수 있다. 아래 수식 3개로 계산이 얼추 맞는 걸 볼 수 있다.

```
200000000×log(200000000) = 1660205999.13279624
2000000000×log(2000000000) = 18602059991.32796239
18602059991 / 1660205999 = 11.20466978
```

아쉽게도 phase별 소요시간은 체크 못했다. 근데 이번에는 concurrently 없었으니까 building index가 대부분일 것 같다.

_여담이지만 concurrently로 하면 저렇게 인덱싱 한 번 쭉 하고, 다시 sorting해서 테이블이랑 비교하는 작업하고 index 최신화 하나보다. 다시 validation 하는 중에 update되면 안되니까 exclusive lock이 어쨌든 필요한 거 아닌가 라는 생각이 들었고 gpt도 맞다고는 했지만, 더 자세하게 찾아보지는 않았다._

테이블이랑 인덱스 용량도 찍어봤다.

```sql
> SELECT
     t.schemaname,
     t.tablename,
     c.reltuples::bigint                            AS num_rows,
     pg_size_pretty(pg_relation_size(c.oid))        AS table_size,
     psai.indexrelname                              AS index_name,
     pg_size_pretty(pg_relation_size(i.indexrelid)) AS index_size,
     CASE WHEN i.indisunique THEN 'Y' ELSE 'N' END  AS "unique",
     psai.idx_scan                                  AS number_of_scans,
     psai.idx_tup_read                              AS tuples_read,
     psai.idx_tup_fetch                             AS tuples_fetched
 FROM
     pg_tables t
     LEFT JOIN pg_class c ON t.tablename = c.relname
     LEFT JOIN pg_index i ON c.oid = i.indrelid
     LEFT JOIN pg_stat_all_indexes psai ON i.indexrelid = psai.indexrelid
 WHERE
     t.schemaname NOT IN ('pg_catalog', 'information_schema')
 ORDER BY 1, 2;
+------------+----------------------------+------------+------------+-------------------------------------------------------+------------+--------+-----------------+-------------+----------------+
| schemaname | tablename                  | num_rows   | table_size | index_name                                            | index_size | unique | number_of_scans | tuples_read | tuples_fetched |
|------------+----------------------------+------------+------------+-------------------------------------------------------+------------+--------+-----------------+-------------+----------------|
| public     | authors                    | 102273888  | 15 GB      | authors pkey                                          | 4149 MB    | Y      | 40235537        | 12617271    | 12617271       |
| public     | papers                     | 220940992  | 177 GB     | papers pkey                                           | 8628 MB    | Y      | 3398357         | 5048344     | 3612894        |
| public     | citations                  | 2775123200 | 154 GB     | 인용 paper id index                                    | 20 GB      | N      | 49902           | 1793972     | 1793972        |
| public     | citations                  | 2775123200 | 154 GB     | 피인용 paper id index                                   | 21 GB      | N      | 48182           | 2478726     | 2478726        |
+------------+----------------------------+------------+------------+-------------------------------------------------------+------------+--------+-----------------+-------------+----------------+
```

author랑 paper를 비교해 보니 레코드 수 차이만큼 인덱스 크기가 차이나는 것을 볼 수 있다. 아마 index size는 레코드 수에 비례하는 모양이다. (참고로 author id랑 paper id는 모양이 유사하다)

paper랑 citation에서 생성된 인덱스가 컬럼 타입이 똑같고 레코드 수는 10배 정도 차이가 나는데 index 크기는 2.5배 정도밖에 차이나지 않는다. 이건 중복된 값이 많아서 그렇다.

전부 unique라면 아마 10배 정도 차이났겠지만 중복값이 많아서 용량이 많이 작아진 듯 하다. distinct count 확인하려고 explain query 해봤으나 비용이 말이 안돼서 (밑에 나올 쿼리들의 10배가 넘게 나옴 - 며칠 걸릴 것으로 예상된다) … 여기까지만 알아보자.

### 5. Title Search 시스템 구축

이제 마지막 단계다. 논문 제목으로 논문을 검색할 수 있어야 한다.

원래는 postgres full text index를 생성해서 검색하려 했다. 근데 위 과정을 거치면서, 저 간단한 btree index도 용량이 꽤 되는데… 200M 레코드에 full text index는 추가하기가 조금 겁났다. 그래서 논문 제목이랑 id만을 뽑아서 검색 엔진에 넣는 구조로 바꿨다.

es에 넣을 생각이었는데, 너무 무거워 보였다. es 클라우드나 aws opensearch에 올리자니 비용이 말이 안됐고, ec2 위에서 돌리려고 했는데 es가 jvm 위에서 돌아가기 때문에 메모리 비용이 너무 겁이 났다. 32GB나 64GB짜리 사용하면 달에 40~50은 우습게 나올 거다. 뭔가 더 간단한 방법이 있을 것 같았다.

그래서 더 가볍고 싼 대안을 찾으려고 gpt한테 딥 리서치를 시켰다. 그랬더니 manticore라는 es 대체재를 알게 되었다. 나같은 사람들이 많았는지 [이런 비교 글](https://manticoresearch.com/blog/manticore-alternative-to-elasticsearch/)도 걔네들이 만들어 놓았다. 비교 글 보고 아래 이유 때문에 사용했다.

1. c++로 작성해서 하드웨어에 더 가깝고 그래서 프로그램이 가볍다
2. Million 단위에서 es보다 검색이 빠르다
3. 이건 좀 부가적인 건데 sql 친화적이래서 mysql 서버로 접속할 수 있다는 게 쿼리 같은거 테스트하기 편해 보였다.

시간이 촉박한 상황에서, 그리고 해보기 전까지는 모르기 때문에 es와 manticore를 비용 측면에서 충분히 비교는 못했다. 지금 돌이켜보니 es로 했으면 동일한 ec2 스펙으로 돌아갔을까 라는 궁금증이 들기는 하다. 어쨌든 위 이유들만으로 결정하기에는 충분하다 생각하고 manticore를 ec2에 띄웠다. (2cpu랑 8mem으로 띄웠다.)

manticore에서는 [postgresql 연결](https://manual.manticoresearch.com/Data_creation_and_modification/Adding_data_from_external_storages/Fetching_from_databases/Database_connection) 및 [distributed table](https://manual.manticoresearch.com/Creating_a_table/Creating_a_distributed_table/Creating_a_distributed_table)을 지원하기 때문에, 우선 한 테이블에 적당한 레코드 개수가 몇개인지 알아봐야 했다. 그래서 10만개 해봤을 때 잘 되고… 1M(100만개) 해보니 잘 된다. 10M도 잘 된다. 아싸 하면서 100M 하는 순간 ec2가 뻗었다 ㅋㅋ… 지표 보니까 메모리 부족이라서 4cpu 16mem으로 업그레이드하고 다시 돌렸다.

명령어 top으로 보면서 하니 100M 레코드를 db에서 뽑아오고 나서 인덱싱하려 메모리에 올리는 과정에서 8GB 정도 들었다. 근데 100M 단위로 하니 테이블 하나 생성이 너무 오래 걸려서 결국 50M 테이블 5개로 구조를 바꿨다. 이렇게 하니까 테이블 하나 만드는 데 20분 정도 든다.

이제 다 했군 하면서 mysql 서버 접속해서 attention is all you need를 검색하는 순간 안 나왔다. 엥 뭐지 하면서 db에서 검색해보니 잘 나왔다. 데이터 레코드 수는 분명 200M으로 rds와 동일했다. 문제는 rds → manticore로 export하는 쿼리 스크립트에 있었다.

```sql
SELECT id, title FROM papers LIMIT n OFFSET m;
```

이렇게 하면 동일한 데이터가 들어있는 동일한 테이블에 쿼리를 때려도 결과가 다르게 나올 수 있다. 테이블 5개 각각 limit, offset를 다르게 해도 결과가 다르니 테이블끼리 중복되는 레코드 수가 많을 것이고, (mutually exclusive하지 않음) 그래서 인덱싱 과정에서 빠지는 논문들이 많았던 것이었다. 하필 테스트로 검색해본 논문이 없어서 다행이었다. 있었으면 모르고 넘어갈 뻔했다.

사실 인지는 하고 있었는데, 멋대로 postgres가 스토리지에서 메모리 주소 순서대로 긁어오는 거라 생각해서, 테이블 레코드 데이터만 안 바뀌면 저 순서가 안 바뀔 거라 나이브하게 저렇게 쿼리를 작성했다. order by 연산이 너무 커보였던 것도 있었다. 그런데 저 attention is all you need가 없다는 결과가 그게 아니라고 말해주었다.

그래서 쿼리를 order by로 붙여서 바꾸려 했다.

```sql
SELECT id, title FROM papers ORDER BY id LIMIT n OFFSET m;
```

이러면 테이블 데이터가 안 바뀐다면 항상 동일한 결과를 가져올 수 있을 것이다. 근데 아무래도 order by가 너무 오버헤드가 컸다. 3번째, 4번째 테이블로 갈수록 1~2시간이 넘게 걸렸다. 나는 정렬하는 게 목적이 아니라 mutually exclusive하게만 가져오는 것이 목적이었다. 그래서 아래와 같이 바꿨다.

```sql
SELECT id, title FROM papers WHERE id >= '0' AND id < '15'
```

id 필드는 `117830989` 이런식으로 생겼는데, text 필드로 저장돼있다. text 필드라도 대소관계는 비교할 수 있으므로 저렇게 조건문을 5개 테이블이 거의 동일한 수의 레코드를 가져오도록 where절에 있는 2개 값만 잘 찾으면 됐다. 이렇게 하니까 쿼리가 확실히 빨라졌다. explain으로 비교해 보자.

```sql
> explain SELECT id, title FROM papers WHERE id >= '0' AND id < '15'
+--------------------------------------------------------------------------------------+
| QUERY PLAN                                                                           |
|--------------------------------------------------------------------------------------|
| Seq Scan on papers  (cost=0.00..26538427.88 rows=47001327 width=96)                  |
|   Filter: ((id >= '0'::text) AND (id < '15'::text))                                  |
+--------------------------------------------------------------------------------------+

> explain SELECT id, title FROM papers ORDER BY id LIMIT 50000000
+-----------------------------------------------------------------------------------------------------------------+
| QUERY PLAN                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|
| Limit  (cost=50482883.74..56316624.36 rows=50000000 width=96)                                                   |
|   ->  Gather Merge  (cost=50482883.74..71964757.83 rows=184117494 width=96)                                     |
|         Workers Planned: 2                                                                                      |
|         ->  Sort  (cost=50481883.71..50712030.58 rows=92058747 width=96)                                        |
|               Sort Key: id                                                                                      |
|               ->  Parallel Seq Scan on papers  (cost=0.00..24144900.47 rows=92058747 width=96)                  |
+-----------------------------------------------------------------------------------------------------------------+

> explain SELECT id, title FROM papers ORDER BY id LIMIT 50000000 OFFSET 100000000
+-----------------------------------------------------------------------------------------------------------------+
| QUERY PLAN                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|
| Limit  (cost=62150364.99..67984105.61 rows=50000000 width=96)                                                   |
|   ->  Gather Merge  (cost=50482883.74..71964757.83 rows=184117494 width=96)                                     |
|         Workers Planned: 2                                                                                      |
|         ->  Sort  (cost=50481883.71..50712030.58 rows=92058747 width=96)                                        |
|               Sort Key: id                                                                                      |
|               ->  Parallel Seq Scan on papers  (cost=0.00..24144900.47 rows=92058747 width=96)                  |
+-----------------------------------------------------------------------------------------------------------------+

> explain SELECT id, title FROM papers ORDER BY id LIMIT 50000000 OFFSET 150000000
+-----------------------------------------------------------------------------------------------------------------+
| QUERY PLAN                                                                                                      |
|-----------------------------------------------------------------------------------------------------------------|
| Limit  (cost=67984105.61..71964757.83 rows=34117494 width=96)                                                   |
|   ->  Gather Merge  (cost=50482883.74..71964757.83 rows=184117494 width=96)                                     |
|         Workers Planned: 2                                                                                      |
|         ->  Sort  (cost=50481883.71..50712030.58 rows=92058747 width=96)                                        |
|               Sort Key: id                                                                                      |
|               ->  Parallel Seq Scan on papers  (cost=0.00..24144900.47 rows=92058747 width=96)                  |
+-----------------------------------------------------------------------------------------------------------------+
```

- seq scan은 테이블 전체 scan
- sort는 정렬
- cost=A…B는 A가 첫번째 결과 나올 때까지 걸리는 최소 비용 (startup cost), B는 쿼리를 끝까지 실행하는 데 드는 비용 (total cost)

대충 이정도로 이해했다. 맨 윗줄 cost보면 나머지랑 확실히 다르다. 똑같이 seq scan해도 1번 명령어는 sort하는 연산이 없어서 cost가 2번 명령어와 2배 차이난다. 심지어 offset이 늘어날 수록 첫 쿼리 나올 때까지 기다려야 하는 cost가 그만큼 늘어나기 때문에 cost도 무시할 수 없을 만큼 늘어난다.

이렇게 쿼리 바꿔서 테이블을 생성하니 인덱스 하나당 20~30분 정도 소요되고 attention is all you need 논문도 검색이 잘 됐다.

## 결과

이렇게 인프라 구축하고 검색 쿼리도 살짝 조정하고 이것저것 만지니까 소요기간 말고는 꽤 만족할 만한 결과가 나왔다. 처음부터 알았더라면 삐걱대지 않고 2~3일만에 타이트하게 할 수 있었을 것 같다. 계획을 잘못 잡았고 절대 못했을 거라는 얘기다 ㅎㅎ… 그렇지만 이번에 작업하면서 db에 대해 많이 배웠다. 재밌었다.

- search api latency: 3.7s → 1.0s
- search api hit rate: 62% → 72%
- 인프라 비용: 매달 70~80만원
- 작업 소요기간: 영업일 5~6일

## References

- [semantic scholar dataset](https://www.semanticscholar.org/product/api%2Ftutorial#download-full-datasets)
- [aria2c](https://aria2.github.io/)
- [aws burstable instance](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/burstable-performance-instances.html)
- [btree time complexity](https://www.geeksforgeeks.org/introduction-of-b-tree-2/)
- [postgres creating index phase](https://www.postgresql.org/docs/current/progress-reporting.html#CREATE-INDEX-PROGRESS-REPORTING)
- [manticore vs elasticsearch](https://manticoresearch.com/blog/manticore-alternative-to-elasticsearch/)
