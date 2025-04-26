---
title: 배포 하는데 걸리던 시간 13분을 5분으로 줄이기
date: 2024-05-14 15:19:10 +/-0800
categories: [Project Insights, AgileHub]
tags: project    # TAG names should always be lowercase
---

### 문제사항

최근 애자일허브 프로젝트는 도커이미지를 만들어 DockerHub에 올리는 방식으로 배포를 하고 있습니다. Dockerfile을 만들어서 GitHub에 올려두고, GitHub Actions로 docker build와 push를 진행하는 방식입니다.

그런데 배포를 할때마다, 매번 빌드 시간이 10분 이상이었고, 코드가 조금만 추가되어도 1분씩 늘어나 최근에는 배포 한번 하는데 13분정도 걸립니다. 이정도의 시간은 매번 배포할때마다 다른 일을 해야하고, 나중에 테스트를 해보며 수정할게 생기면 다시 또 13분을 기다려야 하는 충분히 부담되는 시간입니다. 그리고 이런 사이클은 Continuous Deployment의 장점을 잘 살리지 못한다고 생각했습니다.

### 배경지식 - 도커 레이어와 캐시

도커 빌드 속도에 영향을 미치는 레이어(Layer)와 캐시(Cache)에 대해 알아보겠습니다.

도커 이미지는 빌드 시 Dockerfile의 명령어들을 차례로 실행하면서 레이어를 생성합니다. 이때 명령어(RUN, ADD, COPY)로 생성된 레이어는 이미지 크기를 커지게 하고, 이미지를 생성하는 시간도 길어지게 합니다. 이미지 크기를 줄이는 방법은 다양하게 있는데 뒤에서 알아보겠습니다.

