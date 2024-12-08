---
categories: [spring]
tags: [spring,java]
---

개발을 하다보면 개발 로컬환경과 운영환경에 다른 값을 사용하거나 운영 환경이라도 다른 값을 설정하는 경우가 있다.  
예를 들면 이중화가 구성된 상황에서 일일 정산작업을 할때 양 쪽 WAS에서 동일한 작업을 하면 데이터 무결성 문제가 발생할 수 있으므로 한쪽 WAS에만 정산을 맡기기 위하여 WAS1과 WAS2에 구분되는 값을 하나 지정하여 정산 여부를 판단하는 경우가 있다.

스프링 부트를 도입하기 전에 전통적인 방법의 Spring Framework를 사용할 때에는 tomcat내의 catalina.properties에다가 값을 지정한 후에 코드 내에서 `System.getProperty`를 통해 간단하게 위에서 언급한 내용을 적용할 수 있었다.  
그러나 스프링부트는 jar파일로 배포하면 jar내부에 모든 코드와 설정값이 포함되기 때문에 값을 변경해가며 배포파일을 만들기에는 너무 불필요하게 번거로움이 크고 휴먼에러가 발생할 가능성이 컸다.  

따라서 jar배포파일 외부에 설정파일을 정의하여 jar를 실행시킬 때마다 설정파일을 참조하여 Custom value를 초기화 할 수 있으면 좋겠다고 생각했다.  
그러면 새로운 패치를 적용하더라도 custom value가 적용된 파일은 배포서버에 가만히 두고 jar파일만 교체하여 패치를 하면 된다.

![1](/assets/post/2024-08-08/01.png)

스프링부트로 프로젝트 구조는 대부분 이런 식이다.  
여기 중요한 것은 `applcation.properties`를 내가 하나 더 만들어서 `application`파일이 2개라는 점이다.  
왜 2개인지는 뒤에서 설명하기로 하고, 일단 `application.properties`에 내가 지정하고 싶은 값을 정의하면 된다.

![2](/assets/post/2024-08-08/02.png)

이렇게 값을 정의한 후에 다음과 같은 식으로 java코드 내에서 값을 호출하여 사용할 수 있다.

```java
@Slf4j @Getter
@Component
public class EnvConfiguration {

    private final Environment env;
    private String myValue;

    public EnvConfiguration(Environment env) {
        this.env = env;
    }

    @PostConstruct
    public void init() {
        log.error("EnvConfiguration ▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼▼");
        myValue = env.getProperty("myValue");
        log.error("myValue = " + myValue);
        log.error("EnvConfiguration ▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲▲");
    }
}
```
![3](/assets/post/2024-08-08/03.png)

의도한대로 정상적으로 적용됨을 확인할 수 있다.

여기서 중요한 점은 `Environment`를 사용했다는 것이다.  
`Environment`클래스는 인터넷에 `스프링부트 환경변수` 또는 `스프링부트 custom value`등으로 검색했을때 `application.yml`에 있는 값을 쉽게 가져오는 방법을 설명하는 글을 볼 수 있다.

그런데 이 글에서 제시하는 방법은 `application`이 두개라는 점에서 차이가 있다.

만약에 `application.properties`가 없었다면 `env.getProperty()`는 `application.yml`에서 `myValue`를 찾았을 것이다.  
`application.properties`와 `application.yml`은 근본적으로 동일한 역할을 한다.  
다만 차이가 있다면, 빌드해서 배포파일을 만들때 `application.yml은` 배포파일 안에 들어가고 `application.properties`는 배포파일에 포함되지 않는다는 것이다.

따라서 배포 환경에는 다음과 같이 배포파일과 설정파일을 하나의 디렉토리에 몰아넣고 실행시키면 적용된다.

![4](/assets/post/2024-08-08/04.png)

이 것과 유사한 방법으로는 배포파일을 실행시킬때 매개변수로 넣는 방법이 있다.

```bash
java -jar application.jar --spring.profiles.active=prod --server.port=8090
```
이런식으로도 프로그램을 실행하는 시점에 별도의 변수를 정의한 다음에 위에서 `Environment`를 사용한 것과같이 변수를 가져다 쓸 수 있지만, 이런 방법은 값이 많아질 수록 관리하기 어려워 지기 때문에 개인적으로는 별로 선호하지 않는다.

이런 여러가지 방식을 도입하다보면 값이 많아졌을 때 값이 많이 파편화되면 `myValue`가 어디에 정의되어 있는지 파악하기 어려워질 수 있다.

그래서 나는 db connection, tomcat configuration, server port 등 프로그램 로딩 시점에 필수 데이터들은 `application.yml` 또는 `start.sh`에 정의하고, 필수 객체가 초기화 된 후에(`@PostConstruct`가 적용된) 나중에 초기화 되는 값들은 `application.properties`에 정의하는 것을 원칙으로 하고 있다.
