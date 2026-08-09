---
title: docker-compose.yml 볼륨 경로가 저장소 분리 후 안 맞던 문제
category: troubleshooting
tags: [backend, docker, polyrepo]
created: 2026-08-06
updated: 2026-08-06
---

# docker-compose.yml 볼륨 경로가 저장소 분리 후 안 맞던 문제

## 증상

`backend/docker-compose.yml`로 `docker compose up -d`를 돌리면 에러 없이 컨테이너는 뜨지만, MySQL에 스키마/시드 데이터가 하나도 없는 상태로 올라옴 (조용히 실패).

## 원인

예전 모노레포(`Q_Ket_old`) 시절엔 `docker-compose.yml`이 레포 루트에 있었고, 볼륨 경로가 `./backend/src/main/resources/schema.sql`처럼 **backend 하위 폴더를 가리키는 상대경로**였다.

저장소를 서비스별로 분리하면서 이 파일을 `backend/` 레포 안으로 그대로 복사했는데, 경로는 안 고쳐서 `./backend/src/...`가 그대로 남아있었다. 이 파일이 이제 `backend/` **안에** 있으므로 실제로는 `backend/backend/src/...`를 찾게 되고, 그런 경로는 없다.

Docker는 존재하지 않는 호스트 경로를 볼륨 마운트하려고 하면 **에러를 내지 않고 빈 디렉토리를 만들어 마운트**해버린다 — 그래서 `docker-entrypoint-initdb.d`에 `schema.sql`/`data.sql`이 안 들어가고, MySQL 컨테이너는 정상 기동했지만 테이블이 하나도 없는 상태가 된다.

## 해결

`backend/docker-compose.yml`의 볼륨 경로에서 중복된 `backend/` prefix를 제거:

```diff
-      - ./backend/src/main/resources/schema.sql:/docker-entrypoint-initdb.d/1-schema.sql
-      - ./backend/src/main/resources/data.sql:/docker-entrypoint-initdb.d/2-data.sql
+      - ./src/main/resources/schema.sql:/docker-entrypoint-initdb.d/1-schema.sql
+      - ./src/main/resources/data.sql:/docker-entrypoint-initdb.d/2-data.sql
```

## 재발 방지

**저장소를 분리/이동할 때, 상대경로를 쓰는 설정 파일(docker-compose, CI 스크립트 등)은 옮긴 위치 기준으로 경로를 다시 검증한다.** 특히 Docker 볼륨 마운트는 경로가 틀려도 에러 없이 빈 디렉토리로 대체되므로 겉으로는 정상 기동한 것처럼 보인다 — `docker compose up -d` 후 실제로 테이블이 들어왔는지 확인하는 습관이 필요하다:

```bash
docker exec -i qket-mysql mysql -uroot -p1234 qket -e "SHOW TABLES;"
```

## 관련
- [[getting-started]]
- [[db-schema-change]]
