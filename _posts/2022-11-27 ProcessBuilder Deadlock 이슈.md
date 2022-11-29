---
title: "ProcessBuilder Deadlock 이슈"
excerpt: "Spring에서 bash 명령어 실행하기"

categories:
  - spring

toc: true
toc_sticky: true

date: 2022-11-27
last_modified_at: 2022-11-27
---

# ⚡ 문제 발생

ProcessBuilder 클래스를 이용해서 bash 명령어를 실행하려고 구현하는 과정에서 너무 답답해서 ProcessBuilder 안에서 처리되는 내용을 로그로 남겨보려고 다음과 같이 설계했다.

```java
ProcessBuilder builder = new ProcessBuilder(command);
Process process = builder.start();

BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
String line;
while ((line = reader.readLine()) != null) {
    log.info(line);
}

int exitCode = process.waitFor();
```

그런데 로그가 남긴 커녕 오류라도 뿜어줘야 하는데 해당 로직을 실행하자 무한 로딩이 걸리다가 Time out이 났다...

내가 수정한 코드는 이 부분 밖에 없었기 때문에, 이 부분에서 오류가 발생했을 것이라고 생각하고 구글링한 결과, 원인을 알 수 있었다.


# 😊 해결 - 프로세스가 입력스트림과 출력스트림을 사용할 때 실패가 발생하여 Deadlock 발생

다음 글에서 해결책을 찾을 수 있었다.

https://d2.naver.com/helloworld/1113548

이를 정리해보자면, 프로세스가 입력스트림, 또는 출력스트림을 사용할 때 실패가 발생하여 Deadlock에 빠질 수 있다는 것이다.

따라서 내 경우에서도 process.waitFor() 함수에서 Deadlock이 발생하여 무한 루프에 빠졌던 것이다.

이는 다음과 같이 해결할 수 있다.

```java
ProcessBuilder builder = new ProcessBuilder(command);
Process process = builder.start();

BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));
String line;
while ((line = reader.readLine()) != null) {
    log.info(line);
}

process.getErrorStream().close(); // 추가
process.getInputStream().close(); // 추가
process.getOutputStream().close(); // 추가

int exitCode = process.waitFor();
```

# 정리
1. ProcessBuilder 클래스를 사용할 때, 입력스트림과 출력스트림을 닫는 것을 잊지 말아야 한다.
2. 프로세스가 입력스트림, 또는 출력스트림을 사용할 때 실패가 발생하여 Deadlock에 빠질 수 있다.

이번 포스팅은 개인적으로 조금 더 공부가 필요해보인다. 

왜 입력스트림과 출력스트림을 사용할 때 Deadlock이 발생하는지 아직 명확하게 설명할 수 없는 것 같다.

조금더 면밀하게 공부해봐야겠다.

출처로 남긴 Naver D2 글에서도 나와있듯이, Java에서 외부 프로세스를 실행하는 기능은 주의해서 다루지 않으면 애플리케이션을 불안정하게 만들 수 있다.

지금은 프로젝트에 당장 넣어야 하는 기능이기 때문에 바로 적용했지만, 추후에 더 정확하게 공부해봐야겠다.

> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
