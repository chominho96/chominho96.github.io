---
title: "Amazon Linux에서 opencv 설치"
excerpt: "AWS를 이용한 서버 구축"

categories:
  - aws

toc: true
toc_sticky: true

date: 2022-11-25
last_modified_at: 2022-11-25
---

# ⚡ 문제 발생

이전 글에서도 어느정도 이어지는 내용인 것 같다.

이번 글은 기술적인 부분이라기 보단, 다음에 opencv를 활용하게 될 때 기억하기 위해서 글을 작성하는 느낌이 강하다.

Amazon Linux에 Python과 opencv를 사용해야할 일이 있었고, 자연스럽게 Amazon Linux에 Python과 opencv를 설치하려 했다.

하지만 항상 리눅스는 설치할때부터 발목 잡히는 경우가 많은 것 같다.

Linux의 구현체에 따라 설치하는 명령어도 다르고, 심지어 opencv를 활용한 파이썬 코드를 실행할 때 사용하는 명령어도 다르다.

물론 내가 파이썬을 잘 몰라서 그러는 것도 있겠지만, 어쨌든 이번 기회에 설치하는 법을 정리해보려 한다.


# 😊 해결

먼저 ubuntu에서는 apt-get 명령어를 사용하는 것으로 알고 있었는데, Amazon Linux에서는 yum 명령어를 사용한다.

따라서 python-pip를 설치하는 명령어를 예시로 들면 이러한 형태를 사용한다.

```shell
$ sudo yum -y install python-pip
```

이때 -y 옵션은 install 과정에서 y/n을 물어보는 과정을 스킵하게 해준다.

두 번째로 python-pip가 설치되었으면, numpy와 opencv를 설치해야 한다.

그런데 Amazon Linux에는 Python 2와 Python 3과 함께 설치되어 있다.

따라서 Python 3 용을 설치한다고 명시해줘야 하는데, 이는 다음과 같이 실행하면 된다.

```
$ sudo pip3 install numpy
  # sudo pip install numpy 라고 해도 설치가 되는데 나중에 파이썬 파일 실행시 오류

$ sudo pip3 install opencv-python
  # sudo pip install opencv-python 라고 해도 설치가 되는데 나중에 파이썬 파일 실행시 오류
```

따라서 최종적으로 python과 opencv를 설치하는 명령어들은 다음과 같다.
```
$ sudo yum -y install python-pip
$ sudo pip3 install numpy
$ sudo yum -y install  opencv-python
$ sudo pip3 install opencv-python
```

또한 추가로 파이썬 코드를 실행할 때에도 opencv를 이용할 것이라면 이렇게 실행해야 한다.
```
$ python3 test.py
  # python test.py 오류!
```


# 정리
1. Linux 운영체제 별로 설치하는 명령어가 모두 다르기 때문에 잘 살펴보고 작성해야 한다.
2. 파이썬 버전에 따라 opencv import에서 오류가 발생하므로, 잘 살펴보고 작성해야 한다.

되게 별 것 아니라고 생각한 곳에서 시간을 많이 허비해서 화도 났지만, 그 과정에서 자연스럽게 EC2와 Linux에 친숙해지는 모습을 보면서 마냥 시간을 허비하기만 하진 않았구나 싶은 경험이었다.


> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
