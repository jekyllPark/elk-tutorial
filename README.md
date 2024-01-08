# ELK 란?
ELK는 ElasticSearch, Logstash, Kibana의 약어로, 중앙 집중식 로깅, 로그 분석 및 시각화에 사용되는 강력한 오픈 소스 스택이다.

각 구성 요소는 아래와 같은 특정 목적을 수행한다.

## ElasticSearch
데이터를 저장하고 색인화하는 분산 검색 및 분석 엔진으로 이를 통해 대용량 데이터를 빠르고 효율적으로 검색, 집계, 분석 할 수 있다.

## Logstash
인덱싱을 위해 ElasticSearch로 보내기 전 다양한 소스에서 데이터를 수집, 필터링 및 변환하는 데이터 처리 파이프라인이다.

Logstash는 로그, 지표, 기타 이벤트 데이터 등 다양한 유형의 데이터를 처리할 수 있다.

## Kibana
ElasticSearch(이하 ES)와 함께 작동하는 웹 기반 데이터 시각화 및 탐색도구로써, ES에 저장된 데이터를 검색, 분석, 시각화하기 위한 사용자 친화적 인터페이스를 제공한다.

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/a735820d-371d-464a-90c4-7bd562530959)

# 튜토리얼

## 1. 스프링 부트 프로젝트 생성
```Slf4j``` 사용을 위한 롬복과 스프링 웹 및 로그를 json으로 파싱하여 ES에 전송하기 위한 ```logstash-logback-encoder```의 의존성을 추가한다.

```
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    implementation 'net.logstash.logback:logstash-logback-encoder:7.2'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
```
이후 간단한 헬로 컨트롤러를 통해 요청 발생 시 로그가 찍히도록 구성한다.
```
-- HelloController.java
@Slf4j
@RestController
public class HelloController {
    @GetMapping("/hello")
    public String hello() {
        log.info("Hello Controller's hello method invoked!");
        return "Hello! " + new Date();
    }

    @GetMapping("/test")
    public void test() {
        IntStream.rangeClosed(0, 100).forEach(v -> {
            log.info("count [{}]", v);
        });
    }
}
```

요청이 발생하면 로그 파일을 생성하기 위해 ```logback``` 을 만들어준다.

```
-- logback.xml
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
    <property name="MAX_FILE_SIZE" value="50MB"/>
    <property name="MAX_HISTORY" value="7"/>
    <property name="API_LOG_PATH_NAME" value="c:/works/test/API.log"/>
    <appender name="api" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>${API_LOG_PATH_NAME}</file>
        <encoder class="net.logstash.logback.encoder.LogstashEncoder">
            <provider class="net.logstash.logback.composite.loggingevent.ArgumentsJsonProvider"/>
        </encoder>
        <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
            <fileNamePattern>${API_LOG_PATH_NAME}_%d{yyyy-MM-dd}.%i.log</fileNamePattern>
            <maxFileSize>${MAX_FILE_SIZE}</maxFileSize>
            <maxHistory>${MAX_HISTORY}</maxHistory>
        </rollingPolicy>
        <layout class="ch.qos.logback.classic.PatternLayout">
            <pattern>%d{yyyy-MM-dd HH:mm:ss} | %m%n</pattern>
        </layout>
    </appender>
    <logger name="com.example.elktutorial.controller" level="INFO" additivity="false">
        <appender-ref ref="api"/>
    </logger>
</configuration>
```

여기까지 진행했다면, 우선 스프링 부트 프로젝트에서 해야될 작업은 끝났다.

컨트롤러에 선언한 URL에 요청을 할 경우 logback에 설정한 경로에 로그파일이 생긴다.

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/eaf19dd1-b9f1-4189-8a0d-55b4e3c43049)

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/439aa5d8-eb1b-45a8-9197-d82ba3400e8a)

## 2. ElasticSearch 설치 및 설정

- https://www.elastic.co/kr/downloads/elasticsearch

위 경로에서 OS에 맞는 ES를 다운로드 한다.

압축 파일을 푼 후 해당 폴더의 bin에 들어가 ```elasticsearch.bat``` 을 실행해준다.

최초 실행 시, ES 암호와 토큰을 발급해주는데 이를 별도로 저장해준다.

하지만 이 튜토리얼에서는 별도의 SSL/TLS 등의 설정없이 가볍게 각 스택 별 연동을 통해 시각화된 메트릭을 보는 것이 목적이니 보안 설정은 과감히 패스한다.

