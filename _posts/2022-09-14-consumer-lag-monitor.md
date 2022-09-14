# 실시간 시스템 모니터링

**[목차]**

1. MSK 모니터링
    
    1.1 Cloudwatch Dashboard를 이용한 MSK 모니터링
    
    1.2 생성된 MSK 모니터링 Dashboard 정보
    
2. MirrorMaker Consumer lag 모니터링
    
    2.1 Consumer lag이란?
    
    2.2 모니터링 프로그램 설명
    
    2.3 Lag_monitor.py 설명
    
    2.4 모니터링 프로그램 정보 
    

### 1. MSK 모니터링

**1.1 Cloudwatch Dashborad 를 이용한 MSK 모니터링**

- 별도의 설치 및 구성과정 없이, MSK 내 monitoring metrics의 cloudwatch 설정으로 MSK 모니터링에 필요한 지표를 수집할 수 있다.

- MSK 모니터링에 수집한 지표를 cloudwatch dashboard를 이용해 시각화가 가능하다.

 

**1.2 생성된 MSK 모니터링 Dashboard 정보**

![Untitled](%E1%84%89%E1%85%B5%E1%86%AF%E1%84%89%E1%85%B5%E1%84%80%E1%85%A1%E1%86%AB%20%E1%84%89%E1%85%B5%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%86%B7%20%E1%84%86%E1%85%A9%E1%84%82%E1%85%B5%E1%84%90%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20e53518fcbc0f4845820bf81ec2034e69/Untitled.png)

- Dashboard Name : kafka-monitor-all-in-one
- 수집 지표 :
    
     - MirrorMaker 인스턴스의 Network 유입량
    
     - MirrorMaker 인스턴스의 Cpu 사용률
    
     - topic별 MSK내 유입되는 message의 초당건수  
    

### 2. MirrorMaker Consumer lag 모니터링

**2.1 Consumer lag 이란?** 

Kafka 의 producer가 보낸 메세지의 offset과 consumer가 받은 메세지의 offset 차이로, consumer의 상태 모니터링에 사용하는 지표 중 하나이다.

**2.2 모니터링 프로그램 설명** 

- 연동하는 토픽 별 MirrorMaker Consumer lag 모니터링을 한 곳에 할 수 있다.
- Kibana 대시보드를 이용해 Consumer lag의 상태를 차트로 볼 수 있다.

 모니터링 프로그램 구성

![Untitled](%E1%84%89%E1%85%B5%E1%86%AF%E1%84%89%E1%85%B5%E1%84%80%E1%85%A1%E1%86%AB%20%E1%84%89%E1%85%B5%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%86%B7%20%E1%84%86%E1%85%A9%E1%84%82%E1%85%B5%E1%84%90%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20e53518fcbc0f4845820bf81ec2034e69/Untitled%201.png)

미러링하는 인스턴스의 consumer lag 정보를 Lag_monitor.py (모니터링을 위해 별도로 작성한 프로그램)을 통해  Elasticsearch로 보내고, Kibana 대시보드를 이용해 모니터링 한다. 

** Lag_monitor.py 는 ec2-oasis-prd-streamlauncher 인스턴스 에서 실행

**2.3 Lag_monitor.py 설명** 

![Untitled](%E1%84%89%E1%85%B5%E1%86%AF%E1%84%89%E1%85%B5%E1%84%80%E1%85%A1%E1%86%AB%20%E1%84%89%E1%85%B5%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%86%B7%20%E1%84%86%E1%85%A9%E1%84%82%E1%85%B5%E1%84%90%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20e53518fcbc0f4845820bf81ec2034e69/Untitled%202.png)

Lag_monitor.py의 메인함수 프로세스는 3단계이다.

1. Elasticsearch 연결
2. Elasticsearch Index 생성
3. 파이프라인 함수 실행 

** 여기서, 파이프라인 함수는 현재 연동중인 토픽 별로 thread 실행을 한다.

( 연동중인 토픽이 총 5개로, 5개의 thread를 생성하여 파이프라인 함수를 실행 )

![Untitled](%E1%84%89%E1%85%B5%E1%86%AF%E1%84%89%E1%85%B5%E1%84%80%E1%85%A1%E1%86%AB%20%E1%84%89%E1%85%B5%E1%84%89%E1%85%B3%E1%84%90%E1%85%A6%E1%86%B7%20%E1%84%86%E1%85%A9%E1%84%82%E1%85%B5%E1%84%90%E1%85%A5%E1%84%85%E1%85%B5%E1%86%BC%20e53518fcbc0f4845820bf81ec2034e69/Untitled%203.png)

Lag_monitor.py의 Pipeline 함수 프로세스는 4단계이다.

1. Consumer Lag 출력하는 command line 명령어 생성 함수를 실행
2. mirrormaker instance에 SSH 원격접속 
3. 1. 에서 생성한 Consumer Lag을 출력하는 command line 명령어 실행
4. Consumer Lag 출력결과를 elasticsearch로 전송

