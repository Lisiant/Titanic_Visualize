# 🚢 [우리FISA 3기 클라우드 엔지니어링] 타이타닉 데이터 시각화 프로젝트

# 📚개요

ELK Stack과 MySQL을 연동하여 타이타닉 생존자 데이터를 시각화하고, 인사이트를 도출하는 프로젝트

## 👥 팀원

| <img src="https://avatars.githubusercontent.com/u/104816148?v=4" width="100" height="100"/> | <img src="https://avatars.githubusercontent.com/u/79884688?v=4" width="100" height="100"/> | <img src="https://avatars.githubusercontent.com/u/90691610?v=4" width="100" height="100"/> | <img src="https://avatars.githubusercontent.com/u/127733525?v=4" width="100" height="100"/> |
| --- | --- | --- | --- |
| 박현서<br/>https://github.com/hyleei | 박장우<br/>https://github.com/Lisiant | 이석철<br/>https://github.com/SeokCheol-Lee | 구동길<br/>https://github.com/dkac0012 |

## ⚙️ 개발 환경

- 사용 데이터: https://www.kaggle.com/code/alexisbcook/titanic-tutorial
- OS: Ubuntu 22.04 LTS Server
- DB: MySQL 8.0.37-ubuntu.22.04.3 for Linux
- ELK Stack 7.17.22

## 📖 Data Dictionary

| Name | description |
| --- | --- |
| Survived | 0 = 사망, 1 = 생존 |
| Pclass | 1 = 1등석, 2 = 2등석, 3 = 3등석 |
| Gender | male = 남성, female = 여성 |
| Age | 나이 |
| SibSp | 타이타닉 호에 동승한 자매/배우자의 수 |
| Parch | 타이타닉 호에 동승한 부모/자식의 수 |
| Ticket | 티켓 번호 |
| Embarked | 탑승지, C = 셰르부르, Q = 퀸즈타운, S = 사우샘프턴 |
| Cabin | 방 호수 |
| Fare | 승객 요금 |

# 🔧전처리

### 결측치 대체

데이터를 살펴본 결과 `age` 컬럼과 `cabin` 컬럼에 `null` 값이 많은 것을 확인하여 전처리 과정을 진행하였습니다.

- `age` 컬럼: null 값을 `1` 로 대체
- `cabin` 컬럼: null 값을 `"unknown"` 으로 대체
- `fare` 컬럼:  null값 평균값(33.3)으로 대체
- `embarked` 컬럼: null값 최빈값(S)로 대체

### Logstash 필터 추가

Logstash의 작동 과정을 기술한 `.conf` 파일에 Filter를 적용하여 결측치를 대체하였습니다.

