---
title: kafka-consumer
date: 2022-09-13 00:00:00 +0900
categories: [Stream Data Processing, kafka]
tags: [kafka, kafka-consumer, stream Data processing]
pin: true
---
# Kafka consumer
<h3 data-toc-skip>kafka consumer 란</h3>

데이터 read(poll) 주체<br>
commit을 통해 consumer offset을 카프카에 기록

<br>
<br>
<br>
![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/1.png?raw=true){: width="700" height="400" }


consumer가 자동이나 수동으로 읽은 데이터의 위치를 commit하여 다시 읽음을 방지한다<br>
__consumer_offsets라는 Internal Topic에서 consumer offset을 저장하여 관리한다

<br>
<br>
<h3 data-toc-skip>consumer 작동 방식</h3>
<h4 data-toc-skip>1. single consumer</h4>

![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/2.png?raw=true){: width="700" height="400" }


Topic의 모든 partition 에서 모든 Record를 consume한다.

<h4 data-toc-skip>2. multiple consumer</h4>

> 동일한 group.id로 구성된 모든 consumer들은 하나의 consumer group을 형성한다.
{: .prompt-info }

![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/3.png?raw=true){: width="700" height="400" }

partition 은 항상 consumer group에서 하나의 consumer에 의해서만 사용이 된다.<br>
consumer group의 consumer들은 작업량을 어느정도 균등하게 분할한다. <br>


![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/4.png?raw=true){: width="700" height="400" }

다른 consumer group의 consumer들은 분리되어 독립적으로 작동이 된다. 


<h4 data-toc-skip>3. consumer group 과 rebalancing</h4>

![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/5.png?raw=true){: width="700" height="400" }

> consumer group의 consumer는 자신들이 읽는 토픽 파티션의 소유권을 공유한다. 
{: .prompt-tip }

<br>
새로운 consumer를 그룹에 추가할때, 특정 consumer에 문제가 생겨 중단될때 <br>
consumer가 오랫동안 하트비트를 보내지 않으면 세션 타임아웃 일어난다.<br>
Rebalancing 하는 동안에는 consumer들은 메세지를 읽을 수 없으므로 해당 그룹 전체가 잠시 사용 불가능하다.<br>

> Rebalancing : 한 consumer로 부터 다른 consumer로 파티션 소유권을 이전하는 것
{: .prompt-tip }

<br>
<br>
<h3 data-toc-skip>commit 과 offset</h3>

commit 
: 파티션 내부의 현재 위치를 변경하는 것
    
offset 
: 컨슈머 자신이 읽는 레코드의 현재 위치 

<br>

> 리밸런싱의 문제가 발생하면, 각 consumer는 이전과 다른 파티션을 할당받게 될 수 있다. 이에 따라 메세지를 중복처리하거나 유실되는 경우가 있다. 특히,  consumer 를 구성할 때, enable.auto.commit=true 로 두면 아래와 같은 경우 발생가능성이 있음.
{: .prompt-warning }


<h4 data-toc-skip>1. 중복처리 경우</h4>

![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/6.png?raw=true){: width="700" height="400" }

<h4 data-toc-skip>2. 유실되는 경우</h4>

![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/7.png?raw=true){: width="700" height="400" }

<br>
<br>
<br>

<h4 data-toc-skip>consumer 구성에서 중요한 configuration</h4>
<br>
<br>
auto.offset.reset 
: 커밋된 오프셋이 없는 파티션을 컨슈머가 읽기 시작할때, 또는 커밋된 오프셋이 있지만 유효하지 않을때, 컨슈머가 어떤 레코드를 읽을지 제어하는 매개변수
    
latest(default) 
: (컨슈머가 실행 된 후 새로 추가된 레코드들) 을 읽음
    
earliest 
: 해당 파티션의 맨 앞부터 모든 데이터를 읽음
    