- Lag_monitor.py 소스코드 설명
    
    <모듈 import> Lag_monitor.py를 실행하기 위한 모듈을 불러온다.
    
    ```bash
    import time
    from datetime import datetime
    import json
    import threading
    import paramiko
    from elasticsearch import Elasticsearch
    ```
    
    1. Elasticsearch 연결 
    
    Elasticsearch 모듈을 이용해 oasis-elasticsearch 서버와 연결을 한다.
    
    ```bash
    ## connect elasticsearch
    es = Elasticsearch(['http://es-node1-ip:9200',
    										'http://es-node2-ip:9200',
    										'http://es-node3-ip:9200'], 
    				basic_auth=('ID','PW'))
    ```
    
    1. Elasticsearch Index 생성
    
    Consumer lag 정보를 저장할 index를 생성한다. 
    
    ```bash
    ## create index
    if es.indices.exists(index='consumer_lag') :
            pass
    else : 
       with open('mapping.json', 'r') as f :
            mapping = json.load(f)
       es.indices.create (index='consumer_lag', body=mapping)
    ```
    
    — index명은 consumer_lag으로 지정하였고, 만약 ‘consumer_lag’ index가 존재하면 index 생성과정을 넘긴다. 
    
    — ‘consumer_lag’ index의 schema 정보를 mapping.json 파일에 작성하였고, index 생성시 mapping.json 파일을 읽어 index의 schema 정보를 지정한다. 
    
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
    
    1. 파이프라인 함수 실행
    
    — 파이프 라인 함수
    
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
    
    # Input parameter 설명
    
    ssh_key : mirrormaker instance ssh 접속에 필요한 key (ec2 pem file을 RSA key로 변환한 값)
    
    es_conn : consumer lag의 값을 elasticsearch로 보내기 위해 elasticsearch와 연결한 client
    
    hostname : mirrormaker instance ip 주소
    
    topic_name : consumer lag 지표를 수집하는 대상 토픽 명
    
    monitor_cli : consumer lag 지표를 출력하는 shell script 명령어
    
    — consumer lag 지표 출력하는 shell script 명령어 생성 함수
    
    ```python
    def lag_export_cli(source_bootstrap_server, mm_consumer_properties_file_nm, topic):
        cli_1 = "while sleep 1; 
    						do echo -e $(/home/ec2-user/kafka_2.12-2.6.2/bin/kafka-consumer-groups.sh 
    						--bootstrap-server "
        cli_2 = " --group oasis-group --describe 
    							--command-config /home/ec2-user/kafka_2.12-2.6.2/config/"
        cli_3 = " --describe 2> /dev/null | grep "
        cli_4 = " | sed 's/\s\+/\\t/g' | cut -f 6 | xargs); done"
        cli = cli_1+str(source_bootstrap_server)+cli_2
    					+str(mm_consumer_properties_file_nm)+cli_3+str(topic)+cli_4
        return cli
    ```
    
    # Input parameter 설명
    
    source_bootstrap_server : 토픽 연동하는 source kafka의 bootstrap server 주소 값
    
    mm_consumer_properties_file_nm : mirrormaker에서 사용하는 consumer properties 파일명
    
    topic : 미러링 대상 토픽명
    
    — 메인함수 내 파이프라인 함수 실행 
    
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
    
    미러링 대상 토픽별로 source kafka bootstrap, topic 명, mirrormaker 인스턴스 ip주소, mirrormaker consumer properties 파일명을 수정하여 pipeline 함수 thread를 생성한다. 
    
    #1. ec2 ssh 접속을 위한 key 생성
    
    ssh_key 변수에 ec2 접속을 위한 pem file을 이용해 RSA key로 변환한 값을 할당한다.
    
    ** pem file 경로 : /home/ec2-user/launcher/pemfile/ec2_keypair_dev.pem
    
    #2. lag_export_cli 함수에 인자값으로, 
    
    source kafka bootstrap_server 주소, mirror maker consumer properties file 명, 토픽 명을 입력하여 consumer lag 추출하는 command line 명령어를 생성후 t1_cli 변수에 할당한다.
    
    #3.  파이프라인 함수를 thread로 실행하기 위해 threading 모듈의 thread 함수를 사용하며,
    
    인자값으로는 target 과 args 가 있다.
    
    target : thread 실행할 함수명 즉, lag_pipeline을 입력한다
    
    args : lag_pipeline 함수의 인자값 (ssh_key , es_conn , hostname , topic_name , monitor_cli )
    
        을 차례대로 입력한다.
    

### 3. 모니터링 프로그램 정보

— 프로그램 정보

- lag_monitor.py 경로 : (ec2-oasis-prd-streamlauncher) 내 /home/ec2-user/lag_monitor/
    
    lag_monitor 폴더 구성
    
    - home/ec2-user/lag_monitor/
        
         lag_monitor.py : consuemr lag 정보를 elasticsearch로 보내는 파이프라인 프로그램
        
         mapping.json : consumer_lag index 생성에 필요한 schema 정보 json 
        
        run_monitor.sh : lag_monitor.py실행 shell script
        

— elasticsearch 내 consumer lag 관련 index 정보 

- index명 : consumer_lag
- shcema :
    
    ```json
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
    ```
    

— 대시보드 정보

- Kibana 접속주소 : “10.241.102.4:5601”
- 대시보드 space명 : mm-lag monitor
- 대시보드 구성 : 연동 토픽의 consumer lag 정보
    
    ** 연동 토픽 : topic-navilog2-post-json , HEC_H_RECV_01, btvplus-post-log-json, topic-lgs5.0-json, topic-skt-search-log-json