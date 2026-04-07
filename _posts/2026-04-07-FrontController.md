---
layout: post
title: "[Spring] Front Controller로 이해하는 MVC 구조"
date: 2026-04-07
categories: [Spring]
---

Servlet, JSP, MVC 패턴, Front Controller 패턴 정리

---
## 1. Servlet

웹 기능을 서블릿으로 구현해보면 코드가 점점 복잡하고 비효율적임을 느낀다. <br>
예를 들어 회원 등록 폼을 보여주는 기능을 만든다고 하면 다음과 같이 자바 코드 안에서 HTML을 작성해야 한다.

```java
PrintWriter w = response.getWriter();
w.write("<html>");
w.write("<body>");
w.write("<form action=\"/servlet/members/save\" method=\"post\">");
w.write("username: <input type=\"text\" name=\"username\" />");
w.write("age: <input type=\"text\" name=\"age\" />");
w.write("<button type=\"submit\">전송</button>");
w.write("</form>");
w.write("</body>");
w.write("</html>");
```

이 방식으로 동적인 HTML을 만들 수 있지만 HTML 구조를 파악하기 어렵고 코드가 길어질수록 가독성이 떨어진다. <br>
또한 회원 저장과 같은 비즈니스 로직이 하나의 서블릿에 들어가면서 하나의 파일이 너무 많은 역할을 담당하게 된다. 

## 2. JSP

이 문제를 JSP를 사용하여 해결할 수 있다. JSP는 HTML과 자바 코드를 함께 사용할 수 있어 서블릿으로 작성했던 기능을 JSP로 작성하면 다음과 같이 간결해진다.

```html
<form action="/jsp/members/save.jsp" method="post">
    username: <input type="text" name="username" />
    age: <input type="text" name="age" />
    <button type="submit">전송</button>
</form>
```
그리고 비즈니스 로직을 JSP 내부에서 수행할 수 있다.

```jsp
<%
String username = request.getParameter("username");
int age = Integer.parseInt(request.getParameter("age"));
Member member = new Member(username, age);
memberRepository.save(member);
%>
```

이처럼 HTML과 자바 코드를 분리하여 작성할 수 있기 때문에 서블릿에서의 문제를 개선할 수 있다. <br>
하지만 비즈니스 로직과 화면 출력 코드가 하나의 JSP 파일 안에 존재하기 때문에 구조적인 문제는 여전히 해결되지 않는다.

## 3. MVC 패턴

이러한 한계를 MVC 패턴을 적용하여 해결할 수 있다. <br>
MVC는 Model, View, Controller로 역할을 분리하는 구조이다. 컨트롤러는 요청을 처리하고 모델은 데이터를 담고 뷰는 화면을 출력하는 역할을 담당한다.

예를 들어 앞선 기능을 MVC 구조로 변경하면 다음과 같다. 컨트롤러에서는 요청을 받아 비즈니스 로직을 처리하고 그 결과를 모델에 담은 뒤 뷰로 전달한다.

```java
String username = request.getParameter("username");
int age = Integer.parseInt(request.getParameter("age"));

Member member = new Member(username, age);
memberRepository.save(member);

request.setAttribute("member", member);

RequestDispatcher dispatcher =
        request.getRequestDispatcher("/WEB-INF/views/save-result.jsp");
dispatcher.forward(request, response);
```

그리고 JSP에서는 비즈니스 로직을 처리하지 않고 전달받은 데이터를 이용해 화면을 출력하는 역할만 수행한다.

```html
<ul>
    <li>id=${member.id}</li>
    <li>username=${member.username}</li>
    <li>age=${member.age}</li>
</ul>
```
이 구조의 핵심은 비즈니스 로직이 JSP에서 제거되었다는 점이다. JSP는 데이터를 출력하는 역할만 담당하고 비즈니스 로직과 요청 처리는 컨트롤러에서 이루어진다.
요청 흐름 역시 명확해진다. 사용자의 요청은 컨트롤러로 전달되고 컨트롤러는 필요한 데이터를 모델에 담은 뒤 forward를 통해 뷰로 넘긴다. <br>
(이때 forward는 서버 내부에서 처리되기 때문에 URL이 변경되지 않는다.)

하지만 MVC 패턴도 완벽하다고 보기는 어렵다. 컨트롤러를 작성하다 보면 다음과 같은 코드가 계속 반복된는 것을 볼 수 있다.

```java
RequestDispatcher dispatcher =
    request.getRequestDispatcher("/WEB-INF/views/members.jsp");
dispatcher.forward(request, response);
```
이처럼 뷰로 이동하는 코드가 매번 반복며 공통으로 처리해야 할 로직을 모든 컨트롤러에서 작성해야 하는 문제도 발생한다.

