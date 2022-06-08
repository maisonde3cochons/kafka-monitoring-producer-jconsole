## [JConsole을 이용한 Apache Kafka Monitoring 방법 - Producer 구성]

<br>

### [환경 구성도]

![image](https://user-images.githubusercontent.com/30817824/172522286-963c7072-b045-4711-b0e7-7863f30ad456.png)

(refer: https://github.com/freepsw/kafka-metrics-monitoring)

<br><br>


> ##  STG.01 GCP VM Instance 생성

<br>

####  GCP에 접속하여 VM Instance를 생성한다


#### client-01 이라는 이름의 인스턴스를 생성(Seoul Region > e2-standard-4)

![image](https://user-images.githubusercontent.com/30817824/172522602-e2f8c980-8abc-48ab-b9c5-d77c9c969365.png)

<br>

#### Boot Disk는 Centos로 선택

<br>

![image](https://user-images.githubusercontent.com/30817824/172509380-4ad29e2c-00ba-4ed0-aee9-f5ab8a5f5f1b.png)

<br><br>

> ## STG.02. Install & Configure

<br>


#### Java 설치 및 JAVA_HOME 설정
```
sudo yum install -y java

# 현재 OS 설정이 한글인지 영어인지 확인한다. 
alternatives --display java

# 아래와 같이 출력되면 한글임. 
슬레이브 unpack200.1.gz: /usr/share/man/man1/unpack200-java-1.8.0-openjdk-1.8.0.312.b07-1.el7_9.x86_64.1.gz
현재 '최고' 버전은 /usr/lib/jvm/java-1.8.0-openjdk-1.8.0.312.b07-1.el7_9.x86_64/jre/bin/java입니다.

### 한글인 경우 
alternatives --display java | grep '현재 /'| sed "s/현재 //" | sed 's|/bin/java로 링크되어 있습니다||'
export JAVA_HOME=$(alternatives --display java | grep '현재 /'| sed "s/현재 //" | sed 's|/bin/java로 링크되어 있습니다||')

### 영문인 경우
alternatives --display java | grep current | sed 's/link currently points to //' | sed 's|/bin/java||' | sed 's/^ //g'
export JAVA_HOME=$(alternatives --display java | grep current | sed 's/link currently points to //' | sed 's|/bin/java||' | sed 's/^ //g')

# 제대로 java 경로가 설정되었는지 확인
echo $JAVA_HOME
echo "export JAVA_HOME=$JAVA_HOME" >> ~/.bash_profile
source ~/.bash_profile
```

### Install the logstash(Kafka producer) and run logstash with the jmx option enabled
#### 1. Download the test file (tracks.csv)
- Procuder에서 broker로 전송할 데이터를 다운로드 받는다. 
```
curl -OL https://shared-penguin.s3.ap-northeast-2.amazonaws.com/Shared/tracks.csv
mv tracks.csv /tmp/tracks.csv
```

#### 2. Install the logstash and run
```
cd ~
curl -LO https://artifacts.elastic.co/downloads/logstash/logstash-oss-7.15.0-linux-x86_64.tar.gz
tar xvf logstash-oss-7.15.0-linux-x86_64.tar.gz  
rm -rf logstash-oss-7.15.0-linux-x86_64.tar.gz
cd logstash-7.15.0
```

#### 3. Configure the logstash.yml
- kafka producer로 동작하기 위해서, 일정 주기로 일정 메세지를 전달하도록 설정 필요.
- logstash를 기본으로 사용하면, 전송 성능을 위해서 정의된 batch size 만큼을 보내도록 설정됨. 
- 따라서 본 실습을 위해서는 1개의 thread가 한번에 1개의 메세지만 보내도록 설정한다. 
- 관련 configuration
```yml
# logstash에서 동시에 실행 가능한 thread
pipeline.workers: 2
# logstash에서 한번에 전송할 batch size
pipeline.batch.size: 125
```

- logstash 실행시 적용 가능한 option 
```
thread 게수 : -w 또는 --pipeline.workers
batch 개수 : -b, --pipeline.batch.siz
```


#### 4. Run the logstash 
```
## Add producer config file 
mkdir ~/logstash_conf
```


> #### vi ~/logstash_conf/producer.conf
```
input {
    file {
        path=>"/tmp/tracks*.csv"
        start_position => "beginning"
    } 
}

filter {
    sleep {
        time => "1"   # Sleep 1 second
        every => 1   # on every 1th event
    }
}

output {
    stdout{
        codec => rubydebug
    }

    kafka { 
        bootstrap_servers => "broker-01:9092"
        topic_id =>  "kafka-mon"
        codec => plain {
            format => "%{message}"
        }
        # compression_type => "snappy"
    }
}
```

```
## Run logstash as a kafka producer
export LS_JAVA_OPTS='-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false 
  -Dcom.sun.management.jmxremote.ssl=false 
  -Dcom.sun.management.jmxremote.port=9998 
  -Dcom.sun.management.jmxremote.rmi.port=9998 
  -Djava.rmi.server.hostname=34.64.200.217'

## thread 1개, batch size 1개로 데이터를 kafka로 전송
> ~/logstash-7.15.0/bin/logstash -w 1 -b 1 -f ~/logstash_conf/producer.conf
```

#### extra. LS_JAVA_OPT 설정을 logstash 시작시에 기본으로 적용하는 방법
```
vi logstash-7.15.0/config/jvm.options

## 위 파일 내용에 아래 jmx 관련된 내용을 LS_JAVA_OPTS에 추가
-Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.port=9998 -Dcom.sun.management.jmxremote.rmi.port=9998 -Djava.rmi.server.hostname=34.64.200.217
```


#### 5. GCP VPC Network Firewall 설정 (9998 port 허용)

![image](https://user-images.githubusercontent.com/30817824/172518925-0a960d8a-8a56-4d92-9af6-bf24231fcf53.png)

![image](https://user-images.githubusercontent.com/30817824/172518946-6441c4be-f390-4cba-a2b7-b78768e90f4e.png)



#### 6. JConsole에서 확인

- VS Code Terminal에서 jdk/bin dir에 있는 jconsole.exe 실행 (jdk 설치 후)

![image](https://user-images.githubusercontent.com/30817824/172519146-4645b5d8-b14c-4c76-a70f-5f1acd2ca8fe.png)


```
./jconsole.exe
```

#### 7. JConsole 접속 (GCP VM Instance의 External IP + :9998 포트 입력)

![image](https://user-images.githubusercontent.com/30817824/172538330-747e74b9-dbce-4a53-b171-d17f41b02577.png)


#### 8. MBeans에서 Kafka Producer의 각종 metrics들을 확인할 수 있다

![image](https://user-images.githubusercontent.com/30817824/172538435-7a558357-ae32-4553-960d-094fb496a1a5.png)

