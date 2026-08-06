---
title: docker exec MySQL 한글 이중 인코딩 버그
category: troubleshooting
tags: [database, docker, mysql]
created: 2026-08-06
updated: 2026-08-06
---

# docker exec MySQL 한글 이중 인코딩 버그

## 증상

떠 있는 MySQL 컨테이너(`qket-mysql`)에 `docker exec`로 직접 `ALTER`/`INSERT` 쿼리를 날렸을 때, 한글 데이터가 깨진 채(이중 인코딩) 저장됨.

## 원인

`docker exec -i qket-mysql mysql -uroot -p1234 qket -e "..."`처럼 `--default-character-set` 옵션 없이 접속하면 mysql 클라이언트가 **latin1**로 붙어서, 입력한 한글이 latin1 기준으로 잘못 해석된 채 저장된다.

## 해결

반드시 `--default-character-set=utf8mb4`를 붙여서 접속한다.

```bash
docker exec -i qket-mysql mysql --default-character-set=utf8mb4 -uroot -p1234 qket -e "..."
```

## 재발 방지

로컬 DB에 직접 쿼리를 날려야 하는 모든 상황(스키마 변경 등)은 [[db-schema-change]] 런북의 절차를 따른다 — 이 플래그가 절차에 포함되어 있다.

## 관련
- [[db-schema-change]]
- [[db-schema-conventions]]
