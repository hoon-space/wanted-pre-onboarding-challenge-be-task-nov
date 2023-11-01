# :: 원티드 프리온보딩 챌린지 백엔드 코스 사전과제 안내
## 1-1) 사전과제 진행 가이드
- 아래 총 5 문제에 대한 해설을 해당 레포지토리를 fork 한 후 각 문제에 해당하는 파일에 작성해주세요.
- 문제 해설에 대한 정해진 양식은 없으며, 최대한 자세히 해설해주시면 더욱 좋습니다.
- 문제 유형은 해당 코스와 무관하게 면접을 준비한다면 알고 있으면 좋은 주제들을 몇 가지 선정하였습니다.

<br>

## 1-2) 사전 과제 
## (1) 동시에 같은 DB를 업데이트 하는 상황을 방어 전략

한 쪽에서 DB에서 값을 가져오려 할 때, 본인이 기대한 값이 아닌 값이 읽히는 현상을 Phantom Read라고 한다. 

이는 값을 읽어오는 사이, 다른 트랜잭션에 따라 값이 수정되거나 추가됨에 따라 나타나는 현상이다.

[Dirty read, Non-repeatable read, and Phantom read](https://jennyttt.medium.com/dirty-read-non-repeatable-read-and-phantom-read-bd75dd69d03a)

### 해결방안1) Message Queue 방식 도입

- Kafka와 같은 메시지 큐 사용
- front 단에서 message queue를 사용하여 request를 큐로 정렬

### 해결방안2) Row Lock 도입

- MySQL

트랜잭션 시작할 때, `SELECT ... FOR UPDATE`을 통해 수정될 수 있는 row들에 대한 필수적인 lock을 얻는다.

만약 transaction들이 하나 이상의 테이블을 수정하거나 락을 걸게되면, 순서에 따른 각 트랜잭션들의 명령문들을 실행한다.

InnoDB는 deadlock detection을 지원하고, 인지되면 관련된 트랜잭션들을 롤백한다.

이 때, 동시성이 높은 시스템에 대해서는 굉장한 성능이슈가 발생하게 된다.

이러한 경우,[innodb_deadlock_detect](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_deadlock_detect)설정 옵션을 통해 deadlock detection을 풀고, [innodb_lock_wait_timeout](https://dev.mysql.com/doc/refman/8.0/en/innodb-parameters.html#sysvar_innodb_lock_wait_timeout)설정을 통해 트랜잭션 롤백을 지원하라고 공식문서에서 권고하고 있다.

[MySQL :: MySQL 8.0 Reference Manual :: 8.11.1 Internal Locking Methods](https://dev.mysql.com/doc/refman/8.0/en/internal-locking.html)

### 해결방안3) Table Lock 도입

일반적으로 아래와 같은 경우, 테이블 락이 row 락에 비해 더욱 우수하다.

- 해당 테이블에 대한 대부분의 명령문이 읽기 일 때
- 하나의 키 값으로 delete와 update가 될 때
- concurrent한 insert문과 select문이 섞여있을 때
- write하는 트랜잭션은 없고, 단지 GROUP BY나 scan과 같은 작업이 많을 때

row lock에 대한 오버헤드가 높으므로, table lock을 상황에 맞추어 사용하는 것이 좋다.

[MySQL :: MySQL 8.0 Reference Manual :: 8.11.1 Internal Locking Methods](https://dev.mysql.com/doc/refman/8.0/en/internal-locking.html)

### 해결방안4) 격리 수준을 높인다

MySQL의 InnoDB엔진은 기본적으로 REPEATABLE_READ 수준으로 제공한다.

XA 트랜잭션 — 여러 DB를 사용하는 분산 트랜잭션을 처리 — 를 사용하거나, 동시성 이슈에 대한 트러블 슈팅을 해야하는 특수한 상황에서는 [SERIALIZABLE](https://dev.mysql.com/doc/refman/8.0/en/innodb-transaction-isolation-levels.html#isolevel_serializable)을 사용할 수 있다.

(그렇게 강력하게 권고하는 글은 찾아보기 힘들다)

## 비관적인 방법 vs 낙관적인 방법의 해결방안

### 비관적 락

> 현재 수정하려는 데이터가 언제든 다른 요청에 의해 수정될 가능성을 고려하여 해당 데이터에 Lock을 거는 방식
> 

방법은 세 가지이다.

- DB에서 제공하는 데이터 Lock 수준을 높이는 것
    - 현실적으로 힘듬
- select for update 활용하여 명시적으로 lock 잡기
    - lock 구간이 길어져 성능 저하
- transaction의 write lock 활용
    - 데이터를 수정할 때 write lock이 걸리고 transaction이 끝나야 lock이 풀림
    - DB와 서버성능에 따라 서비스 속도가 좌우됨

### 낙관적 락

> 수정하려는 데이터는 나만 수정할 것이라는 낙관적인 생각의 방법
> 
> 
> 테이블에 version이라는 숫자컬럼 또는 updated_at 이라는 시간컬럼을 만들어서 수정될 때마다 1씩 증가하거나, 현재시간으로 갱신
> 
> 값을 수정할 때 Versino이 동일하면 수정이 가능해지고, 동일하지 않으면 수정에 실패
> 

- 모델에 컬럼을 하나 추가하면 구현이 비교적 쉽다.
- 그러나, 두개의 DB세션이 동일한 버전으로 수정하려고 하면 한 개의 세션에선 version conflict이 발생하여 affected row count가 0이 된다.
- 따라서 이 경우 요청을 재시도하도록 구현이 필요

[동시성 문제 해결방법](https://chrisjune-13837.medium.com/db-동시성-문제-해결방법-f5e52e2e3)

## (2) `TCP` 와 `UDP` 의 차이

둘 다 OSI 7 계층에서, 데이터 전달과 패킷 오류검사 및 재전송요구를 처리하는 Transport Layer 프로토콜이다.

### TCP

3 way handshake를 통해 연결을 하고, 4way handshake로 연결을 해제한다.

따라서 신뢰성을 보장하며, 흐름제어와 혼잡제어의 기능을 지원한다.

이 연결은 1대1 통신을 사용한다.

- 흐름제어 :
    - Stop and Wait
        - 각 패킷 별 확인 응답 이후에만 다음 패킷 전송
    - Sliiding Window
        - 송신버퍼(윈도우)와 수신버퍼(윈도우)를 3way handshake를 통해 송신 윈도우 크기를 맞춤
- 혼잡제어
    - ****AIMD****
        - 1씩 증가시켜가다, 실패 시, window size를 절반으로 줄여감
    - Slow Start
        - 1씩 증가시켜사다, 혼잡 현상 발생 시, 1로 떨어뜨림
    - Fast Retransmit
        - 중간에 하나가 손실되게 되면 송신 측에서는 순번이 중복된 ACK 패킷을 받게 되는데, 이것을 감지하는 순간 문제가 되는 순번의 패킷을 재전송
        - window size를 1로 초기화
    - Fast Recovery
        - slow start 방식을 진행하다가, 혼잡 발생 시 — slow start와 달리— 1로 떨어뜨리지 않고 절반으로 줄이고 선형증가시키는 방법

### UDP

비연결형 서비스이므로 연결 설정이 없다.

흐름제어와 혼잡제어 기능을 제공하지 않고, 데이터 수신여부 또한 확인하지 않는다.

또한 전송순서가 바뀔 수 있으므로 신뢰성있는 연결을 지원하지 않는다.

대신 속도가 빠르며 서로 다른 경로로 독립적으로 처리되므로, 연속성 있는 전송이 필요할 때 사용되는 프로토콜이다.

이 연결은 1대1, 1대n, n대n통신을 지원한다.

[TCP (흐름제어/혼잡제어/오류제어)](https://velog.io/@jsj3282/TCP-흐름제어혼잡제어-오류제어)

[TCP 흐름제어 기법 살펴보기](https://velog.io/@haero_kim/TCP-흐름제어-기법-살펴보기)

[https://gyoogle.dev/blog/computer-science/network/흐름제어 & 혼잡제어.html](https://gyoogle.dev/blog/computer-science/network/%ED%9D%90%EB%A6%84%EC%A0%9C%EC%96%B4%20&%20%ED%98%BC%EC%9E%A1%EC%A0%9C%EC%96%B4.html)

## (3) 웹 브라우저에 `네이버` 를 검색하고 화면에 네이버 화면이 출력이 될 때 까지 내부적으로 어떤 동작들이 수행이 되는지 설명해주세요.

1. 브라우저 오픈
2. www.naver.com을 입력
    1. 브라우저가 입력한 URL을 기반으로 프로토콜과 도메인네임, 포트 등을 파싱
3. Domain Name Server에 도메인네임을 검색하기 위한 요청을 보냄
    1. 도메인네임??
        
        [도메인 네임이란?](https://velog.io/@jisoolee11/도메인-네임이란)
        
    2. DNS resolver와 Root Name Server, .com Name Server, naver.com을 관리하는 Name Server와 통신하여 ip주소를 갱신받음
4. DNS는 도메인네임(www.naver.com)에 대응하는 ip 주소를 응답으로 돌려줌
5. ip주소를 사용하여 ip서버에 TCP를 통한 요청
    1. ARP를 통해 ip주소를 mac주소로 변환
    2. 이후 tcp 소켓 통신
6. (프록시 서버나 리버스 프록시 서버가 처리)
7. ip서버는 요청 내용에 대한 일련에 처리 과정을 거쳐 응답 메시지를 형성
8. 형성한 응답메세지를 ip서버가 TCP를 통해 다시 클라이언트에게 전송
9. 브라우저는 받은 응답메시지를 HTTP를 활용하여 웹페이즈를 구성
10. 구성된 웹 페이지를 사용자에게 보여줌

[인터넷 주소창에 URL을 입력 후 화면에 출력되는 과정](https://1-7171771.tistory.com/134)

[[Network] 5. 주소창에 www.naver.com을 쳤을 때 생기는일](https://develaniper-devpage.tistory.com/88)

## (4) 본인이 주력으로 사용하는 언어에서 설계적 결함 한 가지를 작성해주세요.

### **1. Generics의 타입을 Runtime에 활용할 수 없다.**

- 제네릭을 활용해 컴파일 타임에 타입 체크를 하고 나면 제네릭 인자로 넘겨져 온 타입은 Type erasure라는 절차를 통해 제거된다.
- 따라서 인자로 넘겨진 타입은 Runtime에서는 활용 될 수 없다.

```java
ArrayList<Integer> li = new ArrayList<Integer>();
ArrayList<Float> lf = new ArrayList<Float>();
if (li.getClass() == lf.getClass()) { // evaluates to true
    System.out.println("Equal");
}
```

### **2. Unsigned integer types가 없다.**

### **3. Operator overloading을 할 수 없다.**

### **4. 배열 크기가 2^31으로 제한된다.**

### **5. primitives type과 Array가 클래스와 다르게 취급됨.**

### 6. JVM 위에서 동작하므로 메모리나 속도 측면에서 이슈가 있다.

### 7. 포인터와 같은 메모리 관련 low level 프로그래밍이 불가하다.

### 8. GC에 대해서 완전한 제어가 불가능하다.

[Disadvantages of Java Language - GeeksforGeeks](https://www.geeksforgeeks.org/disadvantages-of-java-language/)

[자바 설계 결함](https://velog.io/@xogml951/자바-설계-결함)

## (5) 본인이 주력으로 사용하는 언어에서 자료구조와 관련 된 클래스가 내부적으로 어떻게 동작하는지 한 가지 사례를 정하여 작성해주세요. ex) `ArrayList`, `HashMap` 등등

### Java8 이전 HashMap

HashMap

내부적으로 hash값, key값, value값, 그리고 다음 노드를 가르키는 값을 가지고 있다.

![image](https://github.com/vanillacake369/wanted-pre-onboarding-challenge-be-task-nov/assets/75259783/0d7043e3-c36b-4a74-8ee5-ad75087842a1)


HashMap은 Hashing이라는 개념을 사용하는데, 객체를 해쉬코드로 변환해주는 hashCode() 함수를 사용한다.

내부적으로 각 연결리스트를 담고있는 bucket에 인덱스를 가르키고, 해당 슬롯에 value를 저장한다.

동일한 bucket인덱스에 해쉬될 여러 객체가 저장되는 collision을 방지하기 위해, hash chaining이라는, 동일한 해쉬값에 여러 객체들을 체이닝하는 기법을 활용한다.
![image](https://github.com/vanillacake369/wanted-pre-onboarding-challenge-be-task-nov/assets/75259783/04507349-be40-426a-8517-1d93dcb68e13)


### Java8 이후 HashMap

Java8 이후로, 연결리스트로 resolve된 hash chaining은 tree구조로 바뀌었다.

이에 따라 값 조회 시, O(n)이었던 성능이 O(log n)으로 개선되었다.

![image](https://github.com/vanillacake369/wanted-pre-onboarding-challenge-be-task-nov/assets/75259783/47b0718e-bd7e-4f7c-aed7-59a50977cfa9)


### 엥?? HashMap은 O(1)아니었나?

사실 아니다. Big O(collision 구조의 평균 크기)이다.

진정한 O(1)은 collision이 아예 없는 해싱에서 가능하다.

### HashMap vs  LinkedHashMap vs TreeMap vs HashTable

***HashMap:***

- HashMap offers O(1) insertion and retrieval time.
- It contains value based on keys.
- The ordering of keys is random
- It uses a linked list to store the data.
- It contains only unique elements
- It may have one null key and multiple null values
- It is not thread-safe

***LinkedHashMap:***

- LinkedHashMap is the same as HashMap but maintains the insertion order.
- The other properties align with that of HashMap.

***TreeMap:***

- TreeMap offers O(log N) insertion and retrieval time.
- It cannot have a null key but can have multiple null values.
- It maintains ascending order of keys.
- The rest of the properties align with that of HashMap.

***Hashtable:***

- It is thread-safe.
- Since it is thread-safe the performance might be low
- Null is not allowed for both key and value

[Internal Working of HashMap in Java.](https://prateeknima.medium.com/internal-working-of-hashmap-in-java-e5b67890e152)
