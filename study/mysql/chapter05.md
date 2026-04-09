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

MySQL에서 사용되는 잠금은 크게 스토리지 엔진 레벨과 MySQL 엔진 레벨로 나눌 수 있다. MySQL 엔진 레벨의 잠금은 모든 스토리지 엔진에 영향을 미치지만, 스토리지 엔진 레벨의 잠금은 스토리지 엔진 간 상호 영향을 미치지는 않는다.

MySQL 서버는 하나의 MySQL 엔진과 여러 스토리지 엔진으로 구성된다. MySQL 엔진은 SQL 파싱, 최적화 등을 담당하는 공통 상위 계층이고, 스토리지 엔진(InnoDB, MyISAM, MEMORY 등)은 실제 데이터를 읽고 쓰는 하위 계층이다. 테이블마다 다른 스토리지 엔진을 지정할 수 있으며, 같은 DB 안에서 공존 가능하다. 이 구조 때문에 MySQL 엔진 레벨 잠금(글로벌 락, 테이블 락 등)은 스토리지 엔진과 무관하게 모든 테이블에 영향을 미치고, 스토리지 엔진 레벨 잠금(InnoDB의 레코드 락 등)은 해당 엔진을 사용하는 테이블에만 적용된다.

#### PostgreSQL과의 비교

PostgreSQL은 단일 스토리지 엔진이므로 이런 2단계 구분이 없다. 잠금 체계가 하나로 통합되어 있다.

| 잠금 종류 | MySQL | PostgreSQL 대응 |
|---|---|---|
| 글로벌 락 (`FLUSH TABLES WITH READ LOCK`) | MySQL 엔진 레벨 | 없음 (pg_start_backup 등 별도 메커니즘) |
| 테이블 락 | MySQL 엔진 레벨 | `LOCK TABLE` (동일 개념) |
| 네임드 락 (`GET_LOCK()`) | MySQL 엔진 레벨 | Advisory Lock (`pg_advisory_lock()`) |
| 메타데이터 락 | MySQL 엔진 레벨 | `AccessExclusiveLock` 등 (DDL 시 자동 획득) |
| 레코드(행) 락 | 스토리지 엔진 레벨 (InnoDB) | 행 수준 잠금 (내장) |
| 갭 락 / 넥스트 키 락 | InnoDB 고유 | 없음 (MVCC + SSI로 처리) |

### 5.2.1 글로벌 락

- `FLUSH TABLES WITH READ LOCK` 명령이 실행되기 전에 쓰기 잠금을 거는 SQL이 실행됐다면, 해당 트랜잭션이 완료될 때까지 기다려야 한다. 장시간 SELECT 쿼리가 실행 중일 때도 테이블 flush를 위해 해당 쿼리가 종료될 때까지 기다려야 한다. (쓰기 잠금 대기는 잠금 충돌 때문이고, SELECT 대기는 테이블 flush 단계에서 열려 있는 테이블 핸들을 닫아야 하기 때문으로 서로 다른 이유다.)
- InnoDB는 MVCC를 통해 백업 시점의 스냅샷을 읽을 수 있으므로, 쓰기를 멈추지 않아도 일관된 백업이 가능하다. 반면 MyISAM은 MVCC가 없어 일관된 상태를 보려면 `FLUSH TABLES WITH READ LOCK`으로 모든 쓰기를 멈춰야 한다.
- MySQL 8.0부터는 DDL이나 사용자 생성 같은 메타데이터 변경은 MVCC로 보호되지 않기 때문에, 이런 메타데이터 변경만 막는 가벼운 백업 락이 도입됐다. MVCC는 테이블 안의 데이터(행)에 대해 버전 관리를 하므로 DML(`INSERT`/`UPDATE`/`DELETE`)은 스냅샷으로 일관된 읽기가 가능하지만, DDL(`ALTER TABLE`/`DROP TABLE` 등)이나 사용자/권한 변경(`CREATE USER`/`GRANT` 등)은 MVCC 보호 범위 밖이다. 백업 도중 스키마가 변경되면 백업 파일에 스키마와 데이터가 불일치하는 상태가 저장되어 복원 시 깨진 백업이 된다. 따라서 백업 락은 DML은 허용하되 DDL만 차단하는 가벼운 잠금이다.

  | | 글로벌 락 | 백업 락 |
  |---|---|---|
  | DML (INSERT/UPDATE/DELETE) | 차단 | 허용 |
  | DDL (ALTER/DROP/CREATE TABLE) | 차단 | 차단 |
  | 영향 | 서비스 중단 수준 | 서비스 정상 운영 가능 |

- 소스-레플리카 구성에서 레플리카 서버에서 백업할 때, 글로벌 락을 사용하면 복제가 백업 시간만큼 지연된다. XtraBackup이나 Enterprise Backup 같은 툴은 글로벌 락 없이 MVCC 스냅샷으로 백업하므로 복제가 계속 진행되지만, 백업 도중 소스에서 DDL이 복제로 전파되면 백업이 실패한다. 백업 락은 이 문제를 해결하기 위해 도입되었으며, DML 복제는 계속 진행하되 DDL 복제를 일시 중지시켜 백업 실패를 방지한다. 즉 백업 툴 단독으로는 DDL을 막을 수 없고, 백업 락과 함께 사용해야 한다. 백업 락은 레플리카 서버에 거는 것이며, 소스 서버는 아무 영향 없이 정상 운영된다. 레플리카가 자기 쪽에서 DDL 적용만 잠시 보류하는 것이다.

