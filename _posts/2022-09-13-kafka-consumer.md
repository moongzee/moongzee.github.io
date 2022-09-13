---
title: "kafka-consumer"
date: 2022-09-13 00:00:00 +0900
header:
    overlay_color: "#000"
    overlay_filter: "0.5"
  
excerpt: "Kafka"

categories: 
- Kafka
- consumer
tag: 
- kafka
- stream_data

---
# Kafka consumer

- kafka consumer 란
    
    데이터 read(poll) 주체 
    
    commit을 통해 consumer offset을 카프카에 기록
    

![/assets/images/posts/2022-09-13-kafka-consumer/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.08.33.png)

— consumer가 자동이나 수동으로 읽은 데이터의 위치를 commit하여 다시 읽음을 방지함

— __consumer_offsets라는 Internal Topic에서 consumer offset을 저장하여 관리함

- consumer 작동 방식

1. single consumer

![스크린샷 2022-04-13 오후 10.11.01.png](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.11.01.png)

— Topic의 모든 partition 에서 모든 Record를 consume한다.

1. multiple consumer

** 동일한 group.id로 구성된 모든 consumer들은 하나의 consumer group을 형성한다.

![스크린샷 2022-04-13 오후 10.14.08.png](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.14.08.png)

— partition 은 항상 consumer group에서 하나의 consumer에 의해서만 사용이 된다.

— consumer group의 consumer들은 작업량을 어느정도 균등하게 분할한다. 

![스크린샷 2022-04-13 오후 10.11.56.png](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.11.56.png)

— 다른 consumer group의 consumer들은 분리되어 독립적으로 작동이 된다. 

- consumer group 과 rebalancing

![스크린샷 2022-04-13 오후 10.30.36.png](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.30.36.png)

— consumer group의 consumer는 자신들이 읽는 토픽 파티션의 소유권을 공유한다. 

** rebalancing : 한 consumer로 부터 다른 consumer로 파티션 소유권을 이전하는 것

새로운 consumer를 그룹에 추가할때, 특정 consumer에 문제가 생겨 중단될때 (consumer가 오랫동안 하트비트를 보내지 않으면 세션 타임아웃) 일어남. 

      리밸런싱 하는 동안에는 consumer들은 메세지를 읽을 수 없으므로 해당 그룹 전체가 잠시 사용 불가능함.

 

- commit 과  offset
    
    commit : 파티션 내부의 현재 위치를 변경하는 것
    
    offset : 컨슈머 자신이 읽는 레코드의 현재 위치 
    
    리밸런싱의 문제가 발생하면, 각 consumer는 이전과 다른 파티션을 할당받게 될 수 있다.
    
    이에 따라 메세지를 중복처리하거나 유실되는 경우가 있다. 
    
    특히,  consumer 를 구성할 때, enable.auto.commit=true 로 두면 아래와 같은 경우 발생가능성이 있음.
    
    1. 중복처리 경우

![스크린샷 2022-04-13 오후 10.42.29.png](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.42.29.png)

1. 유실되는 경우

![스크린샷 2022-04-13 오후 10.43.06.png](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_10.43.06.png)

