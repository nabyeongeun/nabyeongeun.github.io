---
categories: [Java]
tags: [java,jpa]
---

SpringBoot로 JPA를 통해 DB에 연결할때 다른 RDBMS는 괜찮은데 유난히 오라클에서만 임시테이블을 만드는 경우가 있다.

```sql
CREATE SEQUENCE SEQ INCREMENT BY 50 START WITH 1 MAXVALUE 9999999999999999;
```

```java
@Entity @Getter
@Table(name = "USERS")
public class User {

  @Id @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "SEQ")
  private Long userKey;
}
```

오라클에서는 키값을 부여할 때 주로 시퀀스를 많이 사용하기 때문에 시퀀스를 준비하고 User객체를 만들때 JPA가 시퀀스를 통해 key를 자동부여하도록 설정했다.

![1](/assets/post/2024-07-04/01.png)

그런데 ddl-auto를 사용하지 않음으로 지정했음에도 불구하고 `HTE_USERS`라는 이름의 global temporary table이 자동생성된다.

이 임시 테이블이 생기는 것은 사실 `@GeneratedValue` 설정이 반쪽짜리이기 때문이다.  
`generator = "SEQ"`라고 적었기 때문에 `SEQ`시퀀스를 사용하는 것 처럼 보이지만 정확히 말하자면 시퀀스 이름이 아니라 generator를 넣어야 한다.  

```java
@Entity @Getter
@SequenceGenerator(
        name = "SEQUENCE_GENERATOR",
        sequenceName = "SEQ",
        allocationSize = 50
)
@Table(name = "USERS")
public class User {

  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "SEQUENCE_GENERATOR")
  private Long userKey;
}
```

그래서 이런 식으로 `@SequenceGenerator`를 지정해서 `@GeneratedValue`에다가 generator이름을 입력하면 더이상 JPA가 임시테이블을 생성하지 않는다.  

`@SequenceGenerator`가 없으면 시퀀스 설정이 적절하게 설정되지 않은 것으로 판단하여 JPA에서 insert시의 성능 최적화를 위해서 자동으로 임시테이블을 만든다고 한다.

이 문제를 해결하려고 검색을 많이 해봤는데 정보가 많이 나오지 않는 것으로 보아하니 오라클에서만 발생하는 문제인 것 같다.