enable.auto.commit 
: 컨슈머의 오프셋 커밋을 자동으로 할 것인지에 대한 제어 <br>
 true(default) ; [auto.commit.interver.ms](http://auto.commit.interver.ms/) 로 자동으로 오프셋 커밋하는 시간 간격을 제어 할 수 있다. 속도가 가장 빠르고,  commit 관련 코드를 작성할 필요가 없는 장점이 있다.<br>
 false ; commitSync,  commitAsync 사용 하여 offset commit을 제어함
    
<br>
<br>
<br>

자동 커밋 상황
![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/8.png?raw=true){: width="700" height="400" }


자동 커밋 중 리밸런스가 일어났을 때 
![Desktop View](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-13-kafka-consumer/9.png?raw=true){: width="700" height="400" }



<h4 data-toc-skip>commitSync : 현재 오프셋 커밋</h4>
1. consumerRecord 처리순서를 보장한다. 
2. 가장느림 ( 커밋이 완료될 때 까지 block )
3. poll() method로 반환된 consumerRecord의 마지막 offset을 커밋한다. 
    

<h4 data-toc-skip>commitAsync</h4>
1.commitSync 보다는 빠르다 <br> 브로커의 commit 응답을 기다리는 대신, commit 요청을 전송하고 처리를 계속 할 수 있음 
2.중복이 발생할 수 있다 <br> 일시적인 통신문제로 이전 offset 보다 이후 offset 이 먼저 commit 이 될때
3.consumerRecord 처리 순서를 보장하지 못한다



<h3 data-toc-skip>offset을 다루는 방법</h3>

<h4 data-toc-skip>1. Consumer Group의  offset 상태 확인</h4>

consumer group을 지정하고 `--describe`옵션을 사용하면 현재 consumer group의 offset 정보를 볼 수 있다.<br>명령어는 다음과 같다.

```bash
kafka-consumer-groups \
--bootstrap-server <host:port> \
--group <group.id> \
--describe
```

실행결과 예시

```bash
TOPIC         PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                      HOST            CLIENT-ID
example.topic 0          6392623366      6392623859      493             consumer-1-f6f6ffb0-1054-46b9-af13-0b254bc14da0  /10.64.69.95    consumer-1
example.topic 1          6394637143      6394637383      240             consumer-10-6c57b320-7742-4418-8e15-b7d735da346e /10.64.69.95    consumer-2
example.topic 2          6397170269      6397170495      226             consumer-19-dbed41a1-42bb-4ecb-bc8f-84e47c74dbe8 /10.64.69.95    consumer-3
example.topic 3          6397170269      6397170495      226             consumer-19-dbed41a1-42bb-4ecb-bc8f-84e47c74dbe8 /10.64.69.95    consumer-4
```

TOPIC
: 토픽 이름

PARTITION
: consumer group 내의 각 consumer가 할당된 파티션 번호

CURRENT-OFFSET
: 현재 consumer group의 consumer가 각 파티션에서 마지막으로 offset을 commit한 값

LOG-END-OFFSET
: producer쪽에서 마지막으로 생성한 레코드의 offset

LAG
: LOG-END-OFFSET에서 CURRENT-OFFSET를 뺀 값.

`--describe`를 통해 조회를 했을때 LAG이 계속 일정 수준을 유지한다면 consumer가 producer 가 만들어내는 이벤트 레코드의 양을 잘 따라가고있다는 것을 확인할 수 있다.<br> 
하지만 LAG이 계속 증가한다면 consumer의 처리 속도가 느린 것이기 때문에 consumer의 갯수를 충분히 증가시키거나, <br>
consumer의 로직을 더 간략화 해서 빠른 속도로 데이터 처리를 할 수 있도록 변경해야 한다.<br>


<h4 data-toc-skip>2. Consumer Group의  offset reset</h4>

kafka에서 데이터를 불러와서 처리하는 과정에서 오류가 발생하거나 문제가 발견된 경우,<br>
다시 원하는 offset부터 데이터를 재처리를 해야할 경우가 종종 있다.<br> 
이때 consumer group의 offset reset 기능을 활용하면 된다.<br>

** consumer group이 실행중인 상태에 offset reset을 진행하는 경우 reset은 실패한다.<br>

```bash
kafka-consumer-groups \
--bootstrap-server <host:port> \
--group <group> \
--topic <topic> \
--reset-offsets \
--to-earliest \
--execute
```

`-topic` 대신 `-all-topics`를 지정하면 모든 토픽에 대해서 실행이 가능하다.<br>
`-execute` 옵션을 제거하고 실행하면 실제 반영되지 않고 어떻게 변할지 결과만 출력하는 dry run이 가능하다.<br>


<h4 data-toc-skip>offset 의 위치를 재설정 하기 위한 옵션 </h4>

`-shift-by <Long: number-of-offsets>` 형식 (+/- 모두 가능)<br>
`-to-offset <Long: offset>`<br>
`-to-current`<br>
`-by-duration <String: duration>` : 형식 ‘PnDTnHnMnS’<br>
`-to-datetime <String: datetime>` : 형식 ‘YYYY-MM-DDTHH:mm:SS.sss’<br>
`-to-latest`<br>
`-to-earliest`<br>



** `--to-datetime`의 경우 kafka에서 데이터를 읽어서 다른곳에 저장하는 중에 데이터 유실 또는 중복 write 등이 발생한 경우에 날짜 단위로 데이터를 다시 불러와서 재처리하고 싶은 경우 매우 유용하다.




예시 > 특정 topic의 파티션 1번만 offset을 30으로 지정하고 싶을때 

```bash
./kafka-consumer-groups \
--bootstrap-server {bootstrap 정보} \
--group click --topic {topic명}:1 \
--reset-offsets --to-offset 30 --execute
```