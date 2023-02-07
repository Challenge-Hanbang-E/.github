<img width="170" alt="스크린샷 2023-02-07 오후 4 06 41" src="https://user-images.githubusercontent.com/85235063/217172283-63f84d4e-856b-4a2e-a6b6-7217f9b60d6a.png">

# 한방에 검색하고 한방에 주문하는 제 2의 쿠팡 
**Hanbang-e 프로젝트 자세히 보기** 👉 [노션 바로가기](https://extreme-wall-e0e.notion.site/hanbang-e-406c7cb8500b4361b68994a61e7d7ae7)

<br/>

## 기획

<br/>

## 아키텍처 
<img width="851" alt="스크린샷 2023-02-07 오후 4 22 18" src="https://user-images.githubusercontent.com/85235063/217175760-32186efd-ecb7-424e-8aaa-545408d149c3.png">


<br/>

## 핵심 기능
### 💾 데이터 파이프라인 
- 데이터 크롤링부터 logstash활용 elasticsearch 데이터 적재 과정 자동화 
<img width="1256" alt="스크린샷 2023-02-07 오후 5 41 13" src="https://user-images.githubusercontent.com/85235063/217194782-b5d0e5f2-6ef5-47b2-8796-ca889c5310d9.png">

### 🔎 상품 검색
- `Elasticsearch`을 통한 1초 이내의 검색 응답(목표로 했던 Latency 값 달성)
- `판매량 순`, `가격 낮은순`, `가격 높은순` 정렬 제공

### 📦 상품 주문 
- 대규모 트래픽에서 상품 주문 동시성 이슈 발생 
- Redis + Redisson활용 분산락 구현 

### 🗂️ 재입고 데이터 일괄 처리 
- 스프링 배치 활용 재고량 업데이트 

<br/>

## 트러블 슈팅 및 성능 개선

<details>
<summary>비효율적인 데이터 수집 및 저장 로직 </summary>
<div markdown="1">

<br/>

**[문제상황]** 

데이터 수집시 크롤링과 동시에 RDS와 바로 통신하여 데이터 하나씩 저장하는 로직. 
RDS로 바로 바로 전송하므로, 통신 횟수는 백 만건 이상이며 시간 또한 오래 걸림. 
또한 DB에 데이터가 날아갔을 때 다시 (엄청난 시간을 들여) 크롤링을 해줘야 하는 문제 발생

**[해결방안]**

크롤링한 데이터를 Json파일에 적재 후 파일 당 Bulk Insert문을 활용하여 한번에 RDS에 저장


**[결과]**
- 크롤링한 데이터를 파일로 따로 모아두어 최악에 상황을 대비
- Bulk Insert문을 활용하여 RDS 통신 횟수 및 저장 시간 단축 (약 1만 3천개의 데이터 적재 시간: 약 4s로 줄어듬)
<br/>

</details>


<details>
<summary>검색 성능 개선</summary>
<div markdown="1">

- 성능 개선 기준: API Latency
- 성능 테스트 방법: Artillery활용 성능 테스트 진행 → `유저 1명의 단일 요청 100번 결과(response time metrics (p50, p95, p99, min and max))` [@검색 성능 테스트](https://extreme-wall-e0e.notion.site/492d10e38910435197b836964544b793)

- #### 성능 개선 과정 

  <details>
  <summary>1. 상품명(Product 테이블 name 컬럼) index</summary>
  <div markdown="1">
  <br/>
  
  - **검색 조건 `like '%word%'`는 인덱스를 타지 않아 속도 개선 x** <br/>
  <br/>
  
  > 더욱 자세한 내용은 아래 링크를 참조
  - [@검색 성능 개선 - column index, index 구조](https://extreme-wall-e0e.notion.site/column-index-792619f15ed3463c9dcfb4d1714fdfc5)
  - [@검색 성능 개선 - column index2, 실행계획](https://extreme-wall-e0e.notion.site/2-88d78b9337c64b89b2fb8833e8bd2aed)
  <br/>
  
  </details>
  
  <details>
  <summary>2. full-text index</summary>
  <div markdown="1">
  
  - `ngram parser` 사용 
  - search type은 `불린 모드`,구문 검색은 `+`를 사용하여 사용자가 검색한 단어가 필수적으로 들어간 상품명이 조회되도록 쿼리 구성
 
  -  대략 대부분의 키워드 응답 최대시간이 2초내로 향상되었지만, 특정 키워드에서는 응답 속도가 확연히 늦어짐을 발견 
  -  응답 최대 시간 목표 1초를 달성하지 못하여 elasticsearch 도입
  <br/>
  
  > 더욱 자세한 내용은 아래 링크를 참조
  - [@검색 성능 개선 - full text index](https://extreme-wall-e0e.notion.site/full-text-index-2-52e0ec895d2749d599078255e1b5b3c3) 
  
  <br/>
  
   </details>
   
  <details>
  <summary>3. elasticsearch</summary>
  <br/>
  
  - 목표로 했던 API Latency인 1초 이내 달성 🎉 
  - elasticsearch는 역인덱스 활용하기 때문에 검색속도가 빠름 
  <br/>
  
  > 더욱 자세한 내용은 아래 링크를 참조
  - [@검색 성능 개선 - elasticsearch](https://extreme-wall-e0e.notion.site/Elasticsearch-a40d611cd7b34f88b4774a7a42a083cc)
  
  <br/>
  
  </details>

  **테스트 결과 기록* 👉 
  [테스트 결과 기록_한방에](https://docs.google.com/spreadsheets/d/1sqh296eSPjw4vkJcYQfjMg1eFdqQjREh-nOYRqOEW3s/edit?usp=sharing) <br/>
  **테스트 결과 총 정리* 👉 [Index 활용 검색 성능 최종 정리본](https://extreme-wall-e0e.notion.site/Index-e763367a15f140749785fcd84ec008ea)
</details>

<details>
<summary>동시성 제어</summary>
<div markdown="1">

#### - 동시성 제어 방식 비교
1. Redis와 Redisson클라이언트를 이용한 분산락 구현
2. DBMS(Mysql)의 락 기능을 이용한 분산라 구현

- 성능 비교
  - 테스트 환경 : JMeter를 이용하여 동시성문제가 일어나는 동일한 조건으로 테스트를 실행
  - 테스트 조건 : Number Of Threads = 1000, Ramp-up period = 1, Loop Count = 1의 조건으로 총 Sample수는 각각 10000개
  
      ||Throughput|Min|Max|Average|
      |---|---|---|---|---|
      |Redis|6.8/sec|122ms|75098ms|10579ms|
      |DBMS|6.4/sec|99ms|19625ms|9483ms|

- 결론
  - SQL DB Lock
    - **별도의 Redis 인스턴스를 운용하고 있지 않을 경우 인프라의 부담 없이 활용해볼 수 있다는 장점**
    - **SQL connection에 부담이 많이 가며 Redis에 비해서는 무겁다**는 단점이 있음
  - **Redis**
    - **Redisson Pub/Sub를 이용하여 Lock 획득 재시도를 최소화하여** Redis에 가해지는 부담을 최소화
    - **별도의 트랜잭션 작업 처리와 새로운 DB임으로 높은 추가비용이 발생**
  
  #### ❗️ JMeter를 이용한 성능 테스트 결과 유의미한 성능의 차이점은 없었음. 하지만 현재 상품 상세정보 캐싱을 위해서 이미 Redis서버가 존재하고 있었기 때문에 **Redis를 이용한 분산락을 활용하기로 결정**
  
  <br/>
  
  > 더욱 자세한 내용은 아래 링크를 참조
  - [@동시성 제어 정리](https://extreme-wall-e0e.notion.site/b7696e3e4a234fc29ccac3512ab77c79)
  
</details>


<br/>

### 팀원
|이름|Github|
|------|------|
|김예진|https://github.com/yejin0718|
|김재현|https://github.com/tjvm0877|
|민승기|https://github.com/seunGit|
|송민아|https://github.com/m1naworld
|이한주|https://github.com/yanJuicy|
