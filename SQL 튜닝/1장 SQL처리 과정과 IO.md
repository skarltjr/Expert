## 들어가기 앞서.. sql 파싱과 최적화
```
sql은 선언적 질의 언어다.
ex) 우리는 "이거이거 해줘"라고 선언할뿐

그리고 이러한 선언으로부터 결과물을 만들어내는 과정은 절차적이다.
즉 우리는 선언할뿐이고 dbms 내부 엔진인 옵티마이저가 
선언으로부터 결과물을 만들어낼 수 있는 프로그래밍을 대신해준다.

이렇게 dbms 내부에서 프로시저를 작성하고 컴파일해서 실행 가능한 상태로 만드는 전 
과정을 sql 최적화라고한다.
```
### SQL 최적화 과정
```
sql을 실행하기 전 최적화 과정은 다음과 같다.
(우리의 선언으로부터 결과물을 뽑아내는 과정)
```
- 1. sql 파싱
```
먼저 sql parser가 파싱을 진행한다

- 파싱 트리 생성 : sql문을 이루는 개별 구성요소를 분석해서 파싱 트리 생성
- syntax 체크 : 문법저 오류를 검증
- semantic 체크 : 의미상 오류가 없는지 확인 / ex) 없는 테이블 검색 시 에러
```
- 2. sql 최적화
```
이제부터 옵티마이저가 선언 내용으로부터 결과물을 뽑아낼 수 있는 프로시저를 작성하고 컴파일해서
실행 가능한 상태로 만든다.

이 단계에서 옵티마이저는 미리 수집한 시스템 및 오브젝트 통계정보를 바탕으로 
다양한 실행경로를 생성해서 비교한 후 가장 효율적인 경로를 채택한다.
즉, 성능을 결정하는 핵심
```
- 3. 로우 소스 생성
```
옵티마이저가 선택한 실행 경로를 실제 실행 가능한 코드 또는 프로시저로 만들어낸다.
```

### SQL 옵티마이저
```
옵티마이저는 결국 사용자가 원하는 작업을 "가장 효율적으로" 처리할 수 있는 최적의
데이터 액세스 경로를 선택해주는 dbms의 핵심 엔진이다.

옵티마이저 최적화 단계는 아래와 같은데..
```
- 먼저 사용자로부터 전달받은 쿼리를 수행하는데 후보군이 될만하 실행계획들을 찾는다
- 데이터 딕셔너리에 미리 수집해 둔 오브젝트 통계 및 시스템 통계정보를 이용해 각 실행계획의 예상비용 산정
- 최저 비용(최고 효율) 실행계획 선택

### 실행계획과 비용
```
우리가 네비게이션을 사용할때 최적 경로를 보여준다
옵티마이저 또한 일종의 네비게이션이라고 생각하자
최적의 경로를 제공해주는...
```
- 여기서는 mysql을 사용하며 explain 키워드를 통해 mysql 실행계획 미리보기가 가능하다.
- <img width="779" alt="스크린샷 2023-01-02 오후 10 17 29" src="https://user-images.githubusercontent.com/62214428/210236603-6cc21d49-53d8-43d2-9f8c-bcebb2c1f37c.png">
- <img width="1478" alt="스크린샷 2023-01-02 오후 10 17 16" src="https://user-images.githubusercontent.com/62214428/210236613-a14d6a62-8b22-4c41-96f7-46ee07c2aeef.png">
```
sales left join car
- 결국 sales는 type:all로 모든 컬럼 조회
- car의 경우 pk로 조인하며 type:eq_ref로 
   - 조인수행을 위해 각 테이블에서 하나의 행만이 읽혀지는 형태. const 타입 외에 가장 훌륭한 조인타입이다.
```
- 그런데 결과 산출을 위해 접근되는 rows가 1인게 아직 이해가 안된다....


- 참고 : 
```
pk는 기본적으로 인덱스가 걸린다
&
https://jeong-pro.tistory.com/243
```

### 옵티마이저 힌트
```
네비게이션이 항상 옳은길만 찾아줄까?
옵티마이저 또한 마찬가지다.

이럴때 옵티마이저 힌트를 통해 의도적으로 데이터 액세스 경로를 변경할 수 있다.

나아가 인덱스 a를 사용할게, 조인 방식이나 순서는 옵티마이저 너가 알아서해줘와 같이 
일부만 지시할 수 있는데 이는 잘못된 결과를 만들어낼 수 있다.

⭐️ 그러니 이왕에 힌트를 사용할거면 빈틈없이 기술하라고 한다.
```
```
1번 = explain select * from car where carId = 3;

create index c_maker on car(maker);

show index from car;

2번 = explain select /*+ INDEX(car c_maker) */ * from car where carId = 3; // 옵티마이저 힌트 적용하여 c_maker 인덱스 활용하도록
```
```
실제로 
1,2번의 rows 결과가 다르다 1번은 1/ 2번은 2
즉 불필요한 인덱스 사용으로 악영향도 발생한다.
```

