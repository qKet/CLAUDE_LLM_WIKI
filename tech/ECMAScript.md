---
title: ECMAScript (JavaScript)
category: tech
tags: [JavaScript, ECMAScript, ES6, 프론트엔드, DOM]
created: 2026-05-23
updated: 2026-05-23
sources: [ECMAScript_강좌정리.txt]
---

# ECMAScript (JavaScript)

## 한 줄 요약
웹 브라우저의 동작 처리 언어. HTML/CSS와 달리 프로그래밍 언어 — 이벤트 처리, DOM 조작, 서버 통신을 담당.

---

## 타입 시스템

### 기본형 (Primitive)
```js
// 숫자: 정수/실수 구분 없음
let n = 10;
let f = 3.14;

// 문자열
let s = "hello";
let s2 = 'world';

// 논리
let b = true;   // false

// 특수값
let u = undefined;  // 변수 선언 후 값 없음
let nu = null;      // 의도적으로 비어 있음
let nan = NaN;      // Not a Number
```

**falsy 값 5가지**: `0`, `""`, `null`, `undefined`, `NaN`  
→ if 조건에서 false로 처리됨. 나머지는 모두 truthy.

### 변수 선언
```js
var x = 1;   // 함수 scope, 중복 선언 가능 (구식)
let y = 2;   // 블록 scope, 중복 선언 불가 (ES6, 권장)
const z = 3; // 상수, 재할당 불가
```

---

## 객체 (Object)

```js
// JSON 표기법
let person = {
    name: "홍길동",
    age: 20,
    phone: ["010", "011"],   // value는 어떤 타입이든 가능
    getName() { return this.name; },  // ES6 메서드 단축
    setName(n) { this.name = n; }
};

// 접근 방법 2가지
person.name          // dot 표기법
person["name"]       // 대괄호 표기법 (동적 key 가능)

let key = "name";
person[key];         // → person.name과 동일

// 단축 프로퍼티
let name = "홍길동", age = 20;
let p = { name, age };    // { name: name, age: age } 와 동일
```

### JSON 변환
```js
// 문자열 → 객체/배열
JSON.parse('{"name":"홍길동"}')  // → {name: "홍길동"}

// 객체/배열 → 문자열 (서버 전송 시)
JSON.stringify({name: "홍길동"}) // → '{"name":"홍길동"}'
```

---

## 함수 (Function)

### 선언 방식
```js
// 1. 함수 선언식 (호이스팅 됨 — 선언 전 호출 가능)
function greet(name) {
    return "안녕 " + name;
}

// 2. 함수 표현식 (변수에 할당 — 선언 후에만 호출 가능)
const greet2 = function(name) {
    return "안녕 " + name;
};

// 3. 화살표 함수 (ES6, 함수 표현식의 단축)
const add = (a, b) => a + b;
const hello = () => console.log("hi");
const double = n => n * 2;       // 파라미터 1개면 () 생략 가능
const getObj = () => ({ key: 1 }); // 객체 반환 시 () 필요
```

### 함수는 일급 객체
```js
// 함수를 값처럼 변수에 저장, 인자로 전달, 반환 가능
setTimeout(function() {
    console.log("2초 후 실행");
}, 2000);

// 콜백 함수: 직접 호출하지 않고 특정 시점에 시스템이 호출하는 함수
[1, 2, 3].forEach(num => console.log(num));
```

### Generator 함수
```js
function* gen() {
    yield 1;  // 여기서 일시 정지
    yield 2;
    yield 3;
}
const g = gen();
g.next();  // { value: 1, done: false }
g.next();  // { value: 2, done: false }
```

---

## 이벤트 처리

```js
// DOM Level 0 (구식)
document.querySelector("#btn").onclick = function() {
    alert("클릭!");
};

// DOM Level 2 (권장, 여러 핸들러 등록 가능)
const btn = document.querySelector("#btn");
btn.addEventListener("click", function(event) {
    console.log("클릭됨", event);
});

// 주요 이벤트 타입
"click", "dblclick", "keyup", "keydown", "mouseover",
"submit", "change", "load"
```

---

## DOM 조작

```js
// 요소 선택
const el = document.querySelector("#id");         // 단일 (css 선택자)
const els = document.querySelectorAll(".class");   // 여러 개 (NodeList 반환)

// 내용 변경
el.textContent = "새 텍스트";    // 텍스트만
el.innerHTML = "<b>HTML</b>";   // HTML 삽입

// 속성 조작
el.setAttribute("href", "https://google.com");
el.getAttribute("href");

// 스타일 변경
el.style.color = "red";
el.classList.add("active");
el.classList.remove("active");
el.classList.toggle("active");

// 요소 생성/추가/삭제
const div = document.createElement("div");
div.textContent = "새 요소";
document.body.appendChild(div);
el.remove();
```

---

## 구조 분해 할당 (Destructuring)

```js
// 배열
const [x, y] = [10, 20];      // x=10, y=20
const [a, , b] = [1, 2, 3];   // a=1, b=3 (가운데 건너뜀)

// 객체
const { name, age } = { name: "홍길동", age: 20 };
const { name: userName } = { name: "홍길동" };  // 다른 변수명으로
```

---

## Spread 연산자

```js
const arr = [1, 2, 3];

// 배열 복사
const copy = [...arr];                  // [1, 2, 3]

// 배열 합치기
const merged = [...arr, 4, 5];          // [1, 2, 3, 4, 5]
const merged2 = [...arr, ...[4, 5]];    // [1, 2, 3, 4, 5]

// 함수 인자
Math.max(...arr);   // Math.max(1, 2, 3) 과 동일
```

---

## BOM vs DOM

### BOM (Browser Object Model)
```js
window           // 브라우저 창 (전역 객체)
window.location  // URL 정보, 이동
window.history   // 뒤로/앞으로
window.screen    // 화면 크기
navigator        // 브라우저 정보
```

### DOM (Document Object Model)
```js
document         // HTML 문서 전체
document.querySelector()
document.createElement()
```

> `window.location.href = "/main"` = `location.href = "/main"` (window 생략 가능)

---

## 논리 연산자 단축 평가

```js
10 && "홍길동"   // → "홍길동" (좌변이 truthy면 우변 반환)
NaN && 100      // → NaN (좌변이 falsy면 좌변 반환)

10 || "홍길동"   // → 10 (좌변이 truthy면 좌변 반환)
NaN || 100      // → 100 (좌변이 falsy면 우변 반환)

// 실용 예시
const name = user?.name || "익명";   // Optional chaining + OR
```

---

## 관련 개념
- [[HTML5_CSS]] — ECMAScript가 조작하는 구조와 스타일
- [[Servlet_JSP]] — 서버에서 생성한 HTML에 JS가 동작
- [[REST API]] — fetch/axios로 JSON 데이터 주고받기