![https://docs.docker.com/build/cache/](/assets/img/github-actions/image.png)

크기를 줄이는 방법 외에도 도커 캐시가 있습니다. Dockerfile을 작성하고, docker build를 실행하게 되면 빌드 속도를 높이기 위해 캐시를 사용합니다. 첫 번째 빌드에서는 각 단계 별 캐시를 설정하고, 이후 동일한 명령어가 실행되면 만들어둔 레이어를 재사용합니다. 만약 레이어가 변경되면 해당 레이어포함 그 뒤 레이어들을 다시 빌드합니다. 

![](/assets/img/github-actions/image%20copy.png)

### 멀티스테이지 빌드

멀티스테이지 빌드란, 최종 이미지에서 필요 없는 환경을 제거할 수 있도록 여러 단계에 걸쳐 이미지를 만드는 방법입니다.

아래의 Dockerfile 예시에서 볼 수 있듯이, 첫 번째 단계에서는 Gradle 8.3 버전과 함께 JDK가 포함된 상태로 애플리케이션이 빌드되어 이미지 크기가 매우 큽니다. 하지만 빌드가 완료된 후에는 JDK가 더 이상 필요하지 않으므로, FROM 명령어를 사용해 eclipse-temurin을 기반으로 한 JRE만 포함된 훨씬 가벼운 레이어로 전환합니다.

![](/assets/img/github-actions/image%20copy%202.png)

위와 같이 멀티스테이지 빌드로 바꾸면 기존 700MB 였던 이미지의 크기가 약 320 MB로 줄어듭니다. 

또한 앞에서 도커는 캐시 전략에 의해 레이어에 변함이 없다면 빌드 속도는 빨라야합니다. 로컬에서 이미지 빌드를 반복한다면 시간이 줄어듬을 확인할 수 있습니다. 

### GitHub Actions를 이용한 빌드

애자일허브는 코드를 GitHub에 올리고, GitHub Actions로 도커 빌드 및 배포를 실행해주고 있습니다. 

![](/assets/img/github-actions/image%20copy%203.png)

위 workflow를 사용해보겠습니다. main 브랜치로 push 할 때마다 docker build를 실행합니다.

![](/assets/img/github-actions/image%20copy%204.png)

동일한 코드는 아니여서 정확히 판단은 되지 않습니다. 도커 캐시를 잘 사용중인지 GitHub Actions 로그를 살펴보겠습니다.

![](/assets/img/github-actions/image%20copy%205.png)

의존성이 바뀌지 않았음에 불구하고 도커의 캐시가 적용되지 않았습니다.
빌드 시간을 보면 의존성만 빌드하는 `RUN gradle dependencies --no-daemon`의 시간은 2m 39s, `RUN gradle clean build --no-daemon`의 빌드 시간은 8m 13s로 매우 많은 시간이 걸렸습니다.

캐시가 동작하지 않는 겁니다.

### GitHub Actions에서 도커 캐싱

GitHub Actions의 러너는 매번 새로운 가상환경에서 실행됩니다. 작업은 매번 새롭게 다시 시작되는거죠. GitHub에서는 캐싱을 제공하지만, Docker 레이어에 대한 내용은 없습니다.

도커에서 공식적으로 제공하는 buildx라는 CLI 플러그인을 사용하면 <b>GitHub Cache API</b>를 활용할 수 있습니다.

![](/assets/img/github-actions/image%20copy%206.png)

우선 buildx 설정을 해주고, `docker/build-push-action@v5`를 이용해 build와 push를 해주고 있습니다.
그리고 `cache-from` 과 `cache-to에 type=gha`라고 입력해줍니다. 이 부분에서 캐싱이 적용됩니다.

이렇게 설정하고 코드를 변경하지 않은 채 배포를 다시 해보고 비교해보겠습니다. 배포를 하면

![](/assets/img/github-actions/image%20copy%207.png)

아직 저장된 캐싱이 없으므로 앞과 거의 비슷하게 12m 13s 정도 걸렸습니다.

다시 배포를 하면 

![](/assets/img/github-actions/image%20copy%208.png)

빌드 시간이 <b>8m 41s</b>로 약 4분정도 단축했습니다!

로그를 다시 한번만 확인해보면

![](/assets/img/github-actions/image%20copy%209.png)

CACHED가 적용된 것을 확인할 수 있습니다.

그러나 로그를 계속 살펴보니 `RUN gradle clean build --no-daemon` 은 여전히 캐싱이 적용되지 않고 7m 이상의 시간이 소요됐습니다. 이 부분도 캐싱이 적용되면 빌드시간이 훨씬 단축될것 같습니다.

![](/assets/img/github-actions/image%20copy%2010.png)

그래서 gradle 빌드하기 전 clean하지 않고 빌드를 하면 캐싱이 적용되지 않을까 했지만 여전히 적용되지 않았습니다. 

(참고로 도커 내 Gradle의 캐싱은 불가능 합니다. Gradle 빌드되면 .gradle 폴더에 Gradle Cache가 남아있지만 Docker가 그 경로를 마운트 하지 않는 이상 찾을 수 없습니다. 그리고 복잡하다고 하네요 (https://discuss.gradle.org/t/why-gradle-does-not-use-cache-in-docker/33902))

![](/assets/img/github-actions/image%20copy%2011.png)

또한 시간이 오래걸리는 이유를 로그로 보니 

![](/assets/img/github-actions/image%20copy%2012.png)

테스트를 하는데에만 3분 넘게 보내고 있었습니다. SpringBootTest 어노테이션을 사용함에 따라 모든 빈을 띄우는 통합테스트이기 때문에 매우 느립니다. 

### Gradle 빌드 옵션 변경

저희 애자일허브는 main브랜치에 push하기 전, PR을 통해 CI 통합테스트를 먼저 진행하고, 리뷰와 모든 테스트가 통과할 시에 main 브랜치에 push할 수 있습니다. 

따라서 빌드 속도를 높이기 위해 CD 워크플로에서는 테스트를 진행하지 않도록 하고, gradle의 병렬 빌드 옵션을 사용하도록 했습니다. (병렬 빌드는 서브모듈이 있을 때 유용한거라 이부분에 대한 시간은 영향이 없을 것 같습니다)

![](/assets/img/github-actions/image%20copy%2013.png)

테스트를 제외하고 배포를 한 결과

![](/assets/img/github-actions/image%20copy%2014.png)

최종적으로, 배포하는데 걸렸던 13분이 5분으로 8분 가량 줄어들었습니다!

![](/assets/img/github-actions/image%20copy%2015.png)

![](/assets/img/github-actions/image%20copy%2016.png)

의존성 라이브러리를 빌드하는 과정에서 CACHED 됨을 확인할 수 있었습니다.


### 참고 자료

- https://docs.gradle.org/8.1/release-notes.html

- https://findstar.pe.kr/2022/05/13/gradle-docker-cache/

- https://fe-developers.kakaoent.com/2022/220414-docker-cache/



