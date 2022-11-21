---
title: "Nginx Request Entity Too Large 오류"
excerpt: "Elastic Beanstalk을 이용한 배포"

categories:
  - aws

toc: true
toc_sticky: true

date: 2022-11-21
last_modified_at: 2022-11-21
---

# ⚡ 문제 발생

엘라스틱 빈스톡을 이용해서 Spring + Thymeleaf로 구성된 프로젝트를 배포하여 프로젝트를 수행하고 있던 중에 오류를 발견했다.

![J2B_NFT_INHA (2)](https://user-images.githubusercontent.com/66549638/203082523-57eecbe0-9cbb-40e5-acb7-d96f0c4c72da.png)

파일을 포함한 Form을 전송받을 때 발생한 오류였는데, 처음 보자마자 든 생각은 '파일의 크기가 너무 큰가?'였다.

에러 메시지가 Too Large라고 명시되어 있었기 때문이었다.

하지만 이해가 되지 않았다. Spring 상에서 다음과 같이 업로드 가능한 최대 파일 크기를 설정했기 때문이다.

![J2B_NFT_INHA (2)](https://user-images.githubusercontent.com/66549638/203083121-19bb09a5-b3cc-4d46-9a2c-d6fd10c2c926.png)

설마 내가 올린 파일이 20MB나 넘어가나하며 파일 크기를 확인해봤지만, 5MB도 되지 않는 파일이었다.

따라서 스프링 상에서의 오류가 아니라 nginx 상에서의 오류라고 판단했고, 구글링을 통해 해결 방법을 찾았다.


# 😊 해결 : nginx.conf 설정

413 Request Entity Too Large라고 검색만 해도 바로 해결 방법을 알 수 있었다.

원인을 정리해보자면 다음과 같다.

- 스프링의 설정과 무관하게 nginx 서버로 넘어오는 HTTP Request의 최대 크기가 정해져있다.
- 이 크기가 작게 설정되어 있다면, 스프링 상에서 파일의 최대 크기를 크게 설정하더라도, 큰 파일 업로드가 불가능한 것이다.

따라서 nginx 상에서의 최대 업로드 파일 크기를 변경해줘야 한다. 이는 nginx.conf 파일에서 변경할 수 있다.

참고로 필자가 사용하고 있는 운영체제는 다음과 같다.

![제목 없음](https://user-images.githubusercontent.com/66549638/203084600-33f1f101-bf28-4494-89b0-98848c1e2ff1.png)

운영체제별로 nginx.conf의 위치가 다를 수 있기 때문에, 각자의 운영체제에 맞게 nginx.conf 파일을 찾아주면 된다.

필자의 경우, /etc/nginx 디렉토리에 nginx.conf 파일이 있는 모습을 확인할 수 있었다.

nginx.conf 파일에서 다음을 추가해주면 된다.

```shell
http {
  ...
  client_max_body_size 20M; # 원하는 용량으로 설정
}
```

이와 같이 설정하면 최대 20M의 Request를 보낼 수 있게 된다.

이렇게 수정하고 정상적으로 Request가 보내지는 것을 확인했다.


# 정리
1. nginx 서버를 통해서 파일 업로드를 구현할 경우, 대부분 최대 업로드 파일 크기가 작으므로, 스프링의 설정과 더불어 nginx 상에서도 설정을 변경해야 한다.
2. 이는 nginx.conf 파일에서 변경할 수 있으며, client_max_body_size 옵션을 주어 수정하면 된다.

다만 한가지 한계점이, 재배포를 진행할 때마다 nginx.conf 파일이 초기화되어서 재배포할 때마다 nginx.conf 파일을 다시 수정해줘야 한다는 점이다. 배포 스크립트 등을 이용해서 추후 수정해야겠다.



> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
