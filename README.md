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

# Ref
- https://medium.com/cloud-native-daily/elk-spring-boot-a-guide-to-local-configuration-b6d9fa7790f6
