---
title: Spring Boot
category: tech
tags: [SpringBoot, Spring, Java, 웹개발, REST API, IoC, DI]
created: 2026-05-23
updated: 2026-05-23
sources: [SpringBoot_강좌정리.txt]
---

# Spring Boot

## 한 줄 요약
Spring Framework의 복잡한 설정을 자동화한 Java 웹 프레임워크. 내장 Tomcat + auto configuration으로 빠르게 시작 가능.

---

## Spring Framework 핵심 개념

### IoC (Inversion of Control: 제어의 역행)
기존: 개발자가 직접 `new`로 객체 생성  
Spring: **IoC Container**가 객체(Bean) 생성 + 관리 + 주입

```java
// 기존 방식
BoardService service = new BoardServiceImpl(new BoardDAO());

// Spring IoC 방식 → Container가 자동으로 생성하고 주입
@Service
public class BoardServiceImpl {
    @Autowired
    private BoardDAO dao;  // IoC Container가 알아서 주입
}
```

### DI (Dependency Injection: 의존성 주입) 3가지 방식

```java
// 1. 생성자 주입 (권장)
@Service
public class BoardService {
    private final BoardDAO dao;

    public BoardService(BoardDAO dao) {
        this.dao = dao;
    }
}

// 2. setter 주입
public void setDao(BoardDAO dao) { this.dao = dao; }

// 3. 필드 주입 (테스트 어렵고 권장 안함)
@Autowired
private BoardDAO dao;
```

### Bean 등록 — @ComponentScan이 자동 탐색
```java
@Component    // 범용
@Controller   // 웹 요청 처리
@Service      // 비즈니스 로직
@Repository   // DB 접근
@RestController // Controller + @ResponseBody
```

---

## Spring Boot 특징

| 특징 | 설명 |
|------|------|
| **Auto Configuration** | 의존성(starter) 보고 설정 자동화 |
| **내장 Tomcat** | 별도 서버 설치 불필요 |
| **Starter** | 관련 jar 묶음 제공 (`spring-boot-starter-web` 등) |
| **application.yml** | 모든 설정을 한 파일에서 관리 |

### 주요 Starter
```gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-thymeleaf'
    implementation 'org.mybatis.spring.boot:mybatis-spring-boot-starter:3.0.3'
    runtimeOnly 'com.mysql:mysql-connector-j'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}
```

---

## 프로젝트 구조

```
src/main/
  java/com/example/
    Application.java         # 시작점 + @SpringBootApplication
    controller/              # 요청 처리
    service/                 # 비즈니스 로직
    repository/ (dao/)       # DB 연동
    dto/                     # 데이터 전달
  resources/
    application.yml          # 설정
    templates/               # Thymeleaf 뷰 (.html)
    static/                  # 정적 파일 (css, js, images)
```

### Application.java
```java
@SpringBootApplication  // @ComponentScan + @EnableAutoConfiguration + @Configuration
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}
```

---

## MVC 웹 개발

### Controller
```java
@Controller
public class BoardController {

    @Autowired
    private BoardService boardService;

    @GetMapping("/board/list")
    public String list(Model model) {
        List<BoardDTO> list = boardService.getList();
        model.addAttribute("list", list);   // View로 데이터 전달
        return "board/list";                // templates/board/list.html
    }

    @PostMapping("/board/insert")
    public String insert(BoardDTO dto) {
        boardService.insert(dto);
        return "redirect:/board/list";      // PRG 패턴
    }
}
```

### Thymeleaf (View 템플릿)
```html
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<body>
    <h1>게시글 목록</h1>
    <table>
        <tr th:each="item : ${list}">
            <td th:text="${item.title}"></td>
            <td th:text="${item.writer}"></td>
        </tr>
    </table>
</body>
</html>
```

---

## REST API

클라이언트(Vue.js, 모바일 등)와 JSON으로 통신하는 방식.

```java
@RestController     // @Controller + @ResponseBody (자동으로 JSON 변환)
@RequestMapping("/api")
public class BoardApiController {

    @GetMapping("/boards")
    public List<BoardDTO> getList() {
        return boardService.getList();   // 자동으로 JSON 배열로 변환
    }

    @GetMapping("/boards/{num}")
    public BoardDTO getOne(@PathVariable int num) {
        return boardService.getOne(num);
    }

    @PostMapping("/boards")
    public ResponseEntity<String> insert(@RequestBody BoardDTO dto) {
        boardService.insert(dto);
        return ResponseEntity.ok("success");
    }

    @DeleteMapping("/boards/{num}")
    public ResponseEntity<String> delete(@PathVariable int num) {
        boardService.delete(num);
        return ResponseEntity.ok("deleted");
    }
}
```

### HTTP Method 매핑
| 어노테이션 | HTTP | 용도 |
|-----------|------|------|
| `@GetMapping` | GET | 조회 |
| `@PostMapping` | POST | 생성 |
| `@PutMapping` | PUT | 전체 수정 |
| `@PatchMapping` | PATCH | 부분 수정 |
| `@DeleteMapping` | DELETE | 삭제 |

---

## 트랜잭션

```java
@Service
@Transactional   // 클래스 전체 적용
public class BoardService {

    @Transactional   // 메서드별 적용 (RuntimeException 발생 시 자동 rollback)
    public void transfer(int from, int to, int amount) {
        dao.withdraw(from, amount);  // 성공
        dao.deposit(to, amount);     // 실패 시 → 자동 rollback
    }
}
```

---

## application.yml 설정

```yaml
server:
  port: 8080

spring:
  datasource:
    url: jdbc:mysql://localhost:3306/testdb?serverTimezone=Asia/Seoul
    username: root
    password: 1234
    driver-class-name: com.mysql.cj.jdbc.Driver

mybatis:
  mapper-locations: classpath:mappers/**/*.xml
  type-aliases-package: com.example.dto
  configuration:
    map-underscore-to-camel-case: true   # DB 컬럼 emp_no → empNo 자동 변환
```

---

## 실행 & 빌드

```bash
./gradlew bootRun           # 개발 실행
./gradlew clean build       # JAR 빌드 (build/libs/*.jar 생성)
java -jar demo-1.0.0.jar   # 빌드된 JAR 실행
```

---

## Spring 생태계

| 프레임워크 | 역할 |
|-----------|------|
| Spring Framework | 핵심 (IoC, DI, AOP, MVC) |
| Spring Boot | 빠른 시작, 설정 자동화 |
| Spring Security | 인증/인가 |
| Spring Data JPA | JPA 편리한 사용 |
| Spring Batch | 대용량 배치 처리 |

---

## 관련 개념
- [[Servlet_JSP]] — Spring MVC가 내부적으로 사용하는 기반 기술
- [[MyBatis]] — SpringBoot와 연동해서 DB 접근
- [[Transaction]] — @Transactional 동작 원리
- [[REST API]] — JSON 기반 API 설계 개념