#### PostgreSQL과의 비교

PostgreSQL은 모든 테이블이 MVCC를 지원하므로 글로벌 읽기 잠금 자체가 필요 없다. 백업 시 `pg_start_backup()` / `pg_stop_backup()` (PostgreSQL 15부터 `pg_backup_start()` / `pg_backup_stop()`)을 사용하며, 백업 중에도 읽기/쓰기/DDL이 모두 정상적으로 수행된다. PostgreSQL의 물리 백업(pg_basebackup 등)은 WAL(Write-Ahead Log) 기반으로 동작하여, 백업 중 DDL이 실행되어도 WAL에 기록되므로 백업이 실패하지 않는다. MySQL처럼 DDL을 막기 위한 별도의 백업 락이 필요하지 않다.

### 5.2.2 테이블 락

- InnoDB 테이블에도 테이블 락이 설정되지만 대부분의 DML 쿼리에서는 무시되고, DDL의 경우에만 영향을 미친다.

#### PostgreSQL과의 비교

PostgreSQL도 DML과 DDL에 따라 다른 수준의 테이블 락이 자동으로 걸린다.

| 작업 | PostgreSQL 테이블 락 | 다른 DML 차단 여부 |
|---|---|---|
| SELECT | `AccessShareLock` | X |
| INSERT/UPDATE/DELETE | `RowExclusiveLock` | X |
| ALTER TABLE / DROP TABLE | `AccessExclusiveLock` | O (모든 접근 차단) |

DML끼리는 테이블 락이 서로 충돌하지 않으므로 동시 실행 가능하고(행 수준 잠금으로 제어), DDL(`AccessExclusiveLock`)만 다른 모든 락과 충돌하여 테이블 전체를 차단한다. 여기서 "호환된다"는 것은 동시에 걸 수 있다(충돌하지 않는다)는 의미다.

PostgreSQL에서 `RowExclusiveLock` 등 DML 시 테이블 락을 거는 이유는 DML끼리의 충돌을 막기 위한 것이 아니라, **DML 실행 중에 DDL이 끼어드는 것을 막기 위한 것**이다. 예를 들어 `INSERT` 실행 중에 `DROP TABLE`이 실행되면 작업 중인 테이블이 사라지는 문제가 발생한다. `RowExclusiveLock`이 걸려 있으면 `AccessExclusiveLock`과 충돌하므로, DDL은 DML이 끝날 때까지 대기하게 된다. 즉 "지금 이 테이블을 사용 중이니 구조를 바꾸지 마라"는 보호막 역할이다.

InnoDB와의 차이점:

| | InnoDB | PostgreSQL |
|---|---|---|
| DML 시 테이블 락 | 걸리지만 무시됨 | 걸리고 유효하지만 DML끼리 호환 |
| 실제 동시성 제어 | 스토리지 엔진의 레코드 기반 잠금 | 행 수준 잠금 (tuple lock) |
| DDL 차단 | 메타데이터 락으로 처리 | 테이블 락 충돌로 처리 |

InnoDB는 "MySQL 엔진 레벨 테이블 락 + 스토리지 엔진 레벨 레코드 락"이라는 2계층 구조 때문에 테이블 락을 무시하는 것이고, PostgreSQL은 단일 잠금 체계에서 락 호환성 매트릭스로 같은 효과를 달성한다.

DML끼리 같은 행을 동시에 수정하려고 할 때, 테이블 락으로는 막을 수 없다(DML끼리 호환되므로). 그래서 행 단위로 잠금을 걸어야 하는데, 이 구현 방식이 다르다.
- **InnoDB**: 스토리지 엔진 레벨에서 인덱스 레코드에 잠금을 건다(레코드 락, 갭 락 등). 별도 잠금 매니저에 잠금 정보를 저장한다.
- **PostgreSQL**: 행(tuple) 자체의 `xmax` 필드에 어떤 트랜잭션이 이 행을 수정 중인지 표시한다. 행 헤더에 직접 트랜잭션 정보를 기록하는 방식이다.

둘 다 "행 수준 잠금"이지만, InnoDB는 별도 잠금 매니저를 통해, PostgreSQL은 행 헤더에 직접 기록하여 처리한다.

PostgreSQL의 모든 행에는 사용자가 만든 컬럼 외에 시스템 컬럼(`xmin`, `xmax` 등)이 숨겨져 있다. `SELECT xmin, xmax, * FROM tab;`으로 직접 조회할 수 있다.
- `xmin`: 이 행을 생성(INSERT)한 트랜잭션 ID
- `xmax`: 이 행을 삭제/수정한 트랜잭션 ID (0이면 아직 아무도 수정/삭제 안 함)

