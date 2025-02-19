---
categories: [linux]
tags: [linux]
---

이전 까지는 스프링으로 된 레거시 프로젝트만 하다가 최근에 React와 Springboot로 새로운 프로젝트를 수행하게 되었다.  
지금은 프로젝트가 거의 마무리 되는 시점이라 운영서버를 준비하다가 겪은 시행착오를 정리하려고 글을 작성한다.

# Requirements
일단 React로 화면을 만들고 Springboot로 백엔드를 구성하여 개발을 했고, Spring이나 Express등의 웹서버 역할을 하는 서버 프레임워크를 사용하지 않고 Nginx에 SSL인증서를 적용하여 서비스 하는 것을 전제로 아래 내용을 작성한다.  
이 상황에서 일단 중요한 것은 SSL인증서를 2번 적용해야 한다는 것이다.  
만약에 Express나 Spring을 통해서 별도의 웹 서버를 구성했다면 웹서버에만 SSL인증서를 적용하면 화면 내의 모든 상호작용은 일단 웹서버가 받은 다음에 백엔드로 향하기 때문에 request의 내용을 가려주지만,  
그렇지 않고 단순히 Nginx에 React 빌드 결과물을 올려놓으면 Nginx는 html, css, js등의 `static resource`만 내려주고, 실제로 DB를 거치는 request는 백엔드까지 도달한다.  
이 때, Nginx에만 SSL인증서를 적용시키면 브라우저 상에서는 `static resource`는 `https`인데 `dynamic resource`는 `http`로 내려오기 때문에 브라우저 기본 설정으로 `http` request를 차단해버린다.  
Nginx의 reverse proxy는 WAS가 내부망에 위치해 있어서 직접 접근할 수 없으므로 웹서버에 설치된 Nginx를 한 번 경유하게 설정할 뿐이지, SSL인증서를 적용했다고 reverse proxy로 튕겨내는 `dynamic resource`도 `https`로 감싸주지 않는 다는 것을 이해하는 것이 가장 중요하다.


# 1. Nginx에 SSL 인증서 적용하기
SSL인증서를 발급받는 것은 이 글의 주제가 아니기 때문에 다루지 않고, 인증서를 이미 발급받았다는 전제 하에 설명한다.  
Nginx설정 하는 것 자체도 다른 블로그에 잘 설명되어 있기 때문에 자세히 다루지는 않으며 전체적인 흐름을 설명하는 것을 주 목적으로 한다.

SSL인증서를 발급받으면 대략 이런 파일을 받게 된다.

```
ca-bundle.crt
qwerty.crt
qwerty.key
```

이 때 `qwerty.crt`와  `qwerty.key`를 이용하면 Nginx에 SSL인증서를 적용한다.


# 2. Tomcat에 SSL인증서 적용하기

일단 개발자PC에 openssl을 설치한 후, openssl과 위의 인증서를통해 tomcat에 적용할 파일을 생성해야 한다.  
```
openssl pkcs12 -export -in qwerty.crt -inkey qwerty.key -out keystore.p12 -name mycert
```
위의 명령어를 입력하면 비밀번호를 요구하는데 임의의 비밀번호를 입력하면 `keystore.p12` 파일이 생성된다.

파일이 정상적으로 생성되었으면 WAS서버의 배포파일과 동일한 디렉토리에 `keystore.p12`파일을 위치시키고 동일한 위치에 `application.properties`를 생성한 후 다음 내용을 기입한다.

```
server.ssl.key-store-type=PKCS12
server.ssl.key-store=keystore.p12
server.ssl.key-store-password=아까입력한 비밀번호
server.ssl.key-alias=mycert
server.ssl.enabled=true
```
프로젝트 내의 `application.yml`파일에 yml양식으로 입력해도 동일하게 적용되지만, 그렇게 하면 로컬에서 서버를 띄울 때에도 ssh로 테스트 해야하므로 운영서버에만 적용시키기 프로젝트와 분리시키기 위해 운영서버에만 적용되는 별도 설정으로 분리시키는 것을 권장한다.

설정을 완료한후 톰캣을 실행시켰을 때 로그에 `https`라는 단어가 보이면 SSL적용에 성공한 것이다.

# 3. WEB서버 호스트 설정

다시 웹서버로 돌아와서, WAS가 정상적으로 구축되었는지 확인을 해보자.
```
curl -v https://192.168.123.123:8080/testService
```
톰캣에 SSL설정이 정상적으로 적용되었다면 테스트에 실패하는 것이 정상이다.  
도메인이 명시된 인증서를 사용했기 때문에 private IP를 통한 request가 차단당하게 된다.
따라서 WEB → WAS 통신에도 인증서에 명시된 도메인을 통해 수행하게 해야 한다.
```
curl -v https://www.qwerty.co.kr:8080/testService
```
그런데 저 도메인과 public IP를 이어주는 DNS에는 웹서버의 public IP가 명시되어 있기 때문에 위 request는 `localhost:8080`을 대상으로 테스트 하는 것이나 다음이 없다.  

그래서 웹 서버의 호스트 설정이 필요하다.  
레드햇이나 우분투 계열의 리눅스는 `/etc/hosts`를 DNS서버처럼 사용하기 떄문에 여기에 인증서의 도메인과 WAS의 private IP를 명시하면 도메인 호출로 WAS의 private IP로 request를 보낼 수 있다.

`/etc/hosts`파일이 존재한다면 파일을 열어서 내용을 추가하면 되고, 파일이 존재하지 않는다면 다음과 같이 생성과 내용기입을 동시에 할 수 있다.

```
cat "192.168.123.123 www.qwerty.co.kr" >> /etc/hosts
```

그리고 다시 테스트를 했을때 성공하면 운영서버 기초공사가 완료된 것이다.

```
curl -v https://www.qwerty.co.kr:8080/testService
```
