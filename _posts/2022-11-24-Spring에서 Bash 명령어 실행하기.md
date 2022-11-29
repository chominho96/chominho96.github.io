---
title: "Spring에서 Bash 명령어 실행하기"
excerpt: "Spring Boot를 이용한 백엔드 개발"

categories:
  - spring

toc: true
toc_sticky: true

date: 2022-11-24
last_modified_at: 2022-11-24
---

# ⚡ 문제 발생

이번에 겪은 문제는 사실 에러를 잡거나 하는 것이 아닌, 새로운 상황을 맞이한 경우였다.

진행하고 있는 프로젝트에서 내부 로직 처리를 할 때, Python과 OpenCV로 이루어진 .py 파일을 실행해야 하는 일이 있었다.

이런 경우는 생각보다 흔한 것 같았는데, 크롤링이나 이미지 처리 등에서는 파이썬으로 작업하는 경우가 많기 때문이었다.

구글링을 했을 때도 Spring과 파이썬을 같이 사용해서 프로젝트를 풀어나간 사례를 많이 봤다.

Spring (Java)에서 Python 코드를 실행하는 대표적인 방법은 다음과 같다.

  1. Jython 사용
  2. Spring에서 Bash 명령어로 Python 코드를 실행

먼저 Jython의 경우, 자바 코드로 파이썬 코드를 실행할 수 있는 방법이다 보니, 제일 먼저 내가 고려한 방식이다.

하지만 조금더 구글링한 결과, 치명적인 한계점을 발견했다.

Jython은 Numpy 등 파이썬의 외부 라이브러리를 지원하지 않는다는 점이었다. 현재 내 프로젝트는 Numpy, OpenCV 등 외부 라이브러리를 이용한 이미지 인식, 처리 로직을 담당하고 있기 때문에, Jython을 적용하지 못하는 것이었다.

따라서 자연스럽게 2번의 방법으로 구현하는 방식을 택했다.

```
[📌참고] 

사실 어느정도 규모가 있는 서비스에서는 나와 같은 상황에서 이미지 인식/처리를 담당하는 별도의 서버를 따로 둬서 해결한다고 한다. 
또한 반대로 Spring이 아닌 애초에 Python 기반의 Django를 사용하는 것도 방법일 수 있다. 
하지만 내가 해당 파이썬 코드를 담당하지도 않았고, Django를 그 순간에 배워서 적용하기에는 Django를 배워본 적이 없기 때문에 어쩔 수 없이 2번의 방식을 택한 것이다.
```

일단 로컬에서 파이썬 코드를 어떻게 실행하는지에 대해서 익힌 후, 이를 Amazon Linux에서 실행하기 위해 bash 명령어를 호출하는 코드를 Spring 내에 작성하기로 했다.

# 😊 해결 : bash 명령어를 수행하기 위한 수많은 삽질과 해결

수많은 삽질을 반복했는데, 많이 생략하고 본론으로 넘어가고자 한다.

일단 bash 명령어를 수행하는 코드 예제가 구글링 결과 나오긴 하는데, 문제는 사용하는 Linux의 종류에 따라 천차만별의 방식이 존재한다는 것이었다.

먼저 bash 명령어를 수행하는 방식에는 두 가지가 있다.

  1. Runtime 클래스 이용
  2. ProcessBuilder 클래스 이용

(확실하진 않은데) 여기저기 찾아본 결과 버전업이 되면서 Runtime 클래스도 내부적으로 ProcessBuilder 방식으로 구현되도록 변경되었다고 한다.

따라서 ProcessBuilder 방식을 사용하기로 했다.

이제 명령어를 작성해야 하는데, 일반적으로 Putty 등 SSH에 접속하거나 CLI로 직접 리눅스 명령어를 치던 것과 다르게 앞에 내가 입력하는게 명령어다 라는 것을 알려줘야 한다.

여기서 정말 많이 헤맸는데, AWS에서 사용하는 Linux 구현체인 Amazon Linux에서는 다음과 같이 입력하면 된다.

``` 
$ /bin/sh -c 명령어
```

따라서 Spring 상에서 EC2 상으로 bash 명령어 수행을 요청하는 예시 코드는 다음과 같다.

``` java
ProcessBuilder builder = new ProcessBuilder(command);
builder.redirectErrorStream(true);
try {
    Process process = builder.start();

    BufferedReader reader = new BufferedReader(new InputStreamReader(process.getInputStream()));

    String line;
    while ((line = reader.readLine()) != null) {
        log.info(line);
    }

    process.getErrorStream().close();
    process.getInputStream().close();
    process.getOutputStream().close();

    int exitCode = process.waitFor();
    log.info("\nExited with error code : " + exitCode);

} catch (IOException e) {
      e.printStackTrace();
} catch (InterruptedException e) {
      e.printStackTrace();
}
```

이 때 command 변수는 ```List<String>``` 형이다.

해당 리스트의 구조는 다음과 같아야 한다.

```
{ "/bin/sh", "-c", "python test.py" }
```

주의해야할 점이, /bin/sh와 -c는 따로 리스트에 담아두고, 메인 명령어는 이어붙인 String 형태로 붙여서 List에 삽입해야한다는 것이다.

그냥 "/bin/sh -c python test.py" 명령어를 ProcessBuilder의 인자로 넘겨주면 정상적으로 실행되지 않는다.

나는 따라서 각 명령어들의 조각? (python 등)을 상수로 선언해두고, 명령어들의 조각을 넘겨받아 String으로 변환하는 메서드를 생성해서 bash 명령어 실행을 최대한 함수화했다.

또한 마지막으로 한 가지 팁은, Elastic Beanstalk을 사용 중이라면 기본적으로 명령어를 실행할 때 현재 bash 명령어가 실행되고 있는 디렉토리는 /home/ec2-user/ 이다.

따라서 이를 고려해서 /home/ec2-user/ 위치에 실행할 파이썬 파일 등을 넣어두거나, 아니면 경로를 처음부터 마음 편하게 절대경로로 설정하는 방식 등을 사용할 수 있을 것이다.



# 정리
1. Spring에서 bash 명령어를 실행할 때에는 ProcessBuilder 클래스를 사용한다.
2. 명령어를 작성할 때 자신이 사용중인 OS에 따라 명령어를 작성한다.

bash 명령어 실행까진 결국 성공했지만, 아직 한계가 남아 있는 것 같다.

  1. GitHub Action을 통해 CI/CD 구축을 하면 자동 배포가 될때 새로운 EC2 인스턴스를 생성한다. 이때 설치했던 OpenCV, phthon 등을 다시 설치해야 한다.
  2. 실행할 파이썬 코드들을 배포할 때 자동으로 원하는 위치에 넣을 수 있는 방법을 생각해봐야 한다.

물론 이 모든 고민들은 파이썬 코드가 무거워질수록 따로 이미지 인식/변환을 위한 서버를 구축하는 것으로 다 해결할 수 있을 것이다.



> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
