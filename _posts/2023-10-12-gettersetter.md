---
title: '@GetterSetter을 만들어보자'
date: 2023-10-12 15:19:10 +/-0800
categories: [Tech Notes, Java]
tags: java    # TAG names should always be lowercase
---

### 왜 @GetterSetter을 만들어보려 하나요?

어노테이션 프로세서를 공부하면서 직접 Lombok이 가지고 있는 기능인 @Getter와 @Setter을 만들어보고 싶다는 생각이 들었다. 그리고 롬복이 어떤 방식으로 작동을 하는지 궁금증이 생겨 어노테이션을 직접 만들어보면서 동작방식을 이해위해 시작한 것이다. 

### Lombok의 원리가 뭔지 모르면 먼저 이것부터.

Lombok의 어노테이션들 중 @Getter을 예시로 들고 생각해보자. (롬복의 다른 어노테이션을 생각하고 봐도 상관없다)

컴파일타임때만 @Getter을 사용하고 있기 때문에 Javac의 이해가 먼저 필요하다. Javac의 컴파일 과정은 크게 3가지 과정을 거친다. 

![](/assets/img/lombok/image.png)

### 1단계 : 심볼 테이블 구문 분석 및 채우기 

먼저 심볼테이블 파싱은 크게 두가지로 나뉜다. 어휘분석과 구문분석 두 단계로 진행하고 그다음 심볼테이블을 채우는 과정이 있다

<b>어휘분석 과 구문분석</b>

어휘분석은 자바소스코드의 문자를 토큰으로 변환시킨다. 토큰이랑 변수나 키워드 등을 말하는 요소이며 가장 작은 단위인 원자성을 지닌다. 

예를 들어 `public int add(int a,int b){return a+b;}`는 `{public,int,add,int,a,int,b,return,a,+,b}` 총 11개의 토큰을 가지고 있다. 

구문분석은 어휘분석으로 처리된 토큰들을 추상개체트리 즉 AST로 구성할 것이다. 

AST란, 코드의 구문 구조를 설명하는 데 사용되는 트리 표현 방법이며 package,type,modifier(접근제어자),operator(연산자),interface,return(반환 값),코드주석 들은 AST의 노드가 될 토큰들 이다. 

`AST에 대한 더 자세한 설명은 해당 CSDN 블로그에서 볼 수 있다. https://blog.csdn.net/peng425/article/details/102814754`

아까 얻은 토큰들을 AST로 완성시키면 아래 그림과 같다.

![](/assets/img/lombok/image%20copy.png)

구문분석을 통해 만들어진 트리구조는 JCTree로 볼 수가 있다. 그리고 해당 트리의 루트노드는 JCCompliationUnit 이며 모두 JCTree의 구성요소이다. 

