---
categories: [DB]
tags: [db,sql,oracle,tibero]
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
qwert.key
```

이 때 `qwerty.crt`와  `qwerty.key`를 이용하면 Nginx에 SSL인증서를 적용한다.


# 2. Tomcat에 SSL인증서 적용하기

위에서 언급한 인증서를 이용해서 key

# 3. WEB서버 호스트 설정

2번 과정에서 SSL인증서를 설정했기 때문에 더이상 Tomcat에 `http` request를 보낼 수 없다.  
따라서 SSL인증서에 명시된 도메인 규칙을 이용하여 `https` request를 보내야만 정상적인 응답을 받을 수 있다.

`cat "192.168.123.123 www.qwerty.co.kr"`
