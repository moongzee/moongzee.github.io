---
title: consumer-lag-monitoring
date: 2022-09-14 00:00:00 +0900
categories: [Stream Data Processing, mirrormaker consumer lag]
tags: [kafka, kafka consumer, mirrormaker, consumer lag, stream Data processing]
pin: true
---

# MirrorMaker consumer lag 모니터링
---


<h2 data-toc-skip>목차</h2>

1. Consumer lag 이란?
2. MirrorMaker Consumer lag 모니터링 프로그램 설명
3. Lag_monitor.py 설명

---
<br>
<h3 data-toc-skip>1. Consumer lag 이란?</h3>

Kafka 의 producer가 보낸 메세지의 offset과 consumer가 받은 메세지의 offset 차이로, consumer의 상태 모니터링에 사용하는 지표 중 하나이다.
<br>

<h3 data-toc-skip>2. MirrorMaker Consumer lag 모니터링 프로그램 설명</h3>  

연동하는 토픽 별 MirrorMaker Consumer lag 모니터링을 한 곳에 할 수 있다.

Kibana 대시보드를 이용해 Consumer lag의 상태를 차트로 볼 수 있다.
<br>
<br>

<h4 data-toc-skip>모니터링 프로그램 구성</h4>
<br>
![Monitoring_Program_composition](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-14-consumer-lag-monitor/1.png?raw=true)
<br>
미러링하는 인스턴스의 consumer lag 정보를 Lag_monitor.py (모니터링을 위해 별도로 작성한 프로그램)을 통해 Elasticsearch로 보내고, Kibana 대시보드를 이용해 모니터링 한다. 
<br>


<h3 data-toc-skip>3. Lag_monitor.py 설명</h3>  

![Lag_monitor.py](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-14-consumer-lag-monitor/2.png?raw=true)

<h4 data-toc-skip>Lag_monitor.py의 메인함수 프로세스는 3단계이다.</h4>

1. Elasticsearch 연결
2. Elasticsearch Index 생성
3. 파이프라인 함수 실행 

> 여기서, 파이프라인 함수는 연동중인 토픽 별로 thread 실행을 한다.
{: .prompt-tip }


