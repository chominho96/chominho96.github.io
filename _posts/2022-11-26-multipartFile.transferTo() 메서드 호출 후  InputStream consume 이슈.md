---
title: "multipartFile.transferTo() 메서드 호출 후 InputStream consume 이슈"
excerpt: "Spring과 AWS를 이용한 파일 업로드"

categories:
  - spring

toc: true
toc_sticky: true

date: 2022-11-26
last_modified_at: 2022-11-26
---

# ⚡ 문제 발생

프로젝트를 진행할 때, 파이썬 코드를 이용해서 서버 내부에서 이미지 변환을 거친 후, 변환된 이미지를 AWS S3에 업로드하는 로직을 작성하고 있었다.

따라서 해당 로직 프로세스는 다음과 같다.

  1. 사용자로부터 MultipartFile을 넘겨받음
  2. 파일을 EC2 내부 서버에 저장
  3. EC2 내부의 파일 이미지 변환
  4. EC2 내부의 변환된 이미지 AWS S3에 업로드
  5. 원본 MultipartFile도 AWS S3에 업로드

그런데 4번 과정까지는 정상적으로 진행되었는데, 5번 과정, 즉 처음에 받았던 MultipartFile을 AWS S3에 업로드하는 과정에서 문제가 발생했다. (에러 로그는 비슷한 다른 질문에서 가져왔다.)

``` java
java.io.FileNotFoundException: C:\Users\lucas.kahler\AppData\Local\Temp\tomcat.1947057742180166642.8080\work\Tomcat\localhost\ROOT\upload_51753fdf_0308_49d4_800c_bd95bd7760f3_00000001.tmp
```

처음에는 하도 EC2에 많이 데여서 저 .tmp 파일은 어디에 있는 것이고 왜 왜 오류를 일으키는지, 즉 .tmp 파일에 집중했다.

하지만 오류가 발생한 위치가 MultipartFile을 AWS S3에 업로드하는 부분이었는데, 지금까지 단 한번도 오류가 발생하지 않았던 부분이라 다른 이유가 있지 않을까 하고 원인을 다시 찾아봤던 것 같다.


# 😊 해결 - transferTo() 메서드 관련 이슈

다음 글에서 해결책을 찾을 수 있었다.

https://stackoverflow.com/questions/59214385/java-io-filenotfoundexception-for-a-present-multipartfile

이를 해석해보자면, MultipartFile가 transferTo() 메서드를 거친 순간, 관련된 InputStream이 소모되기 때문에, 더 이상 해당 MultipartFile 객체를 사용할 수 없다는 내용이다.

앞서 설명한 프로세스의 2번 과정에서 파일을 EC2 내부에 저장할 때, transferTo() 메서드를 사용한게 원인이었다.

따라서 그냥 MultipartFile을 그대로 AWS S3에 업로드하는 로직은 문제가 없었지만, 중간에 transferTo() 메서드를 실행함으로써 문제가 발생했던 것이다.

따라서 AWS S3에 업로드하는 로직을 File 객체를 받아서 실행되도록 변경하니, 문제가 사라졌다.

# 정리
1. MultipartFile에 대해 transferTo() 메서드를 사용하고 나서는 해당 MultipartFile 객체를 다시 사용할 수 없다.
2. 따라서 다시 MultipartFile 객체를 사용해야 하면, 변환된 File 객체를 이용해야 한다.

같은 기능이라고 생각하고 구현해도, 매번 다른 상황 속에서 항상 새로운 이슈가 발생하는 것 같다.

오류가 발생했을 때 전에 개발했었던 것 같던 코드 같더라도 분명 다른 점이 있으며, 이를 찾아내는 것이 빠르게 오류를 해결하는 방법인 것 같다.


> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
