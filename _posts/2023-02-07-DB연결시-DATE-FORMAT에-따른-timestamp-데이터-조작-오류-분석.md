---
categories: [DB]
tags: [edb,postresql]
---

기존의 오라클DB를 사용하는 프로그램이 있는데, 오라클은 라이센스 비용이 부담스럽기 때문에 비용부담이 적은 DB를 사용하고 싶다는 요청이 오래 전부터 있어왔다.

단순히 DB connection만 바꿔서 해결될 문제가 아니라, 기존에 오라클 종속적으로 작성된 쿼리를 전부 ANSI SQL로 바꿔야 하므로 상당히 부담스러운 작업이기 때문에 차일피일 미루기만 했는데, 어떤 업체의 솔루션을 사용하면 오라클 쿼리 그대로도 PostgreSQL를 사용할 수 있다는 정보를 알게 되어 해당 업체의 솔루션을 통해서 PostgreSQL을 사용할 수 있을지 시험해보고 있었다.

해당 업체의 주장으로는 `기존 오라클 쿼리의 95%를 PostgreSQL에서도 사용할 수 있다.`였는데, 직접 사용해보니 로그인부터 오류가 발생했다.

```
[2023-02-07 10:40:04,220] [ERROR] [jdbc.audit:128]- 8. ResultSet.getTimestamp(accession_date)
com.edb.util.PSQLException: Bad value for type timestamp/date/time: 2017/05/17
    at com.edb.jdbc.TimestampUtils.parseBackendTimestamp(TimestampUtils.java:364)
    at com.edb.jdbc.TimestampUtils.toTimestamp(TimestampUtils.java:396)
    at com.edb.jdbc.PgResultSet.getTimestamp(PgResultSet.java:656)
    at com.edb.jdbc.PgResultSet.getTimestamp(pgResultSet.java:2924)
    at net.sf.log4jdbc.sql.jdbcapi.ResultSetSpy.getTimestamp(ResultSetSpy.java:357)
    at org.apache.ibatis.type.SqlTimestampTypeHandler.getNullableResult(SqlTimestampTypeHandler.java:35)
    at org.apache.ibatis.type.SqlTimestampTypeHandler.getNullableResult(SqlTimestampTypeHandler.java:24)
```
```
Caused by: java.lang.NumberFormatException: Expected time to be colon-separated, get '/'
    at com.edb.jdbc.TimestampUtils.parseBackendTimestamp(TimestampUtils.java:267)
    ... 92 common frames omitted
```
해당 업체가 제공하는 마이그레이션 툴을 이용하여 기존 오라클DB의 모든 테이블과 데이터를 PostgreSQL로 옮겼다. 기존에 Date타입이었단 컬럼들은 전부 Timestamp로 자동 변환되었고, Timestamp타입의 데이터를 취급할 때에 오류가 발생하는 것을 확인하였다.

해당 오류에 의문이 생겨서 jdbc.jar파일을 디컴파일하여 분석 해 보았다.

![1](/assets/post/2023-02-07/01.png)

해당 jdbc에서는 timestamp데이터를 문자열로 늘어놓고 문자 단위로 파싱하는 구조였는데, 문자단위로 파싱을 하다보니 데이터의 길이가 원래 예상한 것과 다르거나, separator가 정확히 일치하지 않으면 필연적으로 오류가 발생할 수 있는 구조였다.

위의 Exception trace와 디컴파일 결과를 종합하여 보았을 때, timestamp는 "년-월-일 시:분:초" 구성되어 있어야 하는데, 실제로 가져온 데이터는 "년/월/일" 이기 때문에 데이터를 파싱하는 부분에서 exception이 발생한 것으로 보였다.

그런데 이상하게 느낀 점은, DB에서 Timestamp 타입의 데이터를 가져오면 실제로 java.sql.Timestamp 객체로 변환되어 가져 올 것인데, 왜 시간 데이터가 누락되어 있을까 였다.


그래서 여러가지를 찾아보던 도중에, 프로그램의 DB connection 설정 옵션에 문제가 있음을 깨달았다.

![1](/assets/post/2023-02-07/02.png)

기존의 회사에서 관습적으로 쓰던 코드에 붙어있는 옵션인데, 위의 NLS_DATE_FORMAT 설정으로 인해서 영향을 받은 것으로 보인다.
해당 옵션으로 인해 날자/시간과 관련된 데이터를 조작할 때, 자동으로 YYYY/MM/DD만 가져오도록 설정되었고, 그로인해 jdbc에서 시간 데이터가 누락된 채로 날자 데이터만을 가지고 파싱을 시도하다가 오류가 발생한 것이다.

정확한 내용을 분석하기는 어렵지만 아마도 오라클 jdbc에서는 시간관련된 데이터를 처리할때 Calendar, Timestamp, Date 같은 클래스를 사용하여 시간 데이터 없이도 넘어갔고, postreSQL에서는 문자 기반의 파싱을 하기 때문에 해당 오류가 발생했거나, 오류처리가 미흡했던 것으로 보인다.

아무튼 해당 옵션값을 'YYYY-MM-DD HH24:MI:SS'로 변경하여 Timestamp 타입의 데이터를 조작하는데 문제가 없는 것을 확인하였다.