![pipeline_process](https://github.com/moongzee/moongzee.github.io/blob/main/assets/images/posts/2022-09-14-consumer-lag-monitor/3.png?raw=true)

<h4 data-toc-skip>Lag_monitor.py의 Pipeline 함수 프로세스는 4단계이다.</h4>

1. Consumer Lag 출력하는 command line 명령어 생성 함수를 실행
2. mirrormaker instance에 SSH 원격접속 
3. 1. 에서 생성한 Consumer Lag을 출력하는 command line 명령어 실행
4. Consumer Lag 출력결과를 elasticsearch로 전송



<h4 data-toc-skip>Lag_monitor.py 소스코드 설명</h4>
    
<모듈 import> Lag_monitor.py를 실행하기 위한 모듈을 불러온다.
    
```python
import time
from datetime import datetime
import json
import threading
import paramiko
from elasticsearch import Elasticsearch
```
    
<h4 data-toc-skip>1. Elasticsearch 연결</h4> 
    
Elasticsearch 모듈을 이용해 oasis-elasticsearch 서버와 연결을 한다.
<br>    

```python
## connect elasticsearch
es = Elasticsearch(['http://es-node1-ip:9200',
					'http://es-node2-ip:9200',
					'http://es-node3-ip:9200'], 
				basic_auth=('ID','PW'))
```
<br>    

#### 2. Elasticsearch Index 생성
    
Consumer lag 정보를 저장할 index를 생성한다. 
<br>    

```python
## create index
if es.indices.exists(index='consumer_lag') :
        pass
else : 
   with open('mapping.json', 'r') as f :
        mapping = json.load(f)
   es.indices.create (index='consumer_lag', body=mapping)
```
<br>
index명은 consumer_lag으로 지정하였고, 만약 ‘consumer_lag’ index가 존재하면 index 생성과정을 넘긴다. <br>
‘consumer_lag’ index의 schema 정보를 mapping.json 파일에 작성하였고, index 생성시 mapping.json 파일을 읽어 index의 schema 정보를 지정한다. <br>
    

** mapping.json

```json
{
    "mappings":{
        "properties":{
            "@timestamp":{"type" : "date"},
						"topic":{"type" : "text"},
						"p0":{"type" : "integer"},
						"p1":{"type" : "integer"},
						"p2":{"type" : "integer"},
						"p3":{"type" : "integer"},
						"p4":{"type" : "integer"},
						"p5":{"type" : "integer"},
						"p6":{"type" : "integer"},
						"p7":{"type" : "integer"},
						"p8":{"type" : "integer"},
						"p9":{"type" : "integer"},
						"p10":{"type" : "integer"},
						"p11":{"type" : "integer"},
						"p12":{"type" : "integer"},
						"p13":{"type" : "integer"},
						"p14":{"type" : "integer"},
						"p15":{"type" : "integer"},
						"p16":{"type" : "integer"},
						"p17":{"type" : "integer"}
        }
    }
}
```
    
<h4 data-toc-skip>3. 파이프라인 함수 실행</h4>
    
파이프 라인 함수
<br>

```python
def lag_pipeline(ssh_key,es_conn,hostname,topic_name,monitor_cli,run_event):
    # SSH client 연결
		client = paramiko.SSHClient()
    client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
    client.connect( hostname=hostname, username='ec2-user', pkey=ssh_key )
	  # SSH 연결 후 consumer lag 지표 출력 shell script 명령어 실행에 따른 결과를 가져옴 
		stdin, stdout, stderr = client.exec_command(monitor_cli)
    
		while True:
        line = stdout.readline()
        if not line:
            break
				# consumer lag 지표를 받아, 전처리 후 elasticsearch로 보냄
        info = [datetime.utcnow().strftime('%Y-%m-%dT%H:%M:%S.%f')[:-3]+'Z',topic_name]
        lag = line.split('\n')[0].split(' ')
        dict1 = dict(zip(['@timestamp','topic'], info))   
        dict2 = {'p'+str(i) : lag[i] for i in range(len(lag))}
        document = dict(dict1, **dict2)
        es_conn.index(index='consumer_lag', body= document)
```
<br>

<h4 data-toc-skip>Input parameter 설명</h4>
    
ssh_key 
: mirrormaker instance ssh 접속에 필요한 key (ec2 pem file을 RSA key로 변환한 값)

es_conn 
: consumer lag의 값을 elasticsearch로 보내기 위해 elasticsearch와 연결한 client

hostname 
: mirrormaker instance ip 주소

topic_name 
: consumer lag 지표를 수집하는 대상 토픽 명

monitor_cli 
: consumer lag 지표를 출력하는 shell script 명령어
<br>

<h4 data-toc-skip>consumer lag 지표 출력하는 shell script 명령어 생성 함수</h4>
    
```python
def lag_export_cli(source_bootstrap_server, mm_consumer_properties_file_nm, topic):
    cli_1 = "while sleep 1; do echo -e $(/home/ec2-user/kafka_2.12-2.6.2/bin/kafka-consumer-groups.sh --bootstrap-server "
    cli_2 = " --group oasis-group --describe --command-config /home/ec2-user/kafka_2.12-2.6.2/config/"
    cli_3 = " --describe 2> /dev/null | grep "
    cli_4 = " | sed 's/\s\+/\\t/g' | cut -f 6 | xargs); done"
    cli = cli_1+str(source_bootstrap_server)+cli_2+str(mm_consumer_properties_file_nm)+cli_3+str(topic)+cli_4
    return cli
```
<br>    
    
<h4 data-toc-skip>Input parameter 설명</h4>
    
source_bootstrap_server 
: 토픽 연동하는 source kafka의 bootstrap server 주소 값

mm_consumer_properties_file_nm 
: mirrormaker에서 사용하는 consumer properties 파일명

topic 
: 미러링 대상 토픽명
    

<h4 data-toc-skip>메인함수 내 파이프라인 함수 실행</h4> 
    
```python
# 1. ec2 ssh 접속을 위한 key 생성
key_path = 'ec2 pemfile path'
ssh_key = paramiko.RSAKey.from_private_key_file(key_path)

# 2. lag 추출 cli 명령어 생성
t1_cli = lag_export_cli(source_bootstrap_server="source kafka bootstrap server",
						mm_consumer_properties_file_nm="mirrormaker consumer properties file name",
						topic="topic name"

# 3. pipeline 함수 thread 실행
t1 = threading.Tread(target=lag_pipeline, 
					 args=(ssh_key, es, 'ec2 mirrormaker ip', 'topic name', t1_cli)

t1.start()
```
<br>
    
미러링 대상 토픽별로 source kafka bootstrap, topic 명, mirrormaker 인스턴스 ip주소, mirrormaker consumer properties 파일명을 수정하여 pipeline 함수 thread를 생성한다. 
    
<h4 data-toc-skip>1. ec2 ssh 접속을 위한 key 생성</h4>

ssh_key 변수에 ec2 접속을 위한 pem file을 이용해 RSA key로 변환한 값을 할당한다.<br>
** pem file 경로 : /home/ec2-user/launcher/pemfile/ec2_keypair_dev.pem<br>
    
<h4 data-toc-skip>2. lag_export_cli 함수 실행</h4> 
    
인자값으로 source kafka bootstrap_server 주소, mirror maker consumer properties file 명, 토픽 명을 입력하여<br> 
consumer lag 추출하는 command line 명령어를 생성후 t1_cli 변수에 할당한다.
    
<h4 data-toc-skip>3. 파이프라인 함수를 thread로 실행</h4>

threading 모듈의 thread 함수를 사용하며,인자값으로는 target 과 args 가 있다.<br>
    
target 
: thread 실행할 함수명 즉, lag_pipeline을 입력한다
    
args 
: lag_pipeline 함수의 인자값 (ssh_key , es_conn , hostname , topic_name , monitor_cli )
    
을 차례대로 입력한다.