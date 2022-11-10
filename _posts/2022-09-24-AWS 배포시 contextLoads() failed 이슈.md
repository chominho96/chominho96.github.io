---
title: "Spring 프로젝트 AWS 배포 시 contextLoads() 실패 이슈"
excerpt: "Spring Boot & AWS"

categories:
  - aws

toc: true
toc_sticky: true

date: 2022-09-24
last_modified_at: 2022-09-24
---

# ⚡문제 발생

GitHub Action을 이용해서 AWS 엘라스틱 빈스톡에 자동 배포하는 과정을 진행 중 배포 단계에서 문제가 생긴 것을 발견했다.
![제목 없음](https://user-images.githubusercontent.com/66549638/201134438-142ce661-d14c-4adc-89ea-97c6ed7a7ec2.png)

DeVinApplicationTests > contextLoads() FAILED

Spring 상에서 테스트가 실패한 것으로 보였는데, 로컬에서 모든 테스트를 통과했기 때문에, 당황스러웠다...

![broken-in-productionp-works-on-my-machine-problem-what-problem-35901872](https://user-images.githubusercontent.com/66549638/201135239-4dda829c-8375-418b-a21e-6e14ad031aaa.png)

일단 배포 시에 모든 테스트를 통과하지 않으면 배포에 실패한다는 것은 알고 있었기 때문에, 이 테스트가 실패했기 때문에 배포에 실패했다고 판단했다.


# 😊해결

이 에러는 비교적 쉽게 해결할 수 있었는데, contextLoads() 메서드를 실행하는 test 폴더 안에는 main 폴더 안에 있는 application.yml이 존재하지 않기 때문이었다.

![제목 없음2](https://user-images.githubusercontent.com/66549638/201136068-fb3f0c8f-f474-42f6-b1c1-b14823d4b2b6.png)

따라서 이렇게 main에 있는 yml 파일들을 모두 복사하고 진행하려 했는데, 문제가 발생했다.

현재 그림 상으로 선택되어있는 파일, application-API-KEY.properties에는 API Key들을 담아 보관하고 있고, .gitignore에 추가해 github에 업로드되지 않도록 설정했다.

하지만 github에 업로드되어있지 않기 때문에 당연히 GitHub Action으로 배포를 진행할 때 이 파일이 같이 포함될 수가 없고, 동일하게 contextLoads() 함수에서 에러를 일으키는 것이다.

이에 해결방법을 정리해봤는데, 이는 다음과 같다.

1. contextLoads() 메서드를 주석 처리 - 가장 쉬운 방법이지만 결국 추후 테스트 코드 작성시 오류를 일으킬 가능성이 큼
2. application-API-KEY.properties에 들어가는 내용을 EB의 환경 변수로 저장해놓고, application.prod.yml에 환경 변수를 이용해서 저장 - 가장 좋은 방식

따라서 2번의 방식으로 해결하기로 했다. 다만 개발하고 있는 지금 시점에서 아직 모든 API들을 적용하지 못한 상태인데, EB에 환경변수를 외부 API를 도입할 때마다 수정하면 시간이 너무 오래 걸리므로 일단은 application-API-KEY.properties를 남겨두기로 했다.

이후 개발이 어느정도 마무리되면 API Key들을 모두 EB의 환경변수로 넘기고 받아오는 방식으로 전환할 예정이다.


# 정리
<span style="color:yellow">main과 같은 yml 파일을 가지고 있어야 contextLoads()가 정상적으로 수행됨</span>을 알게 되었다.

확실히 서버에 배포하는 과정을 거치면서 로컬에서 개발하는거보다 2~3배는 더 많이 헤매는 것 같다.

앞으로 서버 배포에서 만난 다양한 경험들을 정리하는 습관을 들여야 다시 같은 문제를 만났을 때 헤매지 않을 것 같다.



> ⚡개인적으로 공부하면서 포스팅한 글입니다. 오류가 있는 경우 지적해주시면 감사하겠습니다!⚡