## 4. Front Controller 패턴
MVC 패턴을 적용하면 역할은 분리되지만 여전히 반복되는 문제가 존재한다. 
특히 모든 컨트롤러에서 요청 처리 흐름과 관련된 공통 코드가 계속 반복된다. 

이 문제를 해결하기 위한 방식이 바로 Front Controller 패턴이다. 
Front Controller 패턴은 모든 요청을 하나의 진입점에서 처리하고 흐름을 제어하는 구조이다.

<p align="center">
  <img src="/assets/images/mvc_flow.png" width="600"/>
  <br>
  <em>Front Controller 기반 MVC 요청 흐름</em>
</p>

### 4.1 Front Controller 진입
```java
@Override
protected void service(HttpServletRequest request, HttpServletResponse response) {
    Object handler = getHandler(request);
}
```
클라이언트의 요청은 가장 먼저 Front Controller로 전달된다. <br>
이 구조에서는 개별 컨트롤러가 직접 요청을 받지 않고 모든 요청을 하나의 진입점에서 처리한다.

이를 통해 요청 처리 흐름이 한 곳으로 모이게 되고 공통 로직을 중앙에서 관리할 수 있게 된다.

### 4.2 컨트롤러 조회
```java
private Object getHandler(HttpServletRequest request) {
    String requestURI = request.getRequestURI();
    return handlerMappingMap.get(requestURI);
}
```
Front Controller는 요청 URL을 기반으로 어떤 컨트롤러를 실행할지 결정한다.
이를 위해 URL과 컨트롤러를 매핑해둔 handlerMappingMap을 사용한다.

이 과정을 통해 컨트롤러를 직접 호출하지 않고 “어떤 컨트롤러를 실행할지”를 결정한다.

### 4.3 HandlerAdapter 선택
```java
private MyHandlerAdapter getHandlerAdapter(Object handler) {
    for (MyHandlerAdapter adapter : handlerAdapters) {
        if (adapter.supports(handler)) {
            return adapter;
        }
    }
    throw new IllegalArgumentException("adapter not found");
}
```
컨트롤러의 구조가 하나로 고정되어 있지 않기 때문에 찾은 컨트롤러를 처리할 수 있는 HandlerAdapter를 선택한다.

예를 들어, 
- 컨트롤러A는 ModelView 반환
- 컨트롤러B는 viewName 반환 <br>

이처럼 호출 방식이 서로 다르기 때문에 직접 호출하는 대신 Adapter를 통해 실행하도록 한다.


### 4.4 Adapter 내부 로직
```java
ControllerV4 controller = (ControllerV4) handler;

Map<String, String> paramMap = createParamMap(request);
Map<String, Object> model = new HashMap<>();

String viewName = controller.process(paramMap, model);

ModelView mv = new ModelView(viewName);
mv.setModel(model);
```
HandlerAdapter는 실제 컨트롤러를 호출하는 역할을 수행한다.

이 단계의 핵심은 컨트롤러마다 다른 호출 방식을 하나의 형태로 통일하는 것이다.

### 4.5 ViewResolver
```java
MyView view = viewResolver(mv.getViewName());

private MyView viewResolver(String viewName) {
    return new MyView("/WEB-INF/views/" + viewName + ".jsp");
}
```
컨트롤러는 실제 경로가 아닌 논리적인 뷰 이름만 반환한다.
ViewResolver는 이 이름을 실제 물리 경로로 변환하는 역할을 한다.

따라서 뷰의 위치가 변경되더라도 ViewResolver만 수정하면 되며 컨트롤러 코드는 그대로 유지할 수 있다.

### 4.6 View 렌더링
```java
view.render(mv.getModel(), request, response);

private void modelToRequestAttribute(Map<String, Object> model, HttpServletRequest request) {
    model.forEach((key, value) -> request.setAttribute(key, value));
}
```
마지막 단계에서는 ModelView에 담긴 데이터를 기반으로 화면을 렌더링한다.
model 데이터를 request에 저장한 뒤 JSP로 forward를 수행한다.

JSP는 request에 저장된 데이터를 이용해 화면을 구성하며 이 과정을 통해 최종적으로 사용자에게 응답이 전달된다.

---

< 맥북 기준 유용한 단축키...> <br>
Option + Enter: Quick Fix <br>
Ctrl + T: Refactor This <br>
Cmd + Option + V: Extract Variable <br>
Cmd + Option + B: Go to Implementation <br>
Cmd + Option + M: Extract Method 

---