![출처:https://juejin.cn/post/6844904082084233223](/assets/img/lombok/image%20copy%202.png)

<b>심볼테이블 채우기</b>

구문분석과 어휘분석을 마친 후 심볼 테이블을 채운다. 롬복과 크게 관련 없으므로 이는 넘어가겠다

### 2단계 : 어노테이션 처리(Annotation Processor)

Lombok 구현 원칙의 핵심이다. 컴파일 중에 Annotation을 처리하는 API를 제공한다. 그게 바로 Annotation Processor이다. 이는 새로운 소스코드를 생성할 수도 있고 구문트리(AST)를 수정할 수 있다(기존 소스코드 수정). 만약 구문트리가 수정되면 구문분석하고 심볼테이블을 채우는 과정을 다시 한다. 이러한 루프는 라운드가 된다.

사실 기존 클래스의 구문트리를 수정한다는 것, 즉 컴파일러 내부를 수정하는 것은 권장되지 않는 방법이다. 그래서 롬복은 해킹이라 부르기도 한다. 하지만 어노테이션 프로세서를 통해 새로운 소스파일을 만드는 것은 허용된다. 이는 javapoet 오픈소스 라이브러리를 사용하면 된다. 다음 링크에서 살펴보면 되겠다. 나는 롬복과 같은 어노테이션을 만드는 것이 목표라서 약간 차이점이 존재한다.

https://github.com/square/javapoet

### 3단계 : 의미론적 분석 및 바이트 코드 생성

구문분석 후 컴파일러는 프로그램 코드의 추상구문트리를 얻게 된다. 구문 트리는 올바르게 구조화된 소스프로그램의 추상화를 나타낼 수 있지만 소스 프로그램이 논리적이라는 것을 보장할 수는 없다. 의미론적 분석의 주요 작업은 소스코드 컨택스트 관련 속성을 검토하는 것이다. 

예를들어 아래의 코드는 AST는 얻게 되지만, 연산상 의미가 잘못되었다. 컴파일러는 컴파일을 실패하고 컴파일 오류를 나타낸다.

```java
int a = 3;
boolean b = false;
int c = a + b;
```

---

### @GetterSetter을 만들어보자

<b>소스코드</b>

https://github.com/minsang-alt/fakeLombok

<b>개발 환경</b>

- Tool : Intellij
- Language : JDK 1.8 (I initially used JDK 17, but the APT manipulation API is not available.)
    - JDK 9부터 Java 모듈 시스템이 도입되면서 내부 API 접근이 제한되었다.
- ProjectName : evoLombok
- build tool : gradle
- dependencies
    - Apache Commons Lang 
    - AutoService Processor
    - javac 라이브러리를 사용할 수 있게 의존성 모듈 추가 (com.sun.tools 패키지가 JDK1.8에는 없기 때문에 직접 추가가 필요하다)

<b>프로젝트 구조</b>

만든 어노테이션이 다른 애플리케이션에서 잘 동작하는 지 확인하기 위해서 Multi Gradle Module 구조를 이용.

<img src="/assets/img/lombok/image%20copy%203.png" width="300" height="300" alt="Redis Enterprise Cluster Architecture">

<b>settings.gradle</b>

![](/assets/img/lombok/image%20copy%204.png)

<b>annotation 모듈</b>

그 다음 AST를 조작하기 위해서 javac가 제공하는 tools 패키지를 사용하기 위해서 의존성 모듈을 추가한다.

![](/assets/img/lombok/image%20copy%205.png)

![](/assets/img/lombok/image%20copy%206.png)

어노테이션 GetterSetter를 추가한다.

![](/assets/img/lombok/image%20copy%207.png)

<b>GetSetProcessor</b>

어노테이션 프로세서를 편하게 구현하고 사용하기 위해서는 AbstractProcessor를 상속받으면 되는데 이로 인한 장점은 process 메서드만 본인에 알맞게 구현하면 된다.

![](/assets/img/lombok/image%20copy%208.png)

이제 AST를 조작하기 위해 알고 있어야 할 클래스들을 먼저 소개하겠다.

---

### JCTree 소개

![](/assets/img/lombok/image%20copy%209.png)

JCTree는 구문 트리 요소의 기본 클래스이며 구문 트리에서 현재 구문 트리 노드(JCTree)의 위치를 ​​나타내는 데 사용되는 중요한 필드 pos를 포함하므로 new 키워드를 직접 사용하여 구문을 생성할 수 없다. 사용하려면 TreeMaker를 사용해야 한다.

JCTree는 추상클래스라는 것을 알 수 있는데 여기서는 JCTree의 여러 하위클래스에 중점을 둔다.

- JCStatement: 구문 트리 노드를 선언하며 공통 하위 클래스는 다음과 같다.
    - JCBlock: 명령문 블록 구문 트리 노드
    - JCReturn: return 문 구문 트리 노드
    - JCClassDecl: 클래스 정의 구문 트리 노드
    - JCVariableDecl: 필드/변수 정의 구문 트리 노드
- JCMethodDecl: 메소드 정의 구문 트리 노드
- JCModifiers: 접근제어자 구문 트리 노드
- JCExpression: 표현식 구문 트리 노드, 공통 하위 클래스는 다음과 같다.
    - JCAssign: 할당문 구문 트리 노드
    - JCIdent: 변수, 유형, 키워드 등이 될 수 있는 식별자 구문 트리 노드이다.

---

### TreeMaker 소개

TreeMaker은 일련의 구문트리 노드를 생성하는데 사용된다. 위에서 말했듯이 JCTree 생성은 new 키워드를 사용해서 직접 생성할 수 없
으므로 Java는 JCTree객체를 설정하는 TreeMaker 도구를 제공한다.

구현에 사용되는 핵심 몇가지 클래스를 소개한다.

<b>TreeMaker.Modifier</b>

이 메서드는 JCModifiers(접근제어자 구문트리 노드)를 생성하는 데 사용되며 소스코드는 다음과 같다.

![](/assets/img/lombok/image%20copy%2010.png)

보면 flags 변수에 값을 대입하는 데 이는 Flags 열거형 클래스로 표현할 수 있다. (너무 오래전에 만들어진 클래스라서 Enum을 이용하지 않은 듯 하다)

![](/assets/img/lombok/image%20copy%2011.png)

<b>TreeMaker.MethodDef</b>

TreeMaker.MethodDef는 메소드 정의 구문 트리 노드(JCMethodDecl)를 생성하는 데 사용되며 소스 코드는 다음과 같다.

![](/assets/img/lombok/image%20copy%2012.png)

- JCModifiers : 접근 제어자
- Name: 메서드 이름
- JCExpression: 반환 유형
- JCTypeParameter: 일반 매개변수 목록
- JCVariableDecl: 필드/변수 정의
- JCExpression: 예외 선언 목록
- JCBlock: 명령문 블록
- JCExpression 

더 자세한 내용은 [여기서](https://juejin.cn/post/6844904103542456327?searchId=202310121313184656DFDC4C92A7307061) 볼 수 있다. 다음은 이 JCTree와 TreeMaker 클래스를 이용하여 setter 기능을 할수 있는 AST를 만들어보겠다.

![](/assets/img/lombok/image%20copy%2013.png)

getter 기능을 가진 AST도 비슷한 방법으로 만들 수 있다. 

![](/assets/img/lombok/image%20copy%2014.png)

만들어진 트리를 기존 클래스의 AST 트리에 연결해줘야 한다. 이는 다음 아래와 같이 하면된다.

![](/assets/img/lombok/image%20copy%2015.png)

나머지 AST를 순회하고 기존 AST 트리를 수정하는 부분은 다른 어노테이션을 만들어도 중복되는 부분이다. 그래서 TreeEdit로 공통 클래스로 만들었다.

---

### 만든 어노테이션 테스트

<b>app모듈의 build.gradle</b>

annotation모듈에 있는 어노테이션프로세서를 사용하기 위해 의존성 모듈을 추가하였다.

![](/assets/img/lombok/image%20copy%2016.png)

<b>annotation 모듈 빌드</b>

`./gradlew build clean` 을 통해 빌드를 한다.

<img src="/assets/img/lombok/image%20copy%2017.png" width="200" height="200" alt="Redis Enterprise Cluster Architecture">

이걸 잠깐 zip 파일로 확장자를 바꾸고 열면 META_INF > services 경로에 javax.annotation.processing.Processor 파일이 생성됨을 확인할 수 있다. 

그 파일을 열면 me.evo.processor.GetSetProcessor 이 적혀있는데 이는 만든 어노테이션 프로세서 클래스의 경로이다. 이것이 있어야만 다른 프로젝트에서 해당 파일을 컴파일시점에 찾아가서 연 다음 실행시킬 수 있다.

<b>Member 클래스 생성</b>

```java
@GetterSetter
public class Member{
    private String name;
    private int phone_num;
}
```

<b>Member get,set 메서드 테스트</b>

![](/assets/img/lombok/image%20copy%2018.png)

어노테이션을 붙였지만 빨간 글씨로 표시된다. 하지만 테스트를 진행하면 문제 없이 통과한다.

![](/assets/img/lombok/image%20copy%2019.png)

이는 인텔리제이가 미리 확인을 하지 못하기 때문인데 실제 롬복같은 경우에는 인텔리제이에 플러그인이 있어서 컴파일 하기전에도 체크를 할 수가 있다. 롬복 역시 플러그인이 없으면 방금 만든 것처럼 빨간줄이 나타난다.

### 소감

처음에는 공부 목적으로 어떤 방식인지 알기 위해 시작했다.(맨 위에 내가 쓴 것처럼)

그런데 하다보니 많은 장애물이 존재하였다. 해당 부분은 자바에서는 쓰라고 준 클래스가 아니라서 API라 규정하지도 않는다. 따라서 이에 대한 자세한 설명이 자바 doc 어디에도 없다. 또한 한국에는 자료가 거의 없어서(지금까지 두개 발견해서 많은 도움을 받았다) 겨우 해결했다. 그리고 생각보다 너무 오래걸렸으며 앞으로 이 기술이 프로젝트에 쓸 이유가 없기 때문에 멈춰야 했지만 이미 공부한게 너무 아까워 끝까지 해봤다.



컴파일이 더 세분화돼서 나뉘고 있다는 점. 그리고 어노테이션 프로세서의 동작 방식이 이번 공부의 큰 수확으로 생각된다. 

