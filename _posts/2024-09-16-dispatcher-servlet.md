---
title: 디스패처 서블릿. 코드와 함께 이해하기
date: 2024-09-16 10:19:10 +/-0800
categories: [Tech Notes, Spring]
tags: spring    # TAG names should always be lowercase
---

클라이언트에서 처음 Swagger 페이지를 GET 요청했을 때 흐름을 적었습니다.

## DispatcherServlet 요청 흐름

1. 클라이언트의 요청은 서블릿 컨테이너에서 필터들을 지나 스프링 컨텍스트내 `DispatcherServlet`이 가장 먼저 요청을 받습니다.
2. `DispatcherServlet`의 `doService` 메소드에서 웹 요청 처리가 진행됩니다. 

이때 `HttpServletRequest`에 여러 속성들을 넣는 작업을 한 후, `doDispatch()`를 호출합니다. 
![](/assets/img/dispatcher-servlet/img.png)

3. `doDispatch` 작업에서는 먼저, `getHandler()`를 통해 핸들러를 찾습니다. 

이때 핸들러는 6개의 `HandlerMappings` 중 `RequestMappingHandlerMapping` 객체를 이용하여 요청에 대한 핸들러를 찾습니다. 

![](/assets/img/dispatcher-servlet/img_1.png)

그리고 찾은 핸들러는 다음과 같이 빈을 획득합니다. 빈 말고도 다른 정보도 있긴 하지만, 중요한 정보는 역시나 빈이니 사람들이 핸들러 = 컨트롤러 라고 부르는 것 같습니다.

![](/assets/img/dispatcher-servlet/img_2.png)

찾은 핸들러를 바로 반환하지 않고, `HandlerExecutionChain` 객체에 담아 리턴하는데, 이때 요청에 대한 `interceptor`들이 있다면 함께 담아 리턴합니다.
참고로 인터셉터는 해당 핸들러를 실행하기 전에 해야할 작업들이 있을 때 이용합니다.

![](/assets/img/dispatcher-servlet/img_3.png)

4. `doDispatch()` 내에서 `HandlerAdapter`를 찾는 작업을 합니다.

반환받은 핸들러를 직접 바로 실행할 수 없기 때문에 (이해하려면 어뎁터 패턴, PSA를 생각하면 됩니다.) 핸들러에 적합한 어뎁터를 찾습니다.

이 `/api/docs` 경로를 가진 핸들러를 실행할 어뎁터는 바로 `RequestMappingHandlerAdapter` 입니다.
(참고로 스프링 3.2 이전 버전에서는 `AnnotationMethodHandlerAdapter` 이었지만 deprecated 되었습니다.)

![](/assets/img/dispatcher-servlet/img_4.png)

5. 실행될 `interceptor`들이 있다면 interceptor의 `preHandle` 메소드를 차례로 실행합니다.

![](/assets/img/dispatcher-servlet/img_5.png)

6. `HandlerAdapter`의 `handle()` 가 실행하는데, 이때 핸들러가 실행됩니다. 

`handleInternal` 메서드가 호출됩니다. `handleInternal` 내에서 `invokeHandlerMethod`가 호출됩니다.

`invokeHandlerMethod` 내에서
`ServletInvocableHandlerMethod` 객체(`invocableMethod`)가 생성됩니다.
인자 해석기(`argumentResolvers`)와 반환값 핸들러(`returnValueHandlers`)가 설정됩니다.

`invocableMethod.invokeAndHandle()`이 호출되어 실제 컨트롤러 메서드를 실행합니다.
`ServletInvocableHandlerMethod`의 `invokeAndHandle` 메서드는 리플렉션을 사용하여 실제 컨트롤러 메서드를 호출합니다.

![](/assets/img/dispatcher-servlet/img_6.png)

7. interceptor의 `postHandle` 메소드가 실행됩니다.

![](/assets/img/dispatcher-servlet/img_7.png)

8. `resolveViewName` 메소드는 논리적 뷰 이름을 가지고 해당 View 객체를 반환합니다. 
9. Model 객체의 데이터를 보여주기 위해 해당 View 객체의 render 메소드가 수행됩니다.

## 시퀀스 다이어그램

![크게 보면 다음과 같은 흐름입니다. 사진이 작으니 클릭해주세요](/assets/img/dispatcher-servlet/img_8.png)


