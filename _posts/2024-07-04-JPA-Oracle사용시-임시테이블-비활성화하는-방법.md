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

이 임시 테이블이 생기는 것은 사실 `allocationSize` 가 1보다 크기 때문이다.  
`allocationSize`가 1보다 크면 내가 실제로 `multi-row inserts`를 사용하지 않더라도 이를 대비하여 JPA가 global temporary table을 자동으로 생성한다.

> https://discourse.hibernate.org/t/hte-prefix-added-on-default-to-database-table-names/7189/6

```sql
CREATE SEQUENCE SEQ INCREMENT BY 1 START WITH 1 MAXVALUE 9999999999999999;
```

```java
@Entity @Getter
@SequenceGenerator(
        name = "SEQUENCE_GENERATOR",
        sequenceName = "SEQ",
        allocationSize = 1
)
@Table(name = "USERS")
public class User {

  @Id
  @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "SEQUENCE_GENERATOR")
  private Long userKey;
}
```

따라서 이런 식으로 `allocationSize`를 1로 줄이면 더이상 오라클에 임시테이블을 만들지 않는다.
