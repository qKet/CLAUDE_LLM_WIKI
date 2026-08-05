---
title: MySQL
category: tech
tags: [MySQL, SQL, 데이터베이스, RDB]
created: 2026-05-23
updated: 2026-05-23
sources: [MySQL_강좌정리.txt]
---

# MySQL

## 한 줄 요약
행(레코드)/열(컬럼) 구조의 관계형 데이터베이스. SQL로 데이터를 정의·조작·조회한다.

---

## SQL 분류

| 분류 | 명령어 | 용도 |
|------|--------|------|
| DQL | `SELECT` | 조회 |
| DML | `INSERT`, `UPDATE`, `DELETE` | 데이터 조작 → 트랜잭션 대상 |
| DDL | `CREATE`, `ALTER`, `DROP`, `TRUNCATE` | 구조 정의 → 자동 commit |
| TCL | `COMMIT`, `ROLLBACK` | 트랜잭션 제어 |

---

## SELECT 기본 문법

```sql
SELECT  컬럼명, ...          -- 5. 출력할 컬럼 선택
FROM    테이블명              -- 1. 테이블 지정
WHERE   조건식               -- 2. 행 필터링
GROUP BY 컬럼명              -- 3. 그룹화
HAVING  조건식               -- 4. 그룹 필터링 (집계함수 조건)
ORDER BY 컬럼명 [ASC|DESC];  -- 6. 정렬 (기본: ASC)
```

> **실행 순서**: FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY

### WHERE 연산자
```sql
WHERE sal = 1000           -- 같음
WHERE sal != 1000          -- 다름
WHERE sal BETWEEN 1000 AND 2000   -- 범위 (경계값 포함)
WHERE sal IN (800, 1000, 1500)    -- 목록
WHERE ename LIKE '%A%'     -- 패턴 (% = 0개 이상, _ = 정확히 1개)
WHERE comm IS NULL         -- NULL 확인 (= NULL 불가)
WHERE ename = 'SMITH' AND sal > 1000
WHERE ename = 'SMITH' OR sal > 1000
```

---

## 함수

### 문자열 함수
```sql
LOWER(ename), UPPER(ename)
CONCAT('My', 'SQL')                    -- 'MySQL'
SUBSTR(str, 시작위치, 길이)            -- 위치는 1부터
LENGTH(str)                            -- 바이트 수 (한글 3byte)
CHAR_LENGTH(str)                       -- 글자 수
REPLACE('www.mysql.com', 'w', 'W')
TRIM(str), LTRIM(str), RTRIM(str)
FORMAT(12332.12, 2)                    -- '12,332.12' (천단위 콤마)
```

### 숫자 함수
```sql
ABS(-32)                               -- 32 (절댓값)
ROUND(150.567, 2)                      -- 150.57 (반올림)
TRUNCATE(1.999, 1)                     -- 1.9 (버림)
CEIL(1.1), FLOOR(1.9)                  -- 2, 1
MOD(10, 3)                             -- 1 (나머지)
```

### 날짜 함수
```sql
NOW()                                  -- '2026-05-23 14:30:00'
CURDATE(), CURTIME()
ADDDATE('2026-01-01', 31)             -- 날짜 더하기
DATE_ADD('2026-01-01', INTERVAL 1 MONTH)
DATEDIFF('2026-12-31', '2026-01-01')  -- 365
DATE_FORMAT(NOW(), '%Y년%m월%d일')
STR_TO_DATE('2026/05/23', '%Y/%m/%d') -- 문자열 → 날짜
EXTRACT(YEAR FROM NOW())              -- 년도 추출
```

### 흐름 제어 함수
```sql
IF(sal > 3000, '고급', '일반')

CASE job
    WHEN 'CLERK' THEN sal * 1.1
    WHEN 'MANAGER' THEN sal * 1.3
    ELSE sal
END AS 조정급여

CASE
    WHEN sal > 3000 THEN '이상급'
    WHEN sal > 2000 THEN '중급'
    ELSE '하급'
END
```

### 집계 함수
```sql
SUM(sal), AVG(sal), MAX(sal), MIN(sal)
COUNT(sal)     -- NULL 제외 카운트
COUNT(*)       -- 전체 레코드 수

-- 그룹별 집계
SELECT deptno, SUM(sal)
FROM emp
GROUP BY deptno
HAVING SUM(sal) > 9000;     -- WHERE에는 집계함수 사용 불가
```

---

## JOIN

```
emp 테이블과 dept 테이블을 연결해서 부서명도 함께 조회
```

### INNER JOIN (기본)
```sql
-- ON 조건
SELECT e.ename, e.sal, d.dname
FROM emp e JOIN dept d ON e.deptno = d.deptno;

-- USING (공통 컬럼명이 같을 때)
SELECT ename, sal, dname
FROM emp JOIN dept USING(deptno);

-- non-equi JOIN (범위 조건)
SELECT e.ename, e.sal, s.grade
FROM emp e JOIN salgrade s ON e.sal BETWEEN s.losal AND s.hisal;
```

