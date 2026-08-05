---
title: JDBC
category: tech
tags: [JDBC, Java, 데이터베이스, MySQL]
created: 2026-05-23
updated: 2026-05-23
sources: [Jdbc_강좌정리.txt]
---

# JDBC

## 한 줄 요약
Java와 DBMS를 연결하는 표준 API. 어떤 DBMS든 동일한 코드로 접근 가능 — DBMS 독립적.

---

## 왜 DBMS 독립적인가?

Java 측에서 인터페이스(`Connection`, `PreparedStatement` 등)만 정의하고, 각 DBMS 벤더(Oracle, MySQL 등)가 그 인터페이스를 구현한 드라이버(`.jar`)를 제공한다.

```
자바 프로그램                    DBMS 벤더 드라이버
java.sql.Connection  ←------→  mysql-connector-j.jar
java.sql.PreparedStatement      ojdbc.jar (Oracle)
```

---

## 연동 7단계

```java
// 1. 접속 정보 정의
String driver = "com.mysql.cj.jdbc.Driver";
String url    = "jdbc:mysql://localhost:3306/testdb";
String userid = "root";
String passwd = "1234";

// 2. 드라이버 로딩 (클래스를 메모리에 올림)
Class.forName(driver);   // ClassNotFoundException 발생 가능

// 3. Connection 맺기
Connection con = DriverManager.getConnection(url, userid, passwd);

// 4. SQL 작성 (끝의 ; 제외)
String sql = "SELECT deptno, dname, loc FROM dept";
// String sql = "INSERT INTO dept(deptno, dname, loc) VALUES(?, ?, ?)";

// 5. PreparedStatement 생성
PreparedStatement pstmt = con.prepareStatement(sql);

// 6. SQL 실행
// DML (insert/update/delete)
// int n = pstmt.executeUpdate();   // 변경된 레코드 수 반환

// SELECT
ResultSet rs = pstmt.executeQuery();
while (rs.next()) {              // 행 이동
    int deptno  = rs.getInt("deptno");
    String dname = rs.getString("dname");
    String loc   = rs.getString("loc");
}

// 7. 자원 반납 (사용 역순으로)
rs.close();
pstmt.close();
con.close();
```

> 모든 단계에서 `SQLException` 발생 가능 → `try-catch` 또는 `throws` 필수

---

## PreparedStatement의 ? (바인딩)

```java
String sql = "INSERT INTO dept(deptno, dname, loc) VALUES(?, ?, ?)";
PreparedStatement pstmt = con.prepareStatement(sql);

pstmt.setInt(1, 50);          // 첫 번째 ?
pstmt.setString(2, "인사부");  // 두 번째 ?
pstmt.setString(3, "서울");   // 세 번째 ?

int n = pstmt.executeUpdate();
```

> `?` 사용 → SQL Injection 방지. `${변수}` 방식의 직접 삽입은 위험.

---

## 트랜잭션 처리

JDBC는 기본 auto commit → 수동 관리가 필요할 때:

```java
con.setAutoCommit(false);   // auto commit 비활성화

try {
    // insert 문
    // update 문
    con.commit();           // 모두 성공 시 커밋
} catch (SQLException e) {
    con.rollback();         // 하나라도 실패 시 롤백
}
```

---

## 레이어 구조 (Layer Pattern)

실무에서 JDBC 코드는 역할에 따라 3개 레이어로 분리한다.

```
사용자 → Presentation Layer → Service Layer → Persistence Layer → MySQL
         (DeptMain.java)      (DeptService)    (DeptDAO.java)
```

| 레이어 | 역할 | 담당 |
|--------|------|------|
| Presentation | 화면 처리 (입출력) | main() |
| Service | 비즈니스 로직 + 트랜잭션 | Connection 획득/반납 |
| Persistence (DAO) | DB 연동 (SQL 실행) | PreparedStatement, ResultSet |

**DTO (Data Transfer Object)**: 레이어 간 데이터 전달용 클래스
```java
// DeptDTO.java
public class DeptDTO {
    private int deptno;
    private String dname;
    private String loc;
    // getter/setter
}
```

---

## JDBC vs MyBatis 비교

| 항목 | JDBC | [[MyBatis]] |
|------|------|------|
| SQL 위치 | 자바 코드 안 | 외부 XML (Mapper) |
| 설정 | 직접 작성 | configuration.xml |
| 예외처리 | try-catch 필수 | 불필요 |
| DML auto commit | ✅ | ❌ (명시적 commit 필요) |
| 코드 간결성 | 보통 | 간결 |

---

## 관련 개념
- [[MySQL]] — JDBC가 연결하는 DBMS
- [[MyBatis]] — JDBC를 추상화한 SQL 매핑 프레임워크
- [[Transaction]] — commit/rollback 개념
