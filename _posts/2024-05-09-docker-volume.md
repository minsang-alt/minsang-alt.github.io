---
title: docker volume 제대로 설정하기
date: 2024-05-09 15:19:10 +/-0800
categories: [Project Insights, AgileHub]
tags: project    # TAG names should always be lowercase
---

### 문제 사항

애플리케이션 로그를 프롬테일을 이용해 로그들을 로키로 push 하는 과정에서 nginx와 system 로그는 제대로 로키가 받아오지만 애플리케이션 로그(ERROR 레벨)는 받지 못하는 상황이 발생했다. 

![](/assets/img/docker-volume/image.png)

받아오지를 못하고 있다. 먼저 프롬테일 config.yaml은 다음과 같이 작성했다.

![](/assets/img/docker-volume/image%20copy.png)

### 문제 해결 시도

<b>1. 로그 확인</b>

먼저, 애플리케이션이 있는 서버 안의 프롬테일을 컨테이너로 띄었는데 이를 로그로 확인해봤다. 하지만 잘 로그 경로를 잘 찾아가서 탐지하고 있었다.

그렇다면, 모니터링 서버에 띄어놓은 로키의 컨테이너 로그를 확인해봤다.

![](/assets/img/docker-volume/image%20copy%202.png)

대충 로그를 보면 200 status, 즉 프롬테일이 push 한 API를 로키가 잘 응답받아왔다는 것을 알 수 있다. 하지만 분명히 prod-log에 에러로그가 있음에도 불구하고 0B 즉, 받아온  데이터가 아예 없음을 알 수가 있다.

여기까지 파악할 수 있는 것은 로키와 프롬테일 사이에는 문제가 없지만 데이터가 있는 경로를 잘못찾지 않았을까..? 라는 추측이 생겼다.

<b>2. 프롬테일 컨테이너의 볼륨 경로 확인</b>

프롬테일의 이미지를 컨테이너 실행시켰을 때 다음과 같이 실행했다.

![](/assets/img/docker-volume/image%20copy%203.png)

빨간색으로 박스 친 볼륨 경로를 주목하자. 프롬테일의 /var/log 경로를 호스트서버로 로그있는 경로에 마운트 시키는 부분이다.

그런데 호스트 서버의 /var/log에는 맨 위 프롬테일 yml 사진을 보면 알겠지만 nginx 로그의 /var/log/nginx~ , 시스템 로그의 /var/log/~ 의 경로는 제대로 잡혔지만 애플리케이션 로그는 /var/lib/~ 경로에 있었기 때문에 프롬테일이 접근할 수 없었던 것이다.. 이게 로그에 정확히 나와있지 않아서 깨닫느라 오래걸렸다.

### 문제 해결

<b>1. 프롬테일의 볼륨 경로 수정(실패)</b>

![](/assets/img/docker-volume/image%20copy%204.png)

/var/log가 아닌 /var로 마운트 시키면 /var/lib도 접근할 수 있지 않을까 하고 이렇게 수정하고 다시 컨테이너를 재실행 시켜봤다.
그런데 이번엔 nginx로그조차 그라파나에 나타나지 않았다. 프롬테일 로그를 살펴보니 다음과 같았다.

![](/assets/img/docker-volume/image%20copy%205.png)

/var/log/log 즉, 프롬테일은 /var/log로 접근하고나서 프롬테일의 설정 경로로 가기 때문에 /var/log/log로 간다. 그렇기 때문에 완전 잘못 찍히는 것이다. 따라서 프롬테일 볼륨 마운트 경로는 /var/log로 다시 원상복구 시켰다.

<b>2. 프롬테일이 아닌 애플리케이션 볼륨 경로를 수정(해결)</b>

이번엔 프롬테일의 볼륨경로를 바꾸는 것보다 /var/lib에 속하고 있는 애플리케이션 로그를 /var/log로 바꾸는게 더 낫겠다는 판단이 생겼다.

애플리케이션 볼륨이 처음에는 다음과 같았다. prod-vol:/app 으로 생성한 도커볼륨을 앞에 설정했기 때문에 로그 경로가 

`/var/lib/docker/volumes/prod-vol/_data/logs/agilehub-prod.log`에 속했다. 

![](/assets/img/docker-volume/image%20copy%206.png)

이를 다음과 같이 볼륨 경로를 수정했다. (애플리케이션의 로그가 있는 파일을 /var/log/backend에 마운트 경로로 주었다.)

![](/assets/img/docker-volume/image%20copy%207.png)

컨테이너를 재실행하고 docker inspect로 마운트가 제대로 됐는지 확인한다.

![](/assets/img/docker-volume/image%20copy%208.png)

프롬테일 config 파일은 이제 /var/log/backend/agilehub-prod.log로 경로를 수정하면 된다.

![](/assets/img/docker-volume/image%20copy%209.png)

그러고 프롬테일 컨테이너를 재실행하고나서 그라파나에 다시 접속해보았다.

![](/assets/img/docker-volume/image%20copy%2010.png)

제대로 로그를 받고 있음을 확인할 수 있다.