## SQL 공유 및 재사용
- <img width="646" alt="스크린샷 2023-01-03 오후 10 27 36" src="https://user-images.githubusercontent.com/62214428/210366493-801f9d31-4f68-4d85-a541-16d784ed5c0d.png">
```
먼저 sql 파싱, 최적화, 로우 소스 생성 과정을 거쳐 생성한
내부 프로시저를 반복 재사용할 수 있도로 캐싱해두는 메모리 공간을 라이브러리 캐시라고 한다

다시 말해, 
우리가 사용한 sql 자체ex. select * from A)를 재사용할 수 있게 캐싱해두는 공간은 라이브러리 캐시
우리가 얻은 데이터(블럭)을 재사용할 수 있게 캐싱해두는 공간은 db buffer cache
```

### 소프트 파싱 vs 하드 파싱
- 기본적으로 사용자가 sql을 작성하여 전달하면 dbms는 라이브러리 캐시에 접근한다
- 그리고 이때 만약 캐시 hit라면 소프트 파싱
- cache miss로 sql 최적화를 진행해야 한다면 하드 파싱이다.
- <img width="603" alt="스크린샷 2023-01-03 오후 10 33 01" src="https://user-images.githubusercontent.com/62214428/210367210-2560b40b-574c-4e44-98cc-01cb9ead5f3f.png">

```
참고로 하드 파싱.
즉 sql 최적화 과정은 dbms에서 몇안되는 cpu 집약적인 행위
그러니 만약 라이브러리 캐시가 없다면 엄청난 트래픽이 몰렸을때
db cpu 사용량이 터져버릴것... 그래서 라이브러리 캐시가 필요한데...

아직 문제가 하나 있다.
```

### 바인드 변수의 중요성
```
사용자 정의 함수, 프로시저등은 이름이 있다.
그러나.. sql은 이름이 없다.
즉 sql 자체가 하나의 이름이다..

이게 무슨 말인가?
select  * from car;
select * FROM car;
SELECT * from car;

위 3개 sql은 모두 서로 다른 쿼리다.
```
- 이 말은 서로 다른 sql을 모두 저장하거나
- 모두 각각 캐싱해두면 오히려 그 양이 방대하여 역효과다.
- 그래서 오라클이 sql을 영구 저장하지 않는쪽을 선택

### 공유 가능 SQL
```
그럼 어떡해...?

같은 sql같지만 이름이 없기에 서로 다른 sql이라 문제라고 했다.
그래서 필요한 방법이 바인드 변수를 활용하는것

즉 하나의 프로시저에서 파라미터만 달리하여 재사용한다.
```
```
만약 아래와 같이 id만 다른 sql이 존재한다고 해보자

select * from car where id=1;
select * from car where id=2;
select * from car where id=3;

결국 모두 다른 sql이다.
그런데 사실은 기능 자체는 동일하되 id만 서로 다르 값이니
하나의 프로시저로 id(바인디 변수)만 달리한다면 
sql은 하나만 저장하면 되지 않는가?
```
- 그럴때 사용되는것이 `바인드 변수`
```
select * from car where id = ?;
```
### 자바 진영의 preparedStatement
```
java 진영에서는 쿼리문을 사용할때 java.sql 패키지의 statement를 사용한다고 한다.
```
- <img width="793" alt="스크린샷 2023-01-03 오후 11 30 03" src="https://user-images.githubusercontent.com/62214428/210377294-852d136f-2e5b-4d0b-8726-d8b637910189.png">
```
preparedStatement는 statement를 상속받은 인터페이스로
위 그림에서 parsing 결과를 캐싱해두고 나머지 3단계만 거쳐 sql로 변환해준다.
```
```
우리가 jpql을 사용하며 select * from car where id = ?;
위와 같은 sql에서 파라미터만 바꾸는 파라미터 바인딩 또한 preparedStatement를 사용한다
```
```
결론적으로 공유 가능한 sql을 만들고
라이브러리 캐시에 이를 캐싱해두고 재사용함으로써
하드 파싱을 줄여 cpu 사용량을 줄이고 재사용한다.
```
- <img width="1493" alt="스크린샷 2023-01-04 오전 12 22 17" src="https://user-images.githubusercontent.com/62214428/210387159-7c5c2333-1a9f-4fed-94a8-df1d811e0983.png">
```
위 사진은 java statement를 상속한 preparedStatement를 impl한 JdbcPreparedStatement 생성자에
브레이크 포인트를 걸고 왼쪽 쿼리를 실행했을때 걸리는지 확인한것이다.

만들어진 sql은 다음과 같다.
select product0_.product_id as product_1_0_, product0_.stock as stock2_0_, product0_.version as version3_0_ from products product0_ where product0_.product_id=?

jpql에서 파라미터 바인딩또한 preparedStatement를 활용한다고 볼 수 있다.
```
