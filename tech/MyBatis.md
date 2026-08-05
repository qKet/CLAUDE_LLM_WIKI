---
title: MyBatis
category: tech
tags: [MyBatis, Java, SQL, ORM, 백엔드]
created: 2026-05-23
updated: 2026-05-23
sources: [MyBatis_강좌정리.txt]
---

# MyBatis

## 한 줄 요약
JDBC를 편리하게 감싼 SQL Mapping Framework. SQL은 XML에, Java 코드에는 비즈니스 로직만 남긴다.

---

## 핵심 아이디어

JDBC의 반복적인 보일러플레이트 코드(드라이버 로딩, Connection, PreparedStatement, ResultSet...)를 제거하고, SQL을 XML 파일로 분리해서 관리.

```
개발자가 메서드 호출 → MyBatis가 해당 SQL 자동 실행 → 결과를 DTO에 자동 매핑
```

---

## 설정 구조

```
src/
  com/
    config/
      jdbc.properties       # DB 접속 정보
      configuration.xml     # MyBatis 메인 설정
      DeptMapper.xml        # SQL 정의 (테이블당 1개)
    dto/
      DeptDTO.java          # 레코드 ↔ 객체 매핑
    dao/
      DeptDAO.java          # SqlSession 사용
```

### jdbc.properties
```properties
jdbc.driver=com.mysql.cj.jdbc.Driver
jdbc.url=jdbc:mysql://localhost:3306/testdb
jdbc.userid=root
jdbc.passwd=1234
```

### configuration.xml
```xml
<configuration>
    <properties resource="com/config/jdbc.properties"/>

    <!-- DTO 별칭 등록 -->
    <typeAliases>
        <typeAlias alias="DeptDTO" type="com.dto.DeptDTO"/>
    </typeAliases>

    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="${jdbc.driver}"/>
                <property name="url" value="${jdbc.url}"/>
                <property name="username" value="${jdbc.userid}"/>
                <property name="password" value="${jdbc.passwd}"/>
            </dataSource>
        </environment>
    </environments>

    <mappers>
        <mapper resource="com/config/DeptMapper.xml"/>
    </mappers>
</configuration>
```

### DTO 규칙
- 컬럼명 = 변수명 (일치 시 자동 매핑)
- 기본 생성자 + getter/setter 필수

```java
public class DeptDTO {
    private int deptno;
    private String dname;
    private String loc;
    // getter/setter 필수
}
```

---

## Mapper XML 문법

### SELECT
```xml
<!-- 단건 조회: selectOne() -->
<select id="findByDeptno" parameterType="int" resultType="DeptDTO">
    SELECT deptno, dname, loc
    FROM dept
    WHERE deptno = #{deptno}    <!-- JDBC의 ? 역할, SQL Injection 방지 -->
</select>

<!-- 다건 조회: selectList() -->
<select id="findAll" resultType="DeptDTO">
    SELECT deptno, dname, loc FROM dept
</select>
```

### INSERT / UPDATE / DELETE
```xml
<insert id="insert" parameterType="DeptDTO">
    INSERT INTO dept(deptno, dname, loc) VALUES(#{deptno}, #{dname}, #{loc})
</insert>

<update id="update" parameterType="DeptDTO">
    UPDATE dept SET dname=#{dname}, loc=#{loc} WHERE deptno=#{deptno}
</update>

<delete id="delete" parameterType="int">
    DELETE FROM dept WHERE deptno=#{deptno}
</delete>
```

### Java에서 호출
```java
SqlSession session = MySqlSessionFactory.getSession();

// 단건 조회
DeptDTO dto = session.selectOne("findByDeptno", 10);

// 다건 조회
List<DeptDTO> list = session.selectList("findAll");

// DML
int n = session.insert("insert", new DeptDTO(50, "인사", "서울"));
session.commit();    // ⚠️ MyBatis는 auto commit 아님 — 명시적 commit 필요

session.close();
```

---

## 동적 SQL

### `<if>` — 조건별 WHERE절
```xml
<select id="searchEmp" parameterType="EmpDTO" resultType="EmpDTO">
    SELECT * FROM emp
    <where>
        <if test="job != null">
            job = #{job}
        </if>
        <if test="sal != 0">
            AND sal = #{sal}
        </if>
    </where>
</select>
```

### `<foreach>` — IN절, 다건 INSERT/DELETE
```xml
<!-- 다건 DELETE -->
<delete id="deleteMulti" parameterType="arraylist">
    DELETE FROM emp
    <where>
        <foreach item="num" collection="list"
                 open="empno IN (" separator="," close=")">
            #{num}
        </foreach>
    </where>
</delete>
```

### `<set>` — UPDATE의 동적 컬럼
```xml
<update id="updateIf" parameterType="EmpDTO">
    UPDATE emp
    <set>
        <if test="ename != null">ename = #{ename},</if>
        <if test="sal != 0">sal = #{sal}</if>
    </set>
    WHERE empno = #{empno}
</update>
```

### `<choose>` — if-else 대체
```xml
<choose>
    <when test="job == 'SALESMAN'">sal > 1500</when>
    <when test="job == 'CLERK'">sal > 2500</when>
    <otherwise>sal > 3000</otherwise>
</choose>
```

---

## resultMap — 컬럼명 ≠ 변수명 매핑

```xml
<resultMap id="empMap" type="EmpDTO">
    <id property="empno" column="empno"/>
    <result property="salary" column="sal"/>    <!-- sal → salary -->

    <!-- 1:1 조인 (association) -->
    <association property="dept" javaType="DeptDTO">
        <id property="deptno" column="deptno"/>
        <result property="dname" column="dname"/>
    </association>

    <!-- 1:N 조인 (collection) -->
    <collection property="empList" ofType="EmpDTO">
        <id property="empno" column="empno"/>
        <result property="ename" column="ename"/>
    </collection>
</resultMap>
```

---

## 비교 연산자 이슈

XML에서 `<`, `>` 는 태그로 파싱됨 → 두 가지 해결 방법:

```xml
<!-- 방법 1: HTML 엔티티 -->
WHERE deptno &lt; 40

<!-- 방법 2: CDATA 섹션 -->
<![CDATA[
    WHERE deptno < 40
]]>
```

---

## 페이징 (RowBounds)

```java
int offset = (curPage - 1) * perPage;
List<EmpDTO> list = session.selectList("findAll", null, new RowBounds(offset, perPage));
```

---

## JDBC vs MyBatis 한 눈에

| | JDBC | MyBatis |
|--|------|---------|
| SQL 위치 | 자바 코드 | XML (Mapper) |
| 예외처리 | 필수 | 불필요 |
| auto commit | ✅ | ❌ (commit 명시 필요) |
| ConnectionPool | ❌ | ✅ (POOLED) |
| 코드량 | 많음 | 적음 |

---

## 관련 개념
- [[JDBC]] — MyBatis가 추상화한 원본 기술
- [[MySQL]] — 연동 대상 DB
- [[SpringBoot]] — MyBatis를 Spring과 통합하는 방법
- [[Transaction]] — commit/rollback 개념