위 파일을 실행 시 아래 이미지와 같이 로그가 찍힐 것이다.

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/557fb4aa-2b59-4196-933a-04c0c49f106f)

9200 포트로 접근 시 아래와 같은 json 값을 반환 받았다면 이상 없이 정상 실행된 것이다.

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/8a39f09f-c139-41a9-9f81-8ddb078ebc12)

만약 제대로 실행이 되지 않는다면, config 폴더로 들어가 elasticsearch.yml 에서 xpack 관련 보안을 false로 전환해주면 된다. (로컬 테스트용이기 때문에 보안 끄는 것.)

```
-- elasticsearch.yml
# Enable security features
xpack.security.enabled: false

xpack.security.enrollment.enabled: false

# Enable encryption for HTTP API client connections, such as Kibana, Logstash, and Agents
xpack.security.http.ssl:
  enabled: false
  keystore.path: certs/http.p12

# Enable encryption and mutual authentication between cluster nodes
xpack.security.transport.ssl:
  enabled: false
  verification_mode: certificate
  keystore.path: certs/transport.p12
  truststore.path: certs/transport.p12
# Create a new cluster with the current node only
# Additional nodes can still join the cluster later
cluster.initial_master_nodes: ["DESKTOP-NVRBMAI"]

# Allow HTTP API connections from anywhere
# Connections are encrypted and require user authentication
http.host: 0.0.0.0
```

SSL/TLS은 bin 폴더의 elasticsearch-certutil.bat을 통해 생성할 수 있다.

여기까지 되었다면, 우선 ES에서 할 설정도 끝이다.

## 3. Kibana
- https://www.elastic.co/kr/downloads/kibana

위 링크에서 OS에 맞는 Kibana를 다운로드 한다.

동일하게 bin 폴더에 가서 kibana.bat을 실행해준다.

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/1f9f1e02-350d-4623-b932-5034c4d07d82)

반가운 메시지인 ```Kibana is now available```이 뜬다면 5601 포트를 통해 접속하여 아래 페이지에 접근할 수 있다.

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/4894f6f4-fb5c-4fc2-ac49-c5d04e17ec20)


만약~ 실행이 제대로 되지 않을 경우에 ES와 동일하게 config 폴더로 들어가 ```kibana.yml``` 파일을 실행시켜준다.

주석처리 되어있는 기본 값들을 풀어 해결할 수도 있다. (나는 이렇게 해서 됨..)

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/afab2d16-a832-416e-9098-59f254002672)

이제 마지막으로 ```Logstash``` 를 통해 스프링 프로젝트에서 발생한 로그와 파이프라인을 만들어준다.

## 4. Logstash

- https://www.elastic.co/kr/downloads/logstash
만들기 앞서 위 링크에서 OS에 맞는 Logstash를 다운로드 한다.

config 폴더를 들어가 logstash-sample.conf 를 적절하게 구성해준다. 

(https://www.elastic.co/guide/en/logstash/current/config-examples.html 참고)

```
--logstash-sample.conf
input {
	file {
		path => "C:/works/test/API.log"
		type => api
	}}
filter {
	dissect {
		mapping => {
			"message" => "%{timestamp} | %{message}"
		}
	}
	date {
		match => ["timestamp", "ISO8601", "YYYY-MM-dd HH:mm:ss,SSS"]
	}
	json {
		source => "message"
	}
}
output {
	if [type] == "api" {
		elasticsearch {
			codec => json
			hosts => ["http://localhost:9200","http://127.0.0.1:9200"]
			index => "api"
		}
	}
}
```

이제 터미널을 통해 Logstash 경로에 접근하여 아래 명령어를 통해 가동해준다.

```
.\logstash.bat -f ./config/logstash-sample.conf
```

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/f2626f76-a376-4d49-be88-29274f2f7e43)

위 쉘과 같이 Pipelines running 되면,

9600 포트를 통해 접속하여 ES와 같이 json 값을 리턴 받으면 이상없이 실행된 것이다.

![image](https://github.com/jekyllPark/elk-tutorial/assets/114489012/8e764329-15fa-4d10-8251-2c1e677aca47)

이제 모든 설정은 끝이 났고, 키바나를 통해 대시보드를 구성하기만 하면 된다.


# Ref
- https://medium.com/cloud-native-daily/elk-spring-boot-a-guide-to-local-configuration-b6d9fa7790f6
