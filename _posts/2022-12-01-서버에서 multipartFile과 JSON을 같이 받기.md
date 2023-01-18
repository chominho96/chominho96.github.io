---
title: "서버에서 multipartFile과 JSON을 같이 받기"
excerpt: "Spring과 REST API"

categories:
  - spring

toc: true
toc_sticky: true

date: 2022-12-01
last_modified_at: 2022-12-01
---

# ⚡ 문제 발생

어느 프로젝트를 진행하더라도 항상 하나의 Request에서 DTO 등의 데이터와 이미지 (multipartFile)을 함께 받는 상황이 발생한다.

그때마다 까먹어서 구글링과 기존에 했던 프로젝트를 참조해서 진행했는데, 더 이상은 안되겠어서 정리하고자 한다!

# 😊 해결 - Postman 테스트 방법도 설명!

먼저 @RequestParam 또는 @RequestBody를 통해서 데이터를 받던 기존의 방식과 달리, 이미지와 데이터를 함께 받는 경우에는 @RequestPart 애노테이션을 활용해야 한다.

컨트롤러에서는 다음과 같이 작성하면 된다.

```java
public String test(@RequestPart TestDTO dto, @RequestPart List<MultipartFile> files) {
  ...
}
```

즉, 받고자 하는 DTO와 파일(리스트로 여러 파일을 받아와도 되고, 단일 파일만을 받아와도 된다.)을 모두 @RequestPart 애노테이션을 통해 받으면 된다.

이제 이를 Postman을 통해 테스트하려면 다음과 같이 작성하면 된다.
![제목 없음](https://user-images.githubusercontent.com/66549638/213118491-41c71b9d-3890-462d-8cfe-9b3f1f6aca93.png)

사실 이 부분을 항상 까먹어서 글로 정리하고자 한 것인데, dto(이미지가 아닌 일반 데이터)의 content type을 application/json으로 지정해줘야 한다.

지정해주지 않을 경우 다음과 같은 에러를 만날 것이다.

```
Resolved [org.springframework.web.HttpMediaTypeNotSupportedException: Content type 'application/octet-stream' not supported]
```

# 정리

1. 이미지와 데이터를 한번에 Request로 받고자 하면, @RequestPart 애노테이션을 사용한다.
2. 이를 Postman으로 테스트할 경우, dto의 content type을 application/json으로 지정해주는 것을 잊지 말자.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
