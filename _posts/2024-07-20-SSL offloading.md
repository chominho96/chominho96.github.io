---
title: "SSL offloading이란?"
excerpt: "NCP에서 load balancing을 처리하는 방법"

categories:
  - security

toc: true
toc_sticky: true

date: 2024-07-20
last_modified_at: 2024-07-20
---

# 🤔 1. 도메인으로는 접속이 되는데 IP로는 접속이 안된다?

Nexters 24기에서 만든 [Payout](https://pay-out.us) 서비스에 대해 얼마 전 백엔드 팀원 분께서 질문을 주셨다.

> 저희 API 서버에 도메인으로 request를 보내면 정상적으로 response가 오는데, IP로 request를 보내면 response가 오지 않네요?? 그런데 서버 인바운드 규칙을 살펴봤는데 80포트가 0.0.0.0/0으로 설정되어 있어서 response가 와야할 거 같은데 이상하네요?!

질문을 받고 nginx.conf 파일을 살펴봤는데 다음과 같이 되어 있었다.

```nginx
user nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    sendfile on;
    keepalive_timeout 65;

    server {
      listen 80;

      # 해당 문제가 발생한 원인
      if ($http_x_forwarded_proto != 'https') {
        return 301 https://$host$request_uri ;
      }

      location /health {
        return 200 'ok';
        add_header Content-Type text/plain;
      }

      location / {
        proxy_pass http://green-api:8080;
        proxy_set_header    Host                $http_host;
        proxy_set_header    X-Real-IP           $remote_addr;
        proxy_set_header    X-Forwarded-For     $proxy_add_x_forwarded_for;
      }
    }
}
```

문제의 원인은 금방 찾을 수 있었다.

<span style='color: red'>http_x_forwarded_proto 헤더는 로드 밸런서 원본 요청에서 사용된 프로토콜</span>을 의미한다.

그런데 IP로 직접 접근을 시도하면 해당 헤더가 설정되어 있지 않기 때문에, if문이 ture가 되어 실행되고, https로 리다이렉트되게 된다.

이때 <span style='color: red'>443 포트를 listen하도록 설정되어 있지 않아 ERR_CONNECTION_REFUSED 에러가 발생</span>하는 것이다.

문제에 대한 직접적인 답은 도출했는데, 여기서 의문이 들었다. 현재 서비스는 https로 서비스되고 있는데, 그렇다면 어떻게 https로 서비스가 되고 있는건지 이해가 안 간 것이다.

(그렇게 원인을 분석하다가 오늘의 주제인 SSL offloading까지 왔다...)

# ☁️ 2. NCP (Naver Cloud Platform) 분석 과정

우리 서비스는 NCP를 이용해서 서비스되고 있었기 때문에 NCP에서 원인을 추정할 수 있을 것이라고 생각해서 서버 구축을 처음부터 한다고 생각하고 각 서비스를 살펴봤다.

먼저, 서버가 load balancer에 연결되어 있기 때문에 Load balancer를 살펴봤다.

![제목 없음](https://github.com/user-attachments/assets/28672d91-bab5-4606-8783-b10bb7d31e69)

그 결과 다음과 같이 80/443으로 들어오는 트래픽이 모두 서버의 80번 포트로 들어가고 있음을 확인하였다.

(만약 로드밸런서를 통해 http로 연결을 시도한다고 하면, nginx.conf의 if문에 의해 다시 https로 redirect될 것이고, 따라서 다시 https로 접근을 시도할 것이다. 따라서 로드 밸런서의 포트는 최종적으로는 항상 443이 될 것이다.)

이때 의문이 들었던 점은 <span style='color: red'>서버가 80번 포트로 서비스 중인데, 어떻게 로드밸런서 포트가 443이라 해서 https가 정상적으로 적용될 수 있는가</span>였다.

더 알아보기 위해 [load balancer 공식문서](https://guide.ncloud-docs.com/docs/loadbalancer-classiclb-classic#1-%EB%A1%9C%EB%93%9C%EB%B0%B8%EB%9F%B0%EC%84%9C-%EC%83%9D%EC%84%B1)를 살펴보던 도중, 실마리를 찾을 수 있었다. (링크의 "1.로드밸런서 생성 - 참조"를 참고하자.)

![제목 없음](https://github.com/user-attachments/assets/9a5a54a5-6192-40b8-90fa-7b91de3eefed)

요약하자면, SSL Offloading이 적용되어 있기 때문에 서버에서는 https로 서비스를 제공하지 않아도 된다는 뜻이다.

즉, SSL offloading이 https로 서비스가 될 수 있도록 무언가를 제공한다는 것인데, SSL offloading이 무엇인지 알아보자.

# 🔒 3. SSL Offloading

SSL offloading은 L4 switch (Load balancing을 처리하는 장비)의 HTTPS handshake overhead를 줄이기 위한 기법이다.

![제목 없음](https://github.com/user-attachments/assets/b3deb091-d432-440b-a115-243643ac2993)

HTTPS handshake란, https 통신을 맺게 위해 서버와 클라이언트 간 통신을 맺는 과정이다. HTTP를 위한 handshake가 끝나고 나서, 별도로 진행된다.

SSL offloading이 적용되지 않은 로드밸런서-서버의 구조는 다음과 같다.

![제목 없음](https://github.com/user-attachments/assets/1d4191c3-867e-4cc5-bd97-dbccdb5a9bfa)

이 구조에서는 2번의 HTTPS 통신이 발생하게 된다.

1. 클라이언트와 로드 밸런서 간 HTTPS 통신
2. 로드 밸런서와 서버 간 HTTPS 통신

이때 서버는 서비스 로직을 수행할 뿐만 아니라 SSL handshake까지 수행해야 하므로, 이에 대한 오버헤드가 증가하게 된다.

따라서, <span style='color: red'>클라이언트와 로드 밸런서 간 HTTPS 통신만 진행하고, 로드 밸런서와 서버는 HTTP로 통신</span>을 진행하도록 하는 기법이 SSL offloading이다.

![제목 없음](https://github.com/user-attachments/assets/4c65a6fd-bfc7-4685-98e6-f20b51b2b2a3)

위 그림과 같이 SSL offloading을 적용하게 되면, 서버에 가해지는 overhead가 적어짐으로써 성능이 향상될 수 있다.

이로 인해 <span style='color: red'>SSL offloading을 적용한 load balancer을 도입할 경우, 서버가 HTTP 포트만 제공하도록 설정하여도 HTTPS 서비스를 제공</span>할 수 있게 되는 것이다.

# 4. 👀 정리

문제로 다시 돌아와서 IP로 접근이 안되는 원인을 정리해보자면 다음과 같다.

1. IP로 서버에 접근하게 되면 다음과 같은 이유로 접근이 불가능하다.

   1. http로 접근할 경우 nginx.conf의 if문으로 인해(http_x_forwarded_proto 헤더가 없으므로) https로 리다이렉트되는데, 443 포트를 nginx가 listen하고 있지 않기 때문에 ERR_CONNECTION_REFUSED 에러가 발생한다.
   2. https로 접근하는 경우도 마찬가지로 ERR_CONNECTION_REFUSED 에러가 발생한다.

2. 도메인으로 서버에 접근하게 되면 다음의 이유로 접근이 가능하다.

   1. https로 접근할 경우 SSL offloading이 적용되기 때문에, <span style='color: red'>로드 밸런서부터 서버까지 암호화/복호화를 하지 않고도 https를 제공</span>할 수 있다. 즉, 서버는 HTTP만 제공하더라도 HTTPS로 서비스가 가능하다. 또한, <span style='color: red'>로드 밸런서로 들어온 원본 프로토콜(http_x_forwarded_proto)이 https</span>이므로, if문이 false가 되어 수행되지 않는다.
   2. http로 접근할 경우 로드밸런서로 들어온 원본 프로토콜이 http이므로 if문에 걸려서 https로 리다이렉트되는데, 이때 <span style='color: red'>다시 요청을 보낼 때는 다시 로드밸런서를 통해 요청</span>되므로 1번 케이스가 다시 수행된다.

문제의 발단이 되었던 nginx.conf의 if문은 단순히 http로 접근했을 때 https로 리다이렉트하려고 삽입한 코드였다. 당연히 SSL offloading을 고려해서 IP로의 접근을 막기 위해 넣은건 아니다ㅋㅋㅋㅋ😂 그런데 저 if문으로 인해 IP 직접 접근을 차단한 셈이 된 것이다.

문제를 분석하는 과정에서 제대로 된 이해 없이 마구잡이로 "Load Balancer을 붙이고, 인증서 붙여서 https 적용해야지"라는 생각으로 서버 구축을 했던 것에 대해 반성했다.

그때 당시를 되돌아보면, 서버를 연동할 당시에 NCP에서 80번 포트만 사용해도 된다는 말에 의문을 가지고 찾아봤어야 하는데, 개발 시간이 촉박해서 그냥 곧이곧대로 사용했던 것 같다.

팀원 분의 좋은 질문으로 SSL offloading이라는 새로운 개념을 학습해서 유익하기도 했고, 무엇보다 앞으로 어떤 기술을 사용할 때 그 기술이 어떤 방식으로 동작하는지 먼저 이해하려는 태도를 가져야겠다고 생각했다.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
