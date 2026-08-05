---
title: Servlet & JSP
category: tech
tags: [Servlet, JSP, Java EE, Tomcat, 웹개발]
created: 2026-05-23
updated: 2026-05-23
sources: [Servlet_JSP_강좌정리.txt]
---

# Servlet & JSP

## 한 줄 요약
Java EE 웹 컴포넌트. Servlet은 Java 코드 중심(로직), JSP는 HTML 중심(화면) — 함께 MVC 패턴을 구성한다.

---

## 웹 아키텍처 기본

```
클라이언트(브라우저)          서버(Tomcat 컨테이너)
       요청(HTTP Request) →
       ← 응답(HTTP Response)
```

- **정적 컴포넌트** (HTML): 항상 같은 화면 반환
- **동적 컴포넌트** (Servlet/JSP): 실행 결과를 HTML로 생성해서 반환

---

## Servlet 특징

- `.java` 파일, `src/main/java` 에 저장, 반드시 **패키지 필수**
- `main()` 없음 → Tomcat이 생성/소멸/호출 관리 (직접 new 불가)
- `extends HttpServlet` 필수
- `doGet()` 또는 `doPost()` 재정의

### 계층 구조
```
Servlet (인터페이스)
    └── GenericServlet (추상 클래스)
            └── HttpServlet (추상 클래스)
                    └── 내가 만든 Servlet
```

---

## Servlet 기본 구조

```java
@WebServlet("/hello")    // URL 매핑 (/ 필수)
public class HelloServlet extends HttpServlet {

    @Override
    protected void doGet(HttpServletRequest request,
                         HttpServletResponse response)
                         throws ServletException, IOException {

        // 응답 설정
        response.setContentType("text/html;charset=UTF-8");
        PrintWriter out = response.getWriter();

        // HTML 출력
        out.println("<html><body>");
        out.println("<h1>Hello World</h1>");
        out.println("</body></html>");
    }
}
```

---

## Servlet LifeCycle

| 시점 | 메서드 | 호출 횟수 |
|------|--------|----------|
| 최초 생성 | `init()` | **1번** |
| 요청마다 | `doGet()` / `doPost()` | **매번** |
| 서버 종료 | `destroy()` | **1번** |

> 서블릿은 **단 한 번만 생성**되고 여러 요청을 처리한다.  
> → 인스턴스 변수는 모든 사용자가 공유 (thread-unsafe)  
> → 사용자별 독립 데이터는 **로컬 변수** 사용 필수

---

## 요청 처리 (HttpServletRequest)

```java
// form 데이터 읽기
String name = request.getParameter("username");   // 단일 값
String[] hobbies = request.getParameterValues("hobby");  // 다중 값

// URL 정보
String method = request.getMethod();              // "GET" or "POST"
String contextPath = request.getContextPath();    // "/앱이름"
```

### GET vs POST

| | GET | POST |
|--|-----|------|
| 데이터 위치 | URL 뒤 (`?key=value`) | HTTP Body |
| 보안 | 노출됨 | 숨겨짐 |
| 용도 | 조회 | 등록/수정/삭제 |
| 한글 처리 | URL 인코딩 필요 | request.setCharacterEncoding 필요 |

```java
// POST 한글 처리
request.setCharacterEncoding("UTF-8");
```

---

## Scope (데이터 공유 범위)

| Scope | 객체 | 범위 |
|-------|------|------|
| Page | `pageContext` | 현재 JSP/Servlet |
| Request | `request` | 하나의 요청~응답 |
| Session | `session` | 브라우저 세션 유지 중 |
| Application | `application` (ServletContext) | 서버 전체 |

```java
// Request scope (forward 시 데이터 전달)
request.setAttribute("name", "홍길동");
String name = (String) request.getAttribute("name");

// Session scope (로그인 유지 등)
HttpSession session = request.getSession();
session.setAttribute("loginUser", userDto);
session.invalidate();   // 로그아웃
```

---

## forward vs redirect

```java
// forward: 서버 내부 이동 (URL 안 바뀜, request 공유)
RequestDispatcher rd = request.getRequestDispatcher("/result.jsp");
rd.forward(request, response);

// redirect: 클라이언트에게 새 요청 지시 (URL 바뀜, request 새로 생성)
response.sendRedirect("/main");
```

> **언제 뭘 쓰나?**  
> - 데이터를 JSP로 전달할 때 → `forward`  
> - 중복 제출 방지 (POST 후) → `redirect` (PRG 패턴)

---

## JSP (Java Server Pages)

### JSP 처리 흐름
```
hello.jsp → (변환) hello_jsp.java → (컴파일) hello_jsp.class → HTML 출력
```

JSP는 결국 Servlet으로 변환됨. HTML 안에 자바 코드를 넣는 방식.

### JSP 기본 문법
```jsp
<%@ page contentType="text/html; charset=UTF-8" %>

<!-- 스크립틀릿 (자바 코드) -->
<% int sum = 0; for (int i=1; i<=10; i++) sum += i; %>

<!-- 표현식 (출력) -->
결과: <%= sum %>

<!-- EL (Expression Language, 더 간단) -->
이름: ${name}
리스트: ${list[0]}
객체: ${user.email}

<!-- JSTL -->
<%@ taglib prefix="c" uri="http://java.sun.com/jsp/jstl/core" %>
<c:if test="${not empty list}">
    <c:forEach var="item" items="${list}">
        <p>${item.name}</p>
    </c:forEach>
</c:if>
```

---

## MVC 패턴 (Servlet + JSP 조합)

```
사용자 → Servlet (Controller) → 비즈니스 로직 처리
                               → request.setAttribute(데이터)
                               → forward → JSP (View) → HTML 응답
```

| 역할 | 담당 |
|------|------|
| Controller | Servlet (요청 처리, 로직 호출) |
| Model | Service + DAO + DTO |
| View | JSP (화면 표현만) |

---

## 웹 애플리케이션 구조

```
프로젝트명/
  src/main/
    java/
      com/servlet/...   # Servlet 클래스
      com/service/...   # 비즈니스 로직
      com/dao/...       # DB 연동
      com/dto/...       # 데이터 전달 객체
    webapp/
      WEB-INF/          # 외부 직접 접근 불가 (보안)
        web.xml
        lib/            # jar 파일
      *.jsp             # 뷰 파일
      *.css, *.js
```

---

## 관련 개념
- [[SpringBoot]] — Servlet/JSP를 추상화한 웹 프레임워크
- [[JDBC]] / [[MyBatis]] — Servlet에서 DB 연동
- [[Transaction]] — 서비스 레이어에서의 트랜잭션 처리
