---
title: HTML5 & CSS
category: tech
tags: [HTML, CSS, 웹, 프론트엔드]
created: 2026-05-23
updated: 2026-05-23
sources: [HTML5_강좌정리.txt, CSS_강좌정리.txt]
---

# HTML5 & CSS

## 한 줄 요약
HTML은 구조(뼈대), CSS는 스타일(외관), JS는 동작(행동) — 웹의 3요소.

---

## 웹 요청/응답 흐름

```
브라우저                               서버 (Apache/Nginx/Tomcat)
  ──── GET /index.html ──────────────→
  ←── HTML + CSS + JS + 이미지 반환 ───
  화면 렌더링 (DOM 트리 구성)
```

### 주요 포트
| 서비스 | 포트 |
|-------|------|
| HTTP | 80 (기본, 생략 가능) |
| HTTPS | 443 |
| Tomcat | 8080 |
| MySQL | 3306 |
| SSH | 22 |

---

## HTML5 기본 구조

```html
<!DOCTYPE html>
<html lang="ko">
  <head>
    <meta charset="UTF-8">
    <title>페이지 제목</title>
    <link rel="stylesheet" href="style.css">
  </head>
  <body>
    <h1>제목</h1>
    <p>문단</p>
  </body>
</html>
```

### DOM 트리
```
html
├── head
│   └── title
└── body
    ├── h1
    └── p
```
CSS/JS는 이 트리를 탐색해서 스타일 적용 및 동적 조작을 한다.

---

## HTML 태그 분류

### 블록 요소 vs 인라인 요소

| | 블록 (Block) | 인라인 (Inline) |
|--|------------|---------------|
| 특징 | 가로 전체 차지, 줄바꿈 | 내용만큼만 |
| width/height | 설정 가능 | 설정 불가 |
| 예시 | `<div>`, `<p>`, `<h1>~<h6>`, `<table>`, `<ul>` | `<span>`, `<a>`, `<img>` |

### 주요 태그

```html
<!-- 제목 (h1이 가장 크고 중요) -->
<h1>제목1</h1>  <!-- 32px -->
<h4>제목4</h4>  <!-- 16px, 기본 폰트 크기 -->

<!-- 링크 -->
<a href="https://google.com">외부 링크</a>
<a href="https://google.com" target="_blank">새 탭에서 열기</a>
<a href="#section-id">페이지 내 이동</a>

<!-- 이미지 -->
<img src="photo.jpg" alt="대체 텍스트">

<!-- 표 -->
<table>
  <thead><tr><th>이름</th><th>나이</th></tr></thead>
  <tbody>
    <tr><td>홍길동</td><td>20</td></tr>
    <tr><td colspan="2">병합된 셀</td></tr>
  </tbody>
</table>

<!-- 목록 -->
<ul><li>순서 없음</li></ul>
<ol><li>순서 있음</li></ol>

<!-- 그룹화 -->
<div>블록 그룹</div>
<span>인라인 그룹</span>

<!-- 시맨틱 태그 (의미 있는 이름) -->
<header>, <nav>, <main>, <section>, <article>, <aside>, <footer>
```

### 폼 (Form)

```html
<form action="/submit" method="post">
  이름: <input type="text" name="username">
  비밀번호: <input type="password" name="passwd">
  성별: <input type="radio" name="gender" value="M"> 남
        <input type="radio" name="gender" value="F"> 여
  <select name="grade">
    <option value="1">1학년</option>
    <option value="2">2학년</option>
  </select>
  <textarea name="content" rows="5"></textarea>
  <input type="submit" value="전송">
</form>
```

**전송 데이터 형식**: `key=value&key=value`  
- GET: URL 뒤에 붙음 (`?username=홍길동&passwd=1234`)  
- POST: HTTP Body에 숨겨서 전송

---

## CSS 스타일 적용 방법 3가지

```html
<!-- 1. inline (최우선, 권장 안함) -->
<p style="color: red;">빨간 글자</p>

<!-- 2. internal -->
<head>
  <style>
    p { color: red; }
  </style>
</head>

<!-- 3. external (권장) -->
<head>
  <link rel="stylesheet" href="style.css">
</head>
```

---

## CSS 기본 문법

```css
선택자 {
  속성: 값;
}

/* 태그 선택자 */
p { color: blue; }

/* 클래스 선택자 (.클래스명) */
.highlight { background-color: yellow; }

/* id 선택자 (#아이디) */
#header { font-size: 24px; }

/* 자손 선택자 */
div p { color: red; }
```

---

## CSS position (중요)

| 값 | 기준점 | top/left 적용 |
|---|--------|-------------|
| `static` | 기본 흐름 (기본값) | ❌ 무시됨 |
| `relative` | 자신의 원래 위치 기준 | ✅ |
| `absolute` | 가장 가까운 position 조상 기준 (없으면 viewport) | ✅ |
| `fixed` | 브라우저 창(viewport) 기준, 스크롤해도 고정 | ✅ |

```css
/* absolute 정상 사용 패턴 */
.parent {
  position: relative;   /* 기준점 설정 */
}
.child {
  position: absolute;
  top: 10px;
  left: 20px;
}
```

---

## 폰트 (font-family)

```css
/* Generic Family: 최후 보루 */
/* serif (삐침 있음), sans-serif (삐침 없음) */

body {
  font-family: 'Noto Sans KR', '맑은 고딕', sans-serif;
  /* 앞에서부터 있는 폰트 사용, 없으면 다음으로 넘어감 */
}
```

---

## Box Model

모든 HTML 요소는 박스 모델 구조를 가짐:
```
[ margin ]
  [ border ]
    [ padding ]
      [ content ]
```

```css
div {
  width: 200px;
  padding: 10px;         /* 안쪽 여백 */
  border: 1px solid black;
  margin: 20px;          /* 바깥 여백 */
  box-sizing: border-box; /* width에 padding + border 포함 (권장) */
}
```

---

## 관련 개념
- [[ECMAScript]] — HTML/CSS와 함께 동작하는 JavaScript
- [[Servlet_JSP]] — 서버에서 HTML을 동적으로 생성
