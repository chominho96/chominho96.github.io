---
title: "EC2 내부 파일 접근 시 FileNotFoundException 이슈"
excerpt: "AWS를 이용한 서버 구축"

categories:
  - aws

toc: true
toc_sticky: true

date: 2022-11-25
last_modified_at: 2022-11-25
---

# ⚡ 문제 발생

프로젝트를 진행하면서 S3에 파일을 저장하는 게 아닌, EC2 내부에 임시로 파일을 저장해야할 일이 생겼다.

따라서 구글링을 하면서 서버에 파일을 저장하는 코드를 작성하였다.

이후 코드를 실행했는데 FileNotFoundException (Permission Denied)가 뜨면서 정상적으로 파일 저장도, 접근도 되지 않았다... (에러 로그를 보관했어야 하는데 어디갔는지 보이질 않는다 ㅠㅠ)

대신 내가 참조한 질문의 링크를 남기면, 다음과 같다.

https://stackoverflow.com/questions/34630698/aws-ec2-tomcat-permission-denied-creating-writing-to-file

ec2-user로 SSH 접속을 해서 EC2를 다루다가, 이를 Spring에서 다루려고 하니 생기는 문제가 여간 많은게 아닌 것 같다.

어쩄든 에러 메시지 뜻 그대로 권한이 없는 문제임을 확인헀고, 구글링을 통해 문제점을 발견했다.


# 😊 해결

해당 에러를 왜 내가 직접 SSH로 ec2-user를 통해 접속했을 때는 발견하지 못하고, Spring을 통해 접근하려 할 때 발견하게 된거냐면, Spring을 실행시키는 주체인 Tomcat이 /home/ec2-user/ 디렉토리의 접근 권한이 없기 때문이다.

SSH로 접근했을 때는 현재 사용자가 ec2-user이고, 자신의 디렉토리를 접근하는 것이기 때문에 접근이 가능하지만, tomcat은 other에 속하므로 당연히 접근할 수 없는 것이다.

따라서 /home/ec2-user/에 other 권한을 줘야 하는데, 다음과 같이 설정하면 된다.

```
$ chmod o+x /home
$ chmod o+x /home/ec2-user
```

```
[📌주의]
사실 여기까지 봤을 때 내가 들었던 의문이 읽고, 쓰는 권한이 있어야 하는 것 아닌가? 하는 의문이었다.
그래서 처음에 chmod o+rwx /home 과 같이 권한을 줘봤다.
그런데 그 뒤로 SSH 접속이 영영 되지 않았다(...?)
결국 GitHub Action의 rerun all jobs를 통해 서버를 다시 밀어버리고 해결했다.
이는 위의 Stack Overflow 글에서도 볼 수 있는데, rw 권한을 others에게 열어버리면 SSH가 깨진다고 한다.
아직까지 명확한 이유를 찾진 못했지만, 어쨌든 x 권한만 부여하도록 하자.
```



# 정리
1. Spring이 EC2 내부 디렉토리에 접근하려면, /home, /home/ec2-user 디렉토리에 대해 o+x 권한을 설정해줘야 한다.
2. o+x 이외의 권한을 함부로 건드리지 말자


> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