### OUTER JOIN
```sql
-- LEFT: emp 모든 행 + 매칭 dept
FROM emp LEFT OUTER JOIN dept USING(deptno)

-- RIGHT: dept 모든 행 + 매칭 emp
FROM emp RIGHT OUTER JOIN dept USING(deptno)
```

### SELF JOIN
```sql
-- 사원과 매니저 이름을 같은 테이블에서 조회
SELECT e.ename AS 사원, m.ename AS 매니저
FROM emp e JOIN emp m ON e.mgr = m.empno;
```

---

## 서브쿼리 (Subquery)

```sql
-- 단일 행 서브쿼리 (=, >, < 등)
SELECT * FROM emp
WHERE sal > (SELECT AVG(sal) FROM emp);

-- 다중 행 서브쿼리 (IN, ANY, ALL)
SELECT * FROM emp
WHERE sal IN (SELECT MIN(sal) FROM emp GROUP BY job);

-- ALL: 최대값보다 큰 (모든 값보다 큰)
WHERE sal > ALL (SELECT sal FROM emp WHERE job = 'MANAGER')

-- ANY: 최솟값보다 큰 (하나라도 큰)
WHERE sal > ANY (SELECT sal FROM emp WHERE job = 'MANAGER')

-- 인라인 뷰 (FROM절 서브쿼리)
SELECT e.deptno, total_sum
FROM (SELECT deptno, SUM(sal) total_sum FROM emp GROUP BY deptno) e
JOIN dept d ON e.deptno = d.deptno;
```

---

## DML (데이터 조작)

```sql
-- INSERT
INSERT INTO dept (deptno, dname, loc) VALUES (50, '인사', '서울');
INSERT INTO dept VALUES (60, '개발', '부산', NULL);  -- 컬럼 순서대로

-- 다건 INSERT
INSERT INTO dept (deptno, dname)
VALUES (70, 'A'), (80, 'B'), (90, 'C');

-- UPDATE
UPDATE emp SET sal = sal * 1.1 WHERE deptno = 20;

-- DELETE
DELETE FROM emp WHERE empno = 9000;
```

> DML은 [[Transaction]] 대상 — `COMMIT`/`ROLLBACK` 필요 (MySQL은 기본 auto commit)

---

## DDL (테이블 정의)

```sql
CREATE TABLE IF NOT EXISTS board (
    num       INT PRIMARY KEY AUTO_INCREMENT,
    title     VARCHAR(100) NOT NULL,
    author    VARCHAR(10) UNIQUE,
    content   VARCHAR(500) NOT NULL,
    writeday  DATETIME DEFAULT NOW(),
    gender    CHAR(4) CONSTRAINT CHECK (gender IN ('M', 'F')),
    readcnt   INT DEFAULT 0
);

-- 테이블 수정
ALTER TABLE board ADD COLUMN views INT DEFAULT 0;
ALTER TABLE board DROP COLUMN views;
ALTER TABLE board MODIFY title VARCHAR(200) NOT NULL;
ALTER TABLE board RENAME COLUMN author TO writer;

-- 테이블 삭제
DROP TABLE IF EXISTS board;
TRUNCATE TABLE board;   -- 모든 레코드 삭제 (rollback 불가)
```

---

## 제약조건

| 제약조건 | 설명 |
|---------|------|
| `PRIMARY KEY` | NOT NULL + UNIQUE, 레코드 식별자. 자동 인덱스 생성 |
| `UNIQUE` | 중복 불가, NULL은 여러 개 허용 |
| `NOT NULL` | NULL 금지 |
| `CHECK` | 특정 조건 강제 (ex. gender IN ('M','F')) |
| `FOREIGN KEY` | 다른 테이블 PK/UNIQUE 참조. JOIN의 기반 |

### FOREIGN KEY 옵션
```sql
CONSTRAINT FOREIGN KEY(deptno) REFERENCES dept(deptno)
    ON DELETE CASCADE    -- 부모 삭제 시 자식도 삭제
    ON DELETE SET NULL   -- 부모 삭제 시 자식의 FK를 NULL로
```

---

## 데이터 타입

| 분류 | 타입 | 설명 |
|------|------|------|
| 정수 | `TINYINT`, `INT`, `BIGINT` | 1/4/8 byte |
| 실수 | `FLOAT`, `DOUBLE` | |
| 문자 | `CHAR(n)` | 고정길이 (패딩됨) |
| | `VARCHAR(n)` | 가변길이 (실제 길이만 저장) |
| 날짜 | `DATE`, `TIME`, `DATETIME` | |

---

## 관련 개념
- [[Transaction]] — COMMIT/ROLLBACK 동작 원리
- [[JDBC]] — Java에서 MySQL 연동
- [[MyBatis]] — SQL 매핑 프레임워크