PostgreSQL은 DELETE 시 행을 즉시 물리적으로 삭제하지 않는다. `xmax`에 삭제한 트랜잭션 ID만 기록하고 행 데이터는 그대로 남는다. MVCC를 위해 다른 트랜잭션이 이 행을 아직 볼 수 있어야 하기 때문이다. 나중에 VACUUM 프로세스가 "어떤 트랜잭션도 이 행을 볼 필요가 없다"고 판단하면 그때 물리적으로 정리한다. UPDATE도 내부적으로 "기존 행 DELETE + 새 행 INSERT"로 처리되어, 변경이 많은 테이블은 dead tuple이 쌓인다.

DELETE와 UPDATE 모두 `xmax`를 사용하지만 구분할 필요가 없다. `xmax`가 기록된 행은 어떤 경우든 "이 버전은 더 이상 최신이 아니다"는 의미이기 때문이다.
```
-- 원본 행
(xmin=100, xmax=0, id=1, name='a')     ← 현재 유효

-- UPDATE name='b' 실행 (트랜잭션 200)
(xmin=100, xmax=200, id=1, name='a')   ← 이전 버전 (dead)
(xmin=200, xmax=0,   id=1, name='b')   ← 새 버전 (현재 유효)

-- DELETE 실행 (트랜잭션 300)
(xmin=200, xmax=300, id=1, name='b')   ← 삭제됨 (dead)
```

`xmax`가 0이 아니라고 해서 전부 dead tuple인 것은 아니다.
- 롤백된 트랜잭션: `xmax`에 트랜잭션 ID가 기록됐지만 해당 트랜잭션이 롤백됨 → 행은 여전히 유효
- 행 잠금: `SELECT ... FOR UPDATE` 같은 명령도 `xmax`에 기록됨 → 삭제/수정이 아니라 잠금 표시 용도
- 아직 VACUUM이 안 된 경우: 실제 삭제/수정된 행이지만 VACUUM이 아직 정리하지 않음

실제로 유효한 행인지는 `xmax` 값만으로는 판단할 수 없고, 해당 트랜잭션의 상태(커밋/롤백 여부)와 행의 내부 플래그(infomask)를 함께 봐야 한다.

### 5.2.3 네임드 락

#### PostgreSQL과의 비교: Advisory Lock

MySQL의 네임드 락(`GET_LOCK()`)에 대응하는 PostgreSQL의 기능은 Advisory Lock이다.

| MySQL | PostgreSQL |
|---|---|
| `GET_LOCK('name', timeout)` | `pg_advisory_lock(key)` |
| `RELEASE_LOCK('name')` | `pg_advisory_unlock(key)` |
| 문자열 기반 | 정수(bigint) 기반 |

- MySQL 네임드 락은 임의의 문자열로 잠금을 걸어, 애플리케이션 레벨에서 동기화에 사용한다.
- PostgreSQL Advisory Lock은 동일한 목적이지만, 키가 정수(bigint 1개 또는 int 2개)로 지정된다. 문자열을 쓰려면 해시 변환이 필요하다.
- PostgreSQL은 세션 레벨(`pg_advisory_lock`)과 트랜잭션 레벨(`pg_advisory_xact_lock`) 두 가지를 제공한다.
  - **세션 레벨 (`pg_advisory_lock`)**: 명시적으로 `pg_advisory_unlock()`을 호출하거나 세션(커넥션)이 종료될 때 해제. 트랜잭션과 무관하게 유지되므로 여러 트랜잭션으로 나누어 실행하는 장시간 배치 작업에 적합하다.
  - **트랜잭션 레벨 (`pg_advisory_xact_lock`)**: 트랜잭션이 COMMIT/ROLLBACK될 때 자동 해제. 해제를 잊어버릴 위험이 없지만, 전체 작업을 하나의 트랜잭션으로 묶어야 한다. 긴 트랜잭션은 VACUUM 차단(dead tuple 미정리로 테이블 bloat), 잠금 장기 보유로 인한 다른 세션 대기 유발, 리소스 장기 점유 등의 문제가 있으므로 배치 작업에는 세션 레벨이 적합하다.
  ```sql
  -- 세션 레벨: 트랜잭션이 끝나도 잠금 유지
  SELECT pg_advisory_lock(12345);
  BEGIN; /* 작업 1 */ COMMIT;
  BEGIN; /* 작업 2 */ COMMIT;
  SELECT pg_advisory_unlock(12345);  -- 명시적 해제

  -- 트랜잭션 레벨: 트랜잭션 끝나면 자동 해제
  BEGIN;
  SELECT pg_advisory_xact_lock(12345);
  /* 작업 */
  COMMIT;  -- 잠금 자동 해제
  ```
- `pg_try_advisory_lock`: `pg_advisory_lock`은 잠금을 획득할 때까지 대기(블로킹)하지만, `pg_try_advisory_lock`은 대기 없이 즉시 true/false를 반환한다. "이미 실행 중이면 중복 실행하지 않고 스킵"할 때 유용하다.
  ```sql
  SELECT pg_try_advisory_lock(12345);
  -- true  → 잠금 획득 성공
  -- false → 이미 다른 세션이 잠금 보유 중, 대기 없이 바로 실패
  ```

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
