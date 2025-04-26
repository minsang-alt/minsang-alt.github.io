---
title: spring에서 MySQL의 master/slave 구성하기
date: 2024-09-22 10:19:10 +/-0800
categories: [Tech Notes, Spring]
tags: spring    # TAG names should always be lowercase
---

### 0. 실행 환경

- SpringBoot 3.3.2
- MySQL 8.1 (Docker 이용)
- Mac OS

### 1. DockerCompose 구성

Docker Compose 파일은 MySQL 마스터와 슬레이브 인스턴스를 설정합니다.

```java
version: '3'

services:
  mysql-master:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydb
      MYSQL_USER: myuser
      MYSQL_PASSWORD: mypassword
    volumes:
      - ./master/my.cnf:/etc/mysql/my.cnf
      - ./master/data:/var/lib/mysql
    ports:
      - "3306:3306"

  mysql-slave:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: rootpassword
      MYSQL_DATABASE: mydb
      MYSQL_USER: myuser
      MYSQL_PASSWORD: mypassword
    volumes:
      - ./slave/my.cnf:/etc/mysql/my.cnf
      - ./slave/data:/var/lib/mysql
    ports:
      - "3307:3306"
    depends_on:
      - mysql-master
```

### 2. 마스터와 슬레이브를 위한 설정 파일을 생성

**마스터 (./master/my.cnf)**

```text
[mysqld]
server-id=1
log-bin=mysql-bin
binlog-format=ROW
```

**슬레이브 (./slave/my.cnf)**

```text
[mysqld]
server-id=2
log-bin=mysql-bin
binlog-format=ROW
relay-log=relay-bin
read-only=1
```

### 3. Docker Compose 파일을 실행

```shell
docker-compose up -d
```

### 4. 마스터 서버에 접속하여 복제 사용자를 생성

```shell
docker exec -it mysql-master mysql -u root -p

CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'slavepass';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
```
**마지막 명령의 결과를 기록합니다. 다음 슬레이브 접속때 LOG_FILE 및 POS를 지정해야합니다.**

![](/assets/img/spring-replication/img.png)

### 5. 슬레이브 서버에 접속하여 복제를 설정

```shell
docker exec -it mysql-slave mysql -u root -p

CHANGE MASTER TO
MASTER_HOST='mysql-master',
MASTER_USER='repl',
MASTER_PASSWORD='slavepass',
MASTER_LOG_FILE='mysql-bin.000003',
MASTER_LOG_POS=827;
START SLAVE;
SHOW SLAVE STATUS\G
```

![](/assets/img/spring-replication/img_1.png)

마스터 서버에서 테스트 디비를 생성하고 데이터를 넣은다음, 슬레이브 서버에서 복제가 되어있다면 성공입니다. 

### 6. SpringBoot application.yml 설정

```yaml
spring:
  datasource:
    master:
      jdbc-url: jdbc:mysql://localhost:3306/bookchallnegedb
      username: myuser
      password: mypassword
      driver-class-name: com.mysql.cj.jdbc.Driver
    slave:
      jdbc-url: jdbc:mysql://localhost:3307/bookchallnegedb
      username: myuser
      password: mypassword
      driver-class-name: com.mysql.cj.jdbc.Driver
```

### 7. 마스터/슬레이브 DataSource 구성

**1. DataSourceConfig.java**

```java
@Configuration
@EnableTransactionManagement
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    @ConfigurationProperties(prefix = "spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean
    public DataSource routingDataSource(@Qualifier("masterDataSource") DataSource masterDataSource,
                                        @Qualifier("slaveDataSource") DataSource slaveDataSource) {
        RoutingDataSource routingDataSource = new RoutingDataSource();

        Map<Object, Object> dataSourceMap = new HashMap<>();
        dataSourceMap.put(DataSourceType.MASTER, masterDataSource);
        dataSourceMap.put(DataSourceType.SLAVE, slaveDataSource);

        routingDataSource.setTargetDataSources(dataSourceMap);
        routingDataSource.setDefaultTargetDataSource(masterDataSource);

        return routingDataSource;
    }


    @Primary
    @Bean
    public DataSource dataSource(@Qualifier("routingDataSource") DataSource routingDataSource) {
        return new LazyConnectionDataSourceProxy(routingDataSource);
    }

}
```

**2. DataSourceType**

```java
public enum DataSourceType {
    MASTER, SLAVE
}
```

**3. RoutingDataSource**

```java
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
            ? DataSourceType.SLAVE : DataSourceType.MASTER;
    }

}
```
**4. DataSourceContextHolder**

```java
public class DataSourceContextHolder {

    private static final ThreadLocal<DataSourceType> contextHolder = new ThreadLocal<>();

    public static void setDataSourceType(DataSourceType dataSourceType) {
        contextHolder.set(dataSourceType);
    }

    public static DataSourceType getDataSourceType() {
        return contextHolder.get();
    }

    public static void clearDataSourceType() {
        contextHolder.remove();
    }

}
```

### 8. 잘 동작하는 지 테스트

```java
@RestController
public class DataManipulationTestController {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    @PostMapping("/create-user")
    @Transactional
    public String createUser(@RequestParam String name) {
        jdbcTemplate.update("INSERT INTO users (name) VALUES (?)", name);
        return "User created: " + name;
    }

    @GetMapping("/get-users")
    @Transactional(readOnly = true)
    public String getUsers() {
        String containerId = jdbcTemplate.queryForObject("SELECT @@hostname", String.class); // 컨테이너아이디 확인해보기
        List<Map<String, Object>> users = jdbcTemplate.queryForList("SELECT * FROM users");

        return String.format("Container ID: %s\nUsers: %s", containerId, users.toString());
    }
}
```