- consumer 구성에서 중요한 configuration
    
    — auto.offset.reset : 커밋된 오프셋이 없는 파티션을 컨슈머가 읽기 시작할때, 또는 커밋된 오프셋이 있지만 유효하지 않을때, 컨슈머가 어떤 레코드를 읽을지 제어하는 매개변수
    
    latest(default) : (컨슈머가 실행 된 후 새로 추가된 레코드들) 을 읽음
    
    earliest : 해당 파티션의 맨 앞부터 모든 데이터를 읽음
    
    — enable.auto.commit : 컨슈머의 오프셋 커밋을 자동으로 할 것인지에 대한 제어
    
    true(default) : [auto.commit.interver.ms](http://auto.commit.interver.ms/) 로 자동으로 오프셋 커밋하는 시간 간격을 제어 할 수 있다.
    
                      속도가 가장 빠르고,  commit 관련 코드를 작성할 필요가 없는 장점이 있다.
    
    false :  commitSync,  commitAsync 사용 하여 offset commit을 제어함
    
    ![자동 커밋 상황](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_11.17.29.png)
    
    자동 커밋 상황
    

![자동 커밋 중 리밸런스가 일어났을 때 ](Kafka%20consumer%208aceb8fa99044c23b27a758396a6654c/%E1%84%89%E1%85%B3%E1%84%8F%E1%85%B3%E1%84%85%E1%85%B5%E1%86%AB%E1%84%89%E1%85%A3%E1%86%BA_2022-04-13_%E1%84%8B%E1%85%A9%E1%84%92%E1%85%AE_11.18.02.png)

자동 커밋 중 리밸런스가 일어났을 때 

- commitSync : 현재 오프셋 커밋
    
    — consumerRecord 처리순서를 보장한다. 
    
    — 가장느림 ( 커밋이 완료될 때 까지 block )
    
    — poll() method로 반환된 consumerRecord의 마지막 offset을 커밋한다. 
    
- commitAsync
    
    — commitSync 보다는 빠르다 ( 브로커의 commit 응답을 기다리는 대신, commit 요청을 전송하고 처리      를 계속 할 수 있음) 
    
    — 중복이 발생할 수 있다 ( 일시적인 통신문제로 이전 offset 보다 이후 offset 이 먼저 commit 이 될때 )
    
    — consumerRecord 처리 순서를 보장하지 못한다. 
    

---

- offset을 다루는 방법

1. Consumer Group의  offset 상태 확인

consumer group을 지정하고 `--describe`옵션을 사용하면 현재 consumer group의 offset 정보를 볼 수 있다. 명령어는 다음과 같다.

```bash
kafka-consumer-groups --bootstrap-server <host:port> --group <group.id> --describe
```

실행결과 예시

```bash
TOPIC         PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID                                      HOST            CLIENT-ID
example.topic 0          6392623366      6392623859      493             consumer-1-f6f6ffb0-1054-46b9-af13-0b254bc14da0  /10.64.69.95    consumer-1
example.topic 1          6394637143      6394637383      240             consumer-10-6c57b320-7742-4418-8e15-b7d735da346e /10.64.69.95    consumer-2
example.topic 2          6397170269      6397170495      226             consumer-19-dbed41a1-42bb-4ecb-bc8f-84e47c74dbe8 /10.64.69.95    consumer-3
example.topic 3          6397170269      6397170495      226             consumer-19-dbed41a1-42bb-4ecb-bc8f-84e47c74dbe8 /10.64.69.95    consumer-4
```

- TOPIC: 토픽 이름
- PARTITION: consumer group 내의 각 consumer가 할당된 파티션 번호
- CURRENT-OFFSET: 현재 consumer group의 consumer가 각 파티션에서 마지막으로 offset을 commit한 값
- LOG-END-OFFSET: producer쪽에서 마지막으로 생성한 레코드의 offset
- LAG: LOG-END-OFFSET에서 CURRENT-OFFSET를 뺀 값.

`--describe`를 통해 조회를 했을때 LAG이 계속 일정 수준을 유지한다면 consumer가 producer 가 만들어내는 이벤트 레코드의 양을 잘 따라가고있다는 것을 확인할 수 있다. 하지만 LAG이 계속 증가한다면 consumer의 처리 속도가 느린 것이기 때문에 consumer의 갯수를 충분히 증가시키거나, consumer의 로직을 더 간략화 해서 빠른 속도로 데이터 처리를 할 수 있도록 변경해야 한다.

1. Consumer Group의  offset reset

kafka에서 데이터를 불러와서 처리하는 과정에서 오류가 발생하거나 문제가 발견된 경우, 다시 원하는 offset부터 데이터를 재처리를 해야할 경우가 종종 있다. 이때 consumer group의 offset reset 기능을 활용하면 된다.

** consumer group이 실행중인 상태에 offset reset을 진행하는 경우 reset은 실패한다.

```bash
kafka-consumer-groups --bootstrap-server <host:port> --group <group> --topic <topic> --reset-offsets --to-earliest --execute
```

- `-topic` 대신 `-all-topics`를 지정하면 모든 토픽에 대해서 실행이 가능하다.
- `-execute` 옵션을 제거하고 실행하면 실제 반영되지 않고 어떻게 변할지 결과만 출력하는 dry run이 가능하다.

— offset 의 위치를 재설정 하기 위한 옵션 

- `-shift-by <Long: number-of-offsets>` 형식 (+/- 모두 가능)
- `-to-offset <Long: offset>`
- `-to-current`
- `-by-duration <String: duration>` : 형식 ‘PnDTnHnMnS’
- `-to-datetime <String: datetime>` : 형식 ‘YYYY-MM-DDTHH:mm:SS.sss’
- `-to-latest`
- `-to-earliest`

** `--to-datetime`의 경우 kafka에서 데이터를 읽어서 다른곳에 저장하는 중에 데이터 유실 또는 중복 write 등이 발생한 경우에 날짜 단위로 데이터를 다시 불러와서 재처리하고 싶은 경우 매우 유용하다.

예시 > 특정 topic의 파티션 1번만 offset을 30으로 지정하고 싶을때 

```bash
./kafka-consumer-groups --bootstrap-server {bootstrap 정보} \
--group click --topic {topic명}:1 \
--reset-offsets --to-offset 30 --execute
```