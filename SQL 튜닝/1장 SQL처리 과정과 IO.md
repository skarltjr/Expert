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
- 여기에 대한 대답은 바로. 예상치!!라서.. 
- 데이터 1000개를 넣고 돌려보니 얼추 더 정확히 잘 나온다.
```
show create table car;
select * from car;

# 반복 insert 프로시저
CREATE PROCEDURE loopInsert()
BEGIN
    DECLARE i INT DEFAULT 1;
    WHILE i <= 1000
        DO
            INSERT INTO car(carId, carName, maker)
            VALUES (null,concat('carName',concat(i)),concat('maker',concat(i)));
            SET i = i + 1;
        END WHILE;
END;

show procedure status ; # 프로시저 조회
drop procedure loopInsert; # 프로시저 삭제
call loopInsert(); # 프로시저 호출


explain select * from car where carId>30;
```
- <img width="1651" alt="스크린샷 2023-01-07 오후 11 08 31" src="https://user-images.githubusercontent.com/62214428/211154875-8d9ca3b3-9bd1-4556-a9c8-5b7f4f69c29d.png">

```
참고로 type 필드의 경우 공식 문서에 the join type이라고 나온다
그런데 나는 개인적으로
type은 접근 방식을 표시하는 필드다. 접근 방식은 테이블에서 어떻게 행데이터를 가져올것인가를 가리킨다
라는 말이 더 와닿았다.
아래 그림을 통해 생각해보자
```
- <img width="1260" alt="스크린샷 2023-01-07 오후 3 51 33" src="https://user-images.githubusercontent.com/62214428/211135375-7a65cd06-cf00-4cb3-93f2-f567b2460768.png">

- <img width="1295" alt="스크린샷 2023-01-07 오후 3 51 57" src="https://user-images.githubusercontent.com/62214428/211135399-68904bf6-7bdf-4859-9ace-24cf77a9f212.png">

- <img width="1343" alt="스크린샷 2023-01-07 오후 3 52 23" src="https://user-images.githubusercontent.com/62214428/211135389-efd8bc44-5c35-467e-9333-c23a03d93770.png">



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
```

```
불필요한 인덱스 사용으로 악영향도 발생하는걸 직접확인해보자.
아래 2번째 사진은 의도적으로 이상한 인덱스 만들어서 사용한것
그 결과로 1003 row full scan
```
- <img width="1636" alt="스크린샷 2023-01-07 오후 11 25 10" src="https://user-images.githubusercontent.com/62214428/211155535-9dc63fc3-ac03-4e97-8d39-ff41693ad015.png">
- <img width="1607" alt="스크린샷 2023-01-07 오후 11 26 22" src="https://user-images.githubusercontent.com/62214428/211155581-9033ed37-a35e-4577-ac04-bc3f36739c80.png">



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

## 데이터 저장 구조 및 I/O 매커니즘
```
sql이 느린 이유는 거의 I/O 그 중 디스크 I/O가 문제
이 부분을 개선하는게 핵심 중 하나
```
### sql이 느린 이유
- 디스크 I/O를 조심해라
```
i/o는 마치 잠(sleep)과 같다.
io가 발생하는동안 프로세스는 잠을 자는거와 마찬가지이다.
```
- <img width="703" alt="스크린샷 2023-01-04 오후 10 20 20" src="https://user-images.githubusercontent.com/62214428/210563973-cb547075-cfc3-4187-a58d-4ba16d606f78.png">
```
즉 열심히 일해야할 프로세스가 디스크 io를 위해 멈춰있다.
그리고 수 많은 프로세스가 동시에 i/o call을 요청하면
당연히 디스크 경합문제도 발생하고 결국 느려진다!
```

### 데이터베이스 저장구조
- <img width="714" alt="스크린샷 2023-01-04 오후 10 30 24" src="https://user-images.githubusercontent.com/62214428/210565789-9e00d334-3556-4c55-b36e-7aa23de62471.png">
```
데이터를 저장하기 위해선 먼저 테이블스페이스를 생성해야한다.

테이블스페이스 : 세그먼트를 담는 컨테이너 / 여러개의 데이터파일(디스크상 물리적인 os파일로)구성된다.
세그먼트 : 테이블,인덱스처럼 저장공간이 필요한 오브젝트 / 여러개의 익스텐트로 구성
익스텐트 : 공간을 확장하는 단위 / 블록의 집합
블록 : 사용자가 입력한 레코드를 실질적으로 저장하는 공간이자 데이터를 읽고 쓰는 단위
- 참고로 하나의 블록에 저장된 데이터는 모두 같은 테이블의 데이터
- 추가로 한 익스텐트의 블록들도 모두 같은 테이블의 데이터
```
- <img width="554" alt="스크린샷 2023-01-04 오후 10 37 17" src="https://user-images.githubusercontent.com/62214428/210566975-116dd119-741f-403a-8d7a-951cf3922da3.png">
```
참고로 한 세그먼트의 익스텐트는 보통 서로 다른 데이터파일에 분산 저장된다
파일 경합을 줄이기 위해
```

### 블록 단위 I/O
```
블록은 데이터를 읽고 쓰는 단위다. 보통 8kb
특정 레코드 하나를 읽고 싶어도 블록 단위로 읽고 쓰기 때문에
이때도 블럭 전체를 읽는다.

참고로 인덱스또한 블럭단위로 읽고 쓴다.
```

### 시퀀셜 액세스 vs 랜덤 액세스
```
테이블 또는 인덱스 블럭을 읽는 방법으로는 두 방법이 존재한다.
```
- <img width="739" alt="스크린샷 2023-01-04 오후 10 52 10" src="https://user-images.githubusercontent.com/62214428/210569748-ecf8935d-dfc0-4159-b056-f3cb4e282039.png">
- 시퀀셜 액세스
```
논리적 또는 물리적으로 연결된 순서에 따라 차례대로 블록을 읽는 방식이다.
인덱스 리프 블록의 경우 앞뒤를 가리키는 주소값을 통해 논리적으로 연결되어있고 순차적으로 읽을 수 있다.
```
```
그런데 테이블 블록의 경우 연결고리가 없다.

그러나 오라클의 경우 세그먼트에 할당된 익스텐트 목록을 세그먼트 헤더에 맵으로 관리
따라서 익스텐트 맵은 각 익스텐트 첫 번째 블록 주소값을 갖는다

그래서 1번 익스텐트 첫번째 블록부터 쭉~읽고
그 다음 2번 익스텐트 첫번째 블록부터 쭉~ 읽으면 그게 테이블 시퀀셔 액세스다
```
- 랜덤 액세스
```
레코드 하나를 읽기 위해 한 블록씩 접근하는 방법

즉!!! 원하는 레코드가 존재하는곳의 블럭만 읽는다
다시 생각해보면 인덱스는 랜덤 액세스를 사용하여 
원하는 레코드만 빠르게 읽어낼 수 있다.

반대로 엄청 많은 인덱스로 랜덤 액세스하면 하나씩 다 찾으니까
full scan보다 느릴 수 있다.
```

### 논리적 I/O vs 물리적 I/O
```
앞서 라이브러리 캐시는 sql, 프로시저와 같은 코드르 캐시해두는 코드 캐시
버퍼 캐시의 경우 실질적인 데이터를 캐시하여 disk io를 줄이기 위한 영역
```

```
이때
논리적 i/o란 sql을 처리하는 과저에서 발생한 총 블록 i/o를 말한다
일반적으로 메모리(버퍼 캐시) 접근이 일어난 블록 i/o를 말한다.
물론 메모리를 경유하지 않는 direct path io를 포함하기도한다.
```

```
반면 물리적 i/o는 디스크에서 발생한 총 블록 io를 말한다.

당연한 이야기지만 버퍼캐시(메모리) miss가 발생하는 경우에 물리적 io가 발생한다
```
- ⭐️그래서 착각하기 쉬운것은
- 총 io가 논리적 + 물리적 io의 합이라고 생각하는것
- 그러나 논리적 io중 일부 miss가 물리적 io를 진행하는것

```
논리적 io 횟수와 물리적 io 횟수

아주 간단하게 우리가 3개 데이터를 찾는다고 하자
논리적 io 횟수는 항상 같다. 왜?

3개를 찾기 위해 매번 3번 메모리에 접근한다고 생각하면된다.
반면 매 순간 버퍼 캐시에 3개가 다 캐시 되어 있을수도있고
1개,2개만 되어있을수도 있다. 
그러니 물리적 io는 횟수가 매번 다르다.
```

### 버퍼 캐시 히트율
- 공식은 간단하다
- 전체는 논리적 io고 그 중 일부가 물리적 io로 전환된다고 했다
```
그러니까 버퍼 캐시 히트율은

1 - (물리 io / 논리 io)에 100을 곱해주면 된다.

ex) 전체 10건중 3건이 디스크 io로 전환되었다고 해보자
1 - (3/10)에 100을 곱하면 70퍼다.
```
- 그런데 이 책에서 말하길 온라인 트랜잭션을 처리하는 애플리케이션이라면
- 99%를 달성해야한다고한다.

```
사실 중요한것은
그럼 결국에 우리가 줄이고자하는건 디스크 io였다.
그 말은 즉슨 물리적 io를 줄여야한다는것이다.

그런데!!! 논리적 io중 일부가 물리적 io로 전환된다고 했다.
그리고 우리가 버퍼 캐시에 어떤게 캐시되어있을지 항상 제어할 수 있는가?

즉!! 우리가 제어할 수 있는건 논리적 io를 줄이고 이로 하여금 성능을 증진시키는것이다.
```

### single block io VS multi block io
- 이는 블럭을 하나씩 혹은 여러개씩 디스크에서 메모리로 가져오는 방법의 차이다.
- 헷갈리면 안되는게 버퍼 캐시(메모리)에 miss나서 디스크 io로 전환되는 애들만 대상이다. 당연히
```
당연하게도 single block io의 경우
한 블럭씩 옮기니까 빠르다.

인덱스를 사용하는 경우 인덱스와 테이블 모두 single block io를 사용한다.
```
```
반대로 당연하게도..
테이블 full scan일 경우 multi block io가 효과적이다.
큰 수레에 벽돌을 다 담은후에 움직인다.

그 말은... !!!
프로세스가 비교적 적은 횟수만 멈추면 된다는말이기도하다.

multi block의 크기는 크면 클 수록 좋다..
하지만 당연히 그 상한선은 상황에 맞게 가져가야한다.
```

### 캐시 탐색 메커니즘
```
dbms는 버퍼캐시를 해시구조로 관리한다.
```
- <img width="735" alt="스크린샷 2023-01-07 오후 5 14 19" src="https://user-images.githubusercontent.com/62214428/211140940-0706325d-9fdd-48d8-90db-69251334d3b9.png">









