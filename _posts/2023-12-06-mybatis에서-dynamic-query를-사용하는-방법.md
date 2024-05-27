---
categories: [DB]
tags: [db,sql,mybatis]
---

스프링 어플리케이션에서 mybatis를 사용하면 쉽게 DB에 쿼리를 실행시킬 수 있다.  
일반적인 SELECT, INSERT, UPDATE, DELETE쿼리를 다루는 데에는
```sql
SELECT ITEM_NAME FROM ITEM WHERE PK = #{pk}
```
이런 식으로 `#{value}`에 값을 대입하도록 설정할 수 있다.

이 외에도 `${value}`를 사용하는 방법이 있는데
```sql
SELECT ITEM_NAME FROM ${item} WHERE PK = 1234
```
`#{value}`와 `${value}`의 핵심 차이에는 `'`가 있다.

`#{value}`로 대입할 경우에는 대입하는 데이터 타입이 문자일 경우 mybatis가 `'`로 감싸준다.  
데이터 타입이 숫자일 경우에는 `'`이 붙지 않는다.

그에 비해서 `${value}`는 데이터 타입에 상관없이 `'`가 절대로 붙지 않기 때문에 `${item}`과 같이 테이블이나 컬럼을 직접 다루는 데에 사용된다.

그러나 `${value}`는 가급적이면 사용하지 않는 것을 권장하는데, SQL injection과 같은 보안에 직접적인 위협의 원인이 되기 때문이다.

그래서 `${value}`를 사용했을때, 해당 코드를 시큐어 코딩 툴로 돌리면 다음과 같이 취약점으로 잡힌다.

![1](/assets/post/2023-12-06/01.png)

그런데 일부 특수한 상황에서는 `#{value}`는 `${value}`를 100% 대체할 수 없기 때문에 어쩔 수 없이 `${value}`를 쓰는 경우가 있는데, 시큐어코딩 진단툴은 그런 맥락을 고려하지 않고 무조건 `${value}`를 잡기 때문에 실제로 사용자가 입력 불가능한 값이라고 해도 취약점으로 잡히는 다소 억울한 일이 생긴다.  
그런 경우에 해당 서비스는 사용자가 입력불가능한 값이기 때문에 SQL injection으로부터 안전하다고 소명하면 되지만, 애초에 기술적으로 우회할 수 있다면 처음부터 취약점으로 잡히지 않게 코딩하는것이 가장 좋을 것이다.

매달 1일에 테이블에 새로운 파티션을 추가하고, 가장 오래된 파티션 하나를 삭제하는 스케줄러 서비스가 있다고 가정해보자.  
기존의 파티션 삭제 쿼리는 다음과 같이 사용했다.
```sql
<update id="dropTablePartition" parameterType="lowermap">
    ALTER TABLE ${table_name} DROP PARTITION ${partition_name}
</update>
```
프로시저를 활용해서 다음과 같이 수정할 수 있다.

```sql
<update id="dropTablePartition" parameterType="lowermap">
    BEGIN
        EXECUTE IMMEDIATE 'ALTER TABLE ' || #{table_name} || ' DROP PARTITION ' || #{partition_name} || '';
    END;
</update>
```

`EXECUTE IMMEIDATE`를 이용하여 문자로 조합되어 있는 쿼리 내용을 SQL로 실행시키면 `${value}`를 어느정도 대체할 수 있다.
