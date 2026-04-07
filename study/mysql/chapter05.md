# 05장: 트랜잭션과 잠금 `우선순위 상`

## 5.1 트랜잭션

### 5.1.1 MySQL에서의 트랜잭션

#### PostgreSQL과의 비교

MySQL은 스토리지 엔진에 따라 트랜잭션 지원 여부가 달라진다. MyISAM이나 MEMORY 엔진은 트랜잭션을 지원하지 않고, InnoDB만 트랜잭션을 지원한다.

반면 PostgreSQL은 플러그인 방식의 스토리지 엔진 구조가 아닌 단일 내장 엔진을 사용하며, **모든 테이블이 기본적으로 트랜잭션을 지원한다.**

| | MySQL | PostgreSQL |
|---|---|---|
| 스토리지 엔진 | 플러그인 방식 (InnoDB, MyISAM, MEMORY 등) | 단일 내장 엔진 |
| 트랜잭션 지원 | 엔진에 따라 다름 (InnoDB만 지원) | 항상 지원 |
| MVCC | InnoDB만 지원 | 항상 지원 |

- PostgreSQL은 MVCC가 엔진 레벨에 내장되어 있어, 별도 설정 없이 모든 테이블에서 트랜잭션과 동시성 제어가 동작한다.
- MySQL에서 MyISAM 사용 시 발생하는 "부분 업데이트" 문제(트랜잭션 미지원으로 일부 행만 변경되고 롤백 불가)가 PostgreSQL에서는 구조적으로 발생하지 않는다.
- PostgreSQL 14부터 Pluggable Storage API가 도입되어 테이블 Access Method을 확장할 수 있지만(예: columnar, zheap), 이들도 모두 트랜잭션을 전제로 설계된다.
- 부분 업데이트 예시: 이미 fdpk=3이 존재할 때 `INSERT INTO tab (fdpk) VALUES (1), (2), (3)` 실행 시, MyISAM은 1, 2가 삽입된 채로 남지만, InnoDB와 PostgreSQL은 중복 오류로 전체가 롤백되어 아무 행도 삽입되지 않는다.
- 명시적으로 `BEGIN`/`COMMIT`을 쓰지 않아도 단일 SQL 문은 암묵적 트랜잭션으로 감싸져서 실행된다(autocommit 모드). 성공하면 전체 커밋, 중간에 오류가 나면 전체 롤백된다. InnoDB도 autocommit이 기본 ON으로 동일한 방식이며, MyISAM만 트랜잭션 개념 자체가 없어서 실행된 만큼 그대로 남는다.
- PostgreSQL 서버 자체는 항상 autocommit 모드이며, 이를 서버 레벨에서 끄는 설정은 없다. 클라이언트 도구(psql의 `\set AUTOCOMMIT off`, JDBC의 `setAutoCommit(false)` 등)에서 autocommit을 끄는 것처럼 보이지만, 이는 드라이버가 자동으로 `BEGIN`을 발행하는 것일 뿐 서버 설정이 바뀌는 것이 아니다. 명시적 트랜잭션이 필요하면 `BEGIN`~`COMMIT`으로 묶어야 한다.
- Autocommit 모드란 각 SQL 문이 실행 즉시 자동으로 커밋되는 모드다. 문장 하나가 독립적인 트랜잭션으로 처리된다. 여러 문장을 하나의 트랜잭션으로 묶으려면 `BEGIN`~`COMMIT`을 사용해야 한다.
  ```sql
  -- Autocommit: 각 문장이 즉시 커밋됨
  INSERT INTO tab VALUES (1);  -- 즉시 커밋
  INSERT INTO tab VALUES (2);  -- 즉시 커밋
  INSERT INTO tab VALUES (3);  -- 오류 발생해도 1, 2는 이미 커밋되어 남아있음

  -- 명시적 트랜잭션: 전부 성공하거나 전부 롤백
  BEGIN;
  INSERT INTO tab VALUES (1);  -- 아직 커밋 안 됨
  INSERT INTO tab VALUES (2);  -- 아직 커밋 안 됨
  INSERT INTO tab VALUES (3);  -- 오류 발생 시 1, 2도 함께 롤백
  COMMIT;
  ```
- 단일 `INSERT INTO tab VALUES (1),(2),(3)`은 하나의 SQL 문이므로 autocommit 모드에서도 하나의 트랜잭션으로 처리되어, 오류 시 전체가 롤백된다.

### 5.1.2 주의사항

## 5.2 MySQL 엔진의 잠금

### 5.2.1 글로벌 락

### 5.2.2 테이블 락

### 5.2.3 네임드 락

### 5.2.4 메타데이터 락

## 5.3 InnoDB 스토리지 엔진 잠금

### 5.3.1 InnoDB 스토리지 엔진의 잠금

### 5.3.2 인덱스와 잠금

### 5.3.3 레코드 수준의 잠금 확인 및 해제

## 5.4 MySQL의 격리 수준

### 5.4.1 READ UNCOMMITTED

### 5.4.2 READ COMMITTED

### 5.4.3 REPEATABLE READ

### 5.4.4 SERIALIZABLE
