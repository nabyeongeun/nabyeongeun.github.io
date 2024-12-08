---
categories: [linux]
tags: [httpd,server]
---

새로운 서버를 구성해야 할 일이 있었는데, 분명 설정은 멀쩡한 것 같은데 이상하게 연결이 잘 안되는 문제가 있었다.  
서버 구성은 이런식이다.

![0](/assets/post/2024-11-06/img.png)

일단 의심스러운 부분은 WAF → LB 와 LB → WEB 이다.  
그러던 도중에 LB설정에 빨간 불이 들어와 있는 것을 발견하게 되어, 이 것부터 고쳐보기로 했다.

![1](/assets/post/2024-11-06/img_1.png)

기존의 `httpd.conf`내용은 다음과 같이 되어 있었다.

```bash
<VirtualHost *:80>
  ServerName   cloud.qwerty.kr
  JkMount /* worker1

  CustomLog   "logs/cloud-access_log" common
  ErrorLog    "logs/cloud-error_log"
  ErrorDocument 400 /common/common_error.jsp
  ErrorDocument 401 /common/common_error.jsp
  ErrorDocument 402 /common/common_error.jsp
  ErrorDocument 403 /common/common_error.jsp
  ErrorDocument 404 /common/common_error.jsp
  ErrorDocument 405 /common/common_error.jsp
  ErrorDocument 500 /common/common_error.jsp
</VirtualHost>


<VirtualHost *:80>

  ServerName ocean.qwerty.kr
  JkMount /* worker2

  CustomLog   "logs/inmm-access_log" common
  ErrorLog    "logs/inmm-error_log"
  ErrorDocument 400 /common/common_error.jsp
  ErrorDocument 401 /common/common_error.jsp
  ErrorDocument 402 /common/common_error.jsp
  ErrorDocument 403 /common/common_error.jsp
  ErrorDocument 404 /common/common_error.jsp
  ErrorDocument 405 /common/common_error.jsp
  ErrorDocument 500 /common/common_error.jsp
</VirtualHost>


```
WAS에 tomcat을 두 개 설치하여 독립된 서버로 운영할 계획이었기 때문에  
각 서비스의 진입점인 `cloud.qwerty.kr`과 `ocean.qwerty.kr`만 명시해 둔 것이다.  
WAF와 LB를 거치지 않고 WEB서버로 다이렉트로 붙으면 위 설정은 잘 동작한다.  

하지만 LB를 거치면 얘기가 다르게 된다.
LB는 주기적으로 WEB에다가 health check를 보내서 그 요청이 성공하면 유효한 서버로 인식하고, health check가 실패하면 유효하지 않은 서버로 인식하여 그 서버로 요청을 보내지 않기 때문이다.  
따라서 LB가 WEB으로 health check가 정상적으로 수행되는지를 확인해야 한다.

![2](/assets/post/2024-11-06/img_2.png)

카카오 클라우드에서는 위와 같이 LB → WEB으로 health check하는 것을 설정할 수 있다.  
WEB서버로 다이렉트로 붙을 때에는 정상적으로 작동하지만 위의 설정에는 `cloud.qwerty.kr`과 `ocean.qwerty.kr`만 설정되어 있으므로 LB → WEB health check가 정상적으로 수행되지 않는다.

정확하지는 않지만 아마 LB에서는 `curl -X 'GET' 'http://web-ip/'` 같은 방법으로 health check를 하는것으로 보인다.  
그래서 내 컴퓨터에서 `curl -X 'GET' 'http://web-ip/'`를 해보았는데 404 Not Found가 발생하는 것을 확인했다.

그래서 간단한 인덱스 페이지를 만들어두고 LB에서 health check를 시도했을때 인덱스 페이지를 보여주도록 설정했다.

```bash
<VirtualHost _default_:80>

  DocumentRoot "/home/qwerty/www/html"

  CustomLog "logs/default-access_log" combined
  ErrorLog "logs/default-error_log"

  <Directory "/home/qwerty/www/html">
    AllowOverride All
    Require all granted
  </Directory>

</VirtualHost>
```

아마 순서는 상관 없을 것 같지만 `<VirtualHist _default_>`를 가장 위에 작성하고 그 아래에 `cloud.qwerty.kr`과 `ocean.qwerty.kr`을 붙여서 작성했다.

![3](/assets/post/2024-11-06/img_3.png)

정상적으로 적용된 것을 확인할 수 있었다.