```bash
input {
  jdbc {
    jdbc_driver_library => "/usr/share/logstash/logstash-core/lib/jars/mysql-connector-java.jar"
    jdbc_driver_class => "com.mysql.cj.jdbc.Driver"
    jdbc_connection_string => "jdbc:mysql://localhost:3306/fisa"
    jdbc_user => # ...
    jdbc_password => # ...
    statement => "SELECT * FROM titanic_raw"
  }
}

filter {
	// cabin null값 unknown으로 대체
  ruby {
    code => "event.set('cabin', 'unknown') if event.get('cabin').nil?"
  }
  // age null값 -1로 대체
  ruby {
    code => "event.set('age', -1) if event.get('age').nil?"
  }
  // fare null값 평균값(33.3)으로 대체
  ruby {
    code => "
      avg_fare = 33.3
      event.set('fare', avg_fare) if event.get('fare').nil?
    "
  }
  // embarked null값 최빈값(S)로 대체
  ruby {
    code => "
      mode_embarked = 'S'
      event.set('embarked', mode_embarked) if event.get('embarked').nil?
    "
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

# 🔔 가설 설정

데이터에서 의미 있는 내용을 도출해내기 위해 가설을 설정해보고, 검증하는 과정을 거치면서 인사이트를 얻고자 하였습니다.

## 1️⃣ 가설 1:  `동승 가족 수가 적을 수록 생존률이 높을 것이다.`

### 변수 설정

동승 가족 수 `family` : `Sibsp` + `Parch` 

- `Sibsp` : 타이타닉 호에 동승한 자매 / 배우자의 수
- `Parch` : 타이타닉 호에 동승한 부모 / 자식의 수

## 📺 가설 검정 시각화

![image](https://github.com/user-attachments/assets/c72446a9-2e74-4a11-8dd9-61327298b6f6)

![image](https://github.com/user-attachments/assets/bb6ac07b-8bd6-4df2-9a8a-4e88729c4c0f)

![image](https://github.com/user-attachments/assets/0c680af1-8e43-4463-a37f-f64c3dba31c5)

- 동승 가족수가 1~3명일 경우 생존률이 높다는 것을 확인하였습니다.
    - Sibsp, Parch에 대한 생존률을 따로 파악했을 때도 비슷한 경향을 보이는 것을 확인했습니다.
- 동승 가족 수가 가장 적은 `0명`일 경우는 `1~3명`인 경우보다 생존률이 낮은 이유를 알아보기 위해 데이터를 시각화하여 파악하였습니다.

## 원인 파악

동승 가족 수가 0명인 경우와 1~3명인 경우를 비교할 수 있도록 확인하였습니다.

### 연령대 비교

![image](https://github.com/user-attachments/assets/a27692a9-ab77-45ce-bf84-340f35c881b2)

![image](https://github.com/user-attachments/assets/ef0ee2af-88ca-4a77-8e06-86521563475e)

`family` 가 0인 경우는 20세 ~ 40세 연령 분포가 가장 많았고,

`familiy` 가 1~3인 경우는 0인 경우에 비해 0세 ~ 10세의 아동 인구와 60세 이상의 노령 인구 비율이 비교적 높았습니다.

### 성별 비교

![image](https://github.com/user-attachments/assets/95ce03d8-db46-4374-9380-fcdcfb964653)

![image](https://github.com/user-attachments/assets/39fc263a-8b1e-40b0-8d06-28607f4af602)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/599ab8a6-f1c0-476f-9872-378aa7a86093/af80ad2b-3c8b-455e-9c12-6e42e9f7392a/Untitled.png)

![Untitled](https://prod-files-secure.s3.us-west-2.amazonaws.com/599ab8a6-f1c0-476f-9872-378aa7a86093/eee474d6-8201-439e-8d79-54feb4e8c652/Untitled.png)

- `family` 가 0인 경우는 남성 596명, 여성 194명으로 남성 비율이 훨씬 높았습니다.
- `family` 가 1~3인 경우는 남성 192명, 여성 202명으로 남녀 비율이 비슷하다는 것을 알았습니다.

### 좌석등급 비교

![image](https://github.com/user-attachments/assets/62786597-4160-4700-818b-6279a9b68818)

![image](https://github.com/user-attachments/assets/5f9278b9-08d9-4538-9230-e5ffd7903bd4)

- `family` 가 1~3인 경우 `family` 가 0인 경우에 비해 1등급 좌석의 비율이 상대적으로 높았습니다.

### 전체 생존자 기준

1. **연령대 및 성별**

![image](https://github.com/user-attachments/assets/7d68ef78-175e-4e70-b9f5-ba96e23113a5)

전체 생존자 중 0~5세 생존자의 비율이 특히 높았습니다.

또한, 전체 생존자 중 여성의 비율이 높은 것으로 파악되었습니다.

1. **좌석 등급**

![image](https://github.com/user-attachments/assets/33f5f103-3e32-4adc-9261-35b0ceca009d)

전체 생존자 중 1등급 좌석의 비율이 높았습니다.

## 💡 결론

동승 가족 수가 0명이었을 때보다 1~3명이었을 때 생존률이 높은 이유를 다음과 같이 파악하였습니다.

1. 여성, 아동 인구의 생존 비율이 다른 인구보다 비교적 높음
    - 동승 가족 수가 1~3명인 경우 여성, 아동 인구가 많이 포함되어 있음.
2. 동승 가족 수가 1~3명인 경우 1등급 좌석의 비율이 높음

---

## 2️⃣ 가설 2: **`부유한 지역에서 탑승한 승객들의 생존률이 가장 높을 것이다.`**

### 배경

1912년 타이타닉호의 침몰 사건은 역사상 가장 비극적인 해양 재난 중 하나로 기록되어 있습니다. 타이타닉호는 영국 사우샘프턴, 프랑스 셰르부르, 아일랜드 퀸즈타운에서 승객을 태우고 항해를 시작했습니다. 셰르부르는 당시 유럽에서 부유한 인사들이 많이 거주하던 지역으로, 타이타닉호에 탑승한 승객들 중 상당수가 이 도시에서 탑승한 것으로 알려져 있습니다.

### 시각화 분석

1. **탑승지에 따른 승객의 수**

![1](https://github.com/user-attachments/assets/ee7b53b4-afe7-4ec2-ba16-61e921784136)

!https://prod-files-secure.s3.us-west-2.amazonaws.com/599ab8a6-f1c0-476f-9872-378aa7a86093/49991391-573f-47b0-b59c-a6ff3aec076e/Untitled.png

- **사우샘프턴**에서 탑승한 승객 수가 가장 많으며, 다음으로 **셰르부르**와 **퀸즈타운** 순입니다.
- 구체적인 탑승객 수는 사우샘프턴이 782명, 셰르부르가 212명, 퀸즈타운이 50명입니다.
1. **탑승지별 객실 등급**

![2](https://github.com/user-attachments/assets/e0e914ab-babc-48a6-99d8-400cd9b43948)

!https://prod-files-secure.s3.us-west-2.amazonaws.com/599ab8a6-f1c0-476f-9872-378aa7a86093/4f363b87-231f-4718-953b-b7f32a82541f/Untitled.png

- 전체 탑승객 중 1등석 승객 비율은 **셰르부르**에서 가장 높습니다.
1. **탑승지별 생존률**

![3](https://github.com/user-attachments/assets/867d27f2-fbca-43eb-aaec-077f31efa3c5)

!https://prod-files-secure.s3.us-west-2.amazonaws.com/599ab8a6-f1c0-476f-9872-378aa7a86093/fd7fa96e-bdd8-4eca-a801-54c0ab81e79f/Untitled.png

- 1등석 비율이 가장 높은 셰르부르에서 탑승한 승객들의 생존률이 가장 높습니다.
1. **탑승지별 생존률 & 객실 등급별 생존률**

![4](https://github.com/user-attachments/assets/aa7089dc-3e72-45db-ad51-b3f7a5868ed1)

!https://prod-files-secure.s3.us-west-2.amazonaws.com/599ab8a6-f1c0-476f-9872-378aa7a86093/c4bc358e-9412-41c4-8d2a-272ba204e781/Untitled.png

- 탑승지별 생존률과 객실 등급별 생존률은 비슷한 양상을 보입니다. 탑승지는 왼쪽부터 셰르부르(C) → 퀸즈타운(Q) → 사우샘프턴(S) 순입니다.
1. **승객 요금별 생존률**

![5](https://github.com/user-attachments/assets/099ed41b-7c40-46b4-a225-099c3202e7e6)

!https://prod-files-secure.s3.us-west-2.amazonaws.com/599ab8a6-f1c0-476f-9872-378aa7a86093/7cd8b0a2-a9cf-4dc4-9bbf-053c7f2cb803/Untitled.png

- 승객 요금이 높을수록 생존률이 증가하는 경향이 있습니다.
- 이는 고급 객실(1등석) 승객이 상대적으로 더 높은 생존율을 보이는 것과 일치합니다.

## 🔥 결과

1등석의 비율에 따라 [셰르부르(C) → 사우샘프턴(S) → 퀸즈타운(Q)]순으로 생존률이 높을 것이라 예상했으나 실제로는 [셰르부르(C) → 퀸즈타운(Q) → 사우샘프턴(S)] 순으로 생존률이 높았습니다.

## ❓의문점

왜 사우샘프턴(S)보다 퀸즈타운(Q)이 더 많이 생존 했을까 알아보기 위해 **성별, 나이 , 동승자(자매/배우자, 부모/자식)에 따른 생존 비율**을 시각화하였습니다.

## 📺 시각화

**자매/배우자별 생존율**

![image](https://github.com/user-attachments/assets/22939ca9-ad4c-4055-8881-0b8df2f12bcd)

**부모/자식별 생존율**

![image](https://github.com/user-attachments/assets/80fb3b41-47e1-4df4-8a7b-c826f83f73bd)

https://github.com/user-attachments/assets/f44b451d-8f63-4391-8391-3c1d8645ea60

**성별에 따른 생존율**

![image](https://github.com/user-attachments/assets/f0b664a0-a7d5-4b41-a518-00ef7c8ab600)

## 가설 2

그래프를 보아 성별에 따른 생존률이 매우큰 폭으로 차이가 났다 → 여성 비율에 따라 생존자의 생존률이 높을 것이다. 

![image](https://github.com/user-attachments/assets/3114eedc-24af-487e-a254-c446eee025cf)

## 💡 결론

1. 가설1 분석 결과
    
    부유한 승객일수록 생존률이 높았으며, 특히 1등석 승객의 비율이 가장 높은 셰르부르에서 탑승한 승객들의 생존률이 가장 높았습니다. 
    
    이는 당시 셰르부르가 부유한 도시 중 하나였다는 점과 일치하며, 부유한 승객들의 생존 가능성이 높았다는 가설을 뒷받침합니다.
    
    셰르부르에서 탑승한 승객들의 높은 생존률은 객실 등급과 승객 요금이 생존률에 미치는 영향을 명확히 보여줍니다.
    
2. 가설 2의 여성 생존 비율이 높았던 경우는  그 당시 영국에는 ‘어린이들과 여성들의 목숨을 우선적으로 구해야 한다’라는 인식이 있었기에 남성들이 여성들을 지키기 위해 희생했다는 사실을 알 수 있습니다.
    
    결과적으로 타이타닉호 침몰 당시 생존 가능성에 있어 사회 경제적 지위와 약자에 대한 인식이 중요한 요인이었음을 시사합니다.
    

# 🎬 Dashboard

### 주소

http://192.168.0.230:5601/goto/93a802b84d1e35023c9a2a48d6166751
