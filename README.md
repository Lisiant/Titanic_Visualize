# [우리FISA 3기 클라우드 엔지니어링] 타이타닉 데이터 시각화 프로젝트
## 팀원
|<img src="https://avatars.githubusercontent.com/u/104816148?v=4" width="100" height="100"/>|<img src="https://avatars.githubusercontent.com/u/79884688?v=4" width="100" height="100"/>|<img src="https://avatars.githubusercontent.com/u/90691610?v=4" width="100" height="100"/>|<img src="https://avatars.githubusercontent.com/u/127733525?v=4" width="100" height="100"/>|
|:-:|:-:|:-:|:-:|
|[@hyleei](https://github.com/hyleei)|박장우<br/>[@Lisiant](https://github.com/Lisiant)|SeokCheol Lee<br/>[@SeokCheol-Lee](https://github.com/SeokCheol-Lee)|[@dkac0012](https://github.com/dkac0012)|


# 개요

ELK Stack과 MySQL을 연동하여 타이타닉 생존자 데이터 시각화

## 환경

- 사용 데이터: https://www.kaggle.com/code/alexisbcook/titanic-tutorial
- OS: Ubuntu 22.04 LTS Server
- DB: MySQL 8.0.37-ubuntu.22.04.3 for Linux
- ELK Stack

# 전처리

### 결측치 대체

데이터를 살펴본 결과 `age` 컬럼과 `cabin` 컬럼에 `null` 값이 많은 것을 확인하여 전처리 과정을 진행하였습니다.

- `age` 컬럼: null 값을 `-1` 로 대체
- `cabin` 컬럼: null 값을 `"unknown"` 으로 대체

### Logstash `.conf` 파일 수정

Logstash의 작동 과정을 기술한 `.conf` 파일에 

```bash
input {
  jdbc {
    jdbc_driver_library => "/etc/logstash/mysql-connector-java-8.0.30.jar"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://ip_address:3306/database_name"
    jdbc_user => (mysql_username)
    jdbc_password => (mysql_password)
    statement => "SELECT * FROM titanic_raw"
  }
}

filter {
  ruby {
    code => "event.set('cabin', 'unknown') if event.get('cabin').nil?"
  }
  ruby {
    code => "event.set('age', -1) if event.get('age').nil?"
  }
}

output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "titanic"
  }
  stdout { codec => rubydebug }
}
```

# 필터링 아이디어

1. 컬럼 별 결측치 비율 계산
2. 어떤 나이의 사람들이 탑승했는지 비율 
3. 항구 별 생존자 비율

## 가설

가설 설정 → 데이터 시각화
