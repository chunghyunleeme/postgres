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

- DDL과 DML의 충돌을 방지하기 위한 잠금이다. DML 실행 시 자동으로 메타데이터 읽기 잠금을 획득하고, DDL 실행 시 메타데이터 쓰기 잠금을 획득한다. 예를 들어 `SELECT` 실행 중에 `ALTER TABLE`이 들어오면, SELECT가 끝날 때까지 DDL이 대기한다.

#### PostgreSQL과의 비교

PostgreSQL은 별도의 메타데이터 락이 없고, 테이블 수준 잠금으로 동일한 역할을 수행한다.

| | MySQL | PostgreSQL |
|---|---|---|
| 메커니즘 | 별도의 메타데이터 락 (MDL) | 테이블 수준 잠금 |
| DML 시 | MDL 읽기 잠금 자동 획득 | `RowExclusiveLock` 등 획득 |
| DDL 시 | MDL 쓰기 잠금 획득 (DML과 충돌) | `AccessExclusiveLock` 획득 (다른 모든 잠금과 충돌) |
| 존재 이유 | MySQL 엔진과 스토리지 엔진이 분리되어 있어 별도 계층 필요 | 단일 잠금 체계에 통합 |

MySQL은 2계층 구조(MySQL 엔진 + 스토리지 엔진) 때문에 메타데이터 보호를 위한 별도 잠금 계층이 필요하지만, PostgreSQL은 테이블 락의 호환성 매트릭스로 같은 효과를 달성한다.

## 5.3 InnoDB 스토리지 엔진 잠금

- InnoDB는 MySQL 엔진의 잠금과는 별개로 스토리지 엔진 내부에서 레코드 기반의 잠금을 탑재하고 있어 MyISAM보다 뛰어난 동시성 처리를 제공한다. 하지만 이원화된 잠금 처리 탓에 잠금 정보에 접근하기가 까다로웠다.
- 예전: `innodb_lock_monitor` 테이블 생성으로 잠금 정보를 덤프하거나 `SHOW ENGINE INNODB STATUS` 정도만 가능 (가독성 매우 낮음)
- 이후: `information_schema`의 `INNODB_TRX`, `INNODB_LOCKS`, `INNODB_LOCK_WAITS` 테이블을 조인하여 트랜잭션의 잠금 대기 상태 조회 가능
- 최근: `performance_schema`로 InnoDB 내부 잠금(세마포어)까지 모니터링 가능

#### PostgreSQL과의 비교

PostgreSQL은 단일 잠금 체계이므로 처음부터 통합된 모니터링을 제공한다.

| 용도 | MySQL | PostgreSQL |
|---|---|---|
| 현재 잠금 조회 | `INNODB_LOCKS` / `performance_schema` | `pg_locks` 뷰 |
| 잠금 대기 조회 | `INNODB_LOCK_WAITS` | `pg_locks`에서 `granted = false` 조회 |
| 커넥션/트랜잭션 조회 | `SHOW PROCESSLIST` (커넥션) + `INNODB_TRX` (트랜잭션) 분리 | `pg_stat_activity` 뷰 하나에 통합 |
| 블로킹 세션 찾기 | 여러 테이블 조인 필요 | `pg_blocking_pids()` 함수 |

MySQL은 2계층 구조 때문에 잠금 정보가 분산되어 있어 모니터링이 복잡했지만, PostgreSQL은 단일 체계라 `pg_locks`와 `pg_stat_activity` 정도로 대부분 파악 가능하다. `pg_locks`는 `pg_catalog` 스키마에 있는 내장 시스템 뷰로, 별도 설치 없이 사용 가능하며, 조회 시점에 서버 메모리에서 실시간 잠금 정보를 가져오는 가상 테이블이다. `pg_stat_activity`는 커넥션(세션) 단위로 정보를 보여주는 뷰이지만, 각 커넥션의 트랜잭션 상태(`state`, `xact_start`, `query`, `query_start` 등)도 함께 포함하고 있어 트랜잭션 모니터링에도 사용된다. MySQL에서는 커넥션 조회(`SHOW PROCESSLIST`)와 트랜잭션 조회(`INNODB_TRX`)가 분리되어 있지만, PostgreSQL은 `pg_stat_activity` 하나에 통합되어 있다.

### 5.3.1 InnoDB 스토리지 엔진의 잠금

- 락 에스컬레이션 없음: 다른 DBMS(예: SQL Server)에서는 잠금 수가 임계치를 넘으면 메모리 절약을 위해 더 큰 단위(페이지 → 테이블)로 합치는 락 에스컬레이션이 발생하여 동시성이 떨어진다. InnoDB는 잠금 정보 하나가 차지하는 공간이 매우 작아서 레코드가 아무리 많아도 항상 레코드 단위 잠금을 유지한다. PostgreSQL도 `xmax` 방식이라 별도 메모리가 불필요하여 락 에스컬레이션이 없다.

#### 5.3.1.1 레코드 락

- 인덱스 레코드 자체에 거는 잠금이다. InnoDB는 레코드 자체가 아니라 인덱스의 레코드를 잠근다. 인덱스가 하나도 없는 테이블이더라도 내부적으로 자동 생성된 클러스터 인덱스를 이용해 잠금을 설정한다.
- InnoDB는 **스캔한 모든 인덱스 레코드에 잠금을 건다.** 인덱스가 좁은 범위를 가리키면 적게 잠기고, 풀스캔이면 전부 잠긴다.
  - `WHERE id = 1` (PK 조회): 클러스터 인덱스에서 id=1을 바로 찾음 → 1건만 잠금
  - `WHERE first_name = 'John'` (세컨더리 인덱스 조회): 세컨더리 인덱스에서 'John'을 찾아 잠금 → PK를 따라 클러스터 인덱스로 이동하여 잠금 (두 인덱스를 모두 경유했으므로 두 곳에 잠금)
  - `WHERE last_name = 'Smith'` (인덱스 없는 컬럼 조회): 클러스터 인덱스 풀스캔 → 스캔한 모든 레코드에 잠금 (사실상 테이블 전체 잠금)
- DML 실행 시 자동으로 걸리며, 모드에 따라 배타적 잠금과 공유 잠금으로 나뉜다.

  | 동작 | 잠금 모드 |
  |---|---|
  | `UPDATE`, `DELETE`, `SELECT ... FOR UPDATE` | 배타적 잠금 (X lock) |
  | `SELECT ... FOR SHARE` | 공유 잠금 (S lock) |
  | 일반 `SELECT` | 잠금 없음 (MVCC 스냅샷 읽기) |

- 배타적 잠금과 공유 잠금의 호환성:

  | | X lock 요청 | S lock 요청 |
  |---|---|---|
  | X lock 보유 중 | 대기 | 대기 |
  | S lock 보유 중 | 대기 | 허용 |

- 인덱스 없는 컬럼으로 조회 시 클러스터 인덱스를 풀스캔하며 모든 레코드에 잠금을 건다. 조건에 맞지 않는 레코드의 잠금은 확인 후 해제되지만, 스캔하는 순간에는 잠금이 걸려 있으므로 다른 세션이 짧은 시간 동안 대기할 수 있다. 그래서 InnoDB에서는 인덱스 설계가 잠금 성능에 직접적인 영향을 준다.
- INSERT의 경우: 기존 레코드를 스캔하는 것이 아니라 새 레코드를 추가하므로, 삽입할 위치의 갭에 **인서트 인텐션 락(Insert Intention Lock)**을 건다. 이는 갭 락의 특수한 형태로, 같은 갭에 다른 위치로 삽입하는 것은 서로 충돌하지 않는다. 삽입 완료 후 해당 레코드에 배타적 잠금(X lock)이 걸린다.
  ```sql
  -- 기존 데이터: id = 3, 7
  -- 세션 A: INSERT (id=4) → 갭(3~7)에 인서트 인텐션 락 → 삽입 → 레코드 락
  -- 세션 B: INSERT (id=5) → 같은 갭이지만 다른 위치 → 충돌 없이 동시 삽입 가능

  -- 하지만 다른 세션이 이미 갭 락을 걸고 있으면
  -- 세션 A: SELECT * FROM tab WHERE id BETWEEN 3 AND 7 FOR UPDATE;  -- 갭 락
  -- 세션 B: INSERT (id=5) → 갭 락과 충돌 → 대기
  ```

##### PostgreSQL과의 비교

| | InnoDB | PostgreSQL |
|---|---|---|
| 잠금 대상 | 인덱스 레코드 | 행(tuple) 헤더의 `xmax`로 처리 |
| 인덱스 없을 때 | 풀스캔하며 모든 레코드 잠금 | 조건에 맞는 행만 잠김 |
| INSERT 시 | 인서트 인텐션 락 + 레코드 락 | 행에 `xmax` 기록만, 중복은 UNIQUE 제약이 처리 |
| 구현 방식 | 별도 잠금 매니저 | 행 헤더에 직접 기록 |

#### 5.3.1.2 갭 락

- 레코드와 레코드 사이의 간격에 거는 잠금이다. 아직 존재하지 않는 레코드가 삽입되는 것을 방지한다.
- 갭 락이 해결하는 문제는 팬텀 리드(Phantom Read)이다. InnoDB와 PostgreSQL은 이를 다른 방식으로 처리한다.

```sql
-- InnoDB (REPEATABLE READ): 삽입 자체를 차단 (비관적 방식)
-- 세션 A (일반 SELECT는 갭 락이 걸리지 않음. FOR UPDATE/FOR SHARE 같은 잠금 읽기를 사용해야 함)
SELECT * FROM tab WHERE age BETWEEN 20 AND 30 FOR UPDATE;  -- 3건, age 20~30 사이에 갭 락 설정
-- 세션 B
INSERT INTO tab (age) VALUES (25);  -- 갭 락에 걸려서 대기, 삽입 불가

-- PostgreSQL (REPEATABLE READ): 삽입은 허용하되 안 보이게 함 (낙관적 방식)
-- 세션 A
SELECT * FROM tab WHERE age BETWEEN 20 AND 30 FOR UPDATE;  -- 3건, 갭 락 없음
-- 세션 B
INSERT INTO tab (age) VALUES (25);  -- 갭 락이 없으므로 바로 삽입 성공
-- 세션 A
SELECT * FROM tab WHERE age BETWEEN 20 AND 30;  -- 여전히 3건 (MVCC 스냅샷이므로 안 보임)
```

PostgreSQL은 갭 락 없이도 MVCC 스냅샷으로 팬텀 리드를 방지한다. SERIALIZABLE 격리 수준에서는 갭 락 대신 SSI(Serializable Snapshot Isolation)를 사용하여 충돌이 감지되면 트랜잭션을 롤백시킨다.

갭 락이 없는 PostgreSQL에서 범위 충돌을 방지하는 방법 (예: 같은 시간대 미팅 예약 방지):

1. **EXCLUDE 제약 조건** (권장): 스키마에 명시적으로 "겹침 불가"를 선언하는 방식. btree_gist 확장이 필요하다.
   ```sql
   ALTER TABLE meetings
   ADD CONSTRAINT no_overlap
   EXCLUDE USING gist (tsrange(start_time, end_time) WITH &&);
   ```
2. **Advisory Lock**: 특정 리소스(예: 회의실)에 대해 직렬화하여 처리.
   ```sql
   BEGIN;
   SELECT pg_advisory_xact_lock(hashtext('meeting_room_1'));
   SELECT * FROM meetings WHERE start_time BETWEEN '09:00' AND '10:00';
   INSERT INTO meetings ...;
   COMMIT;
   ```
3. **SERIALIZABLE 격리 수준**: 충돌 감지 시 serialization failure가 발생하며, 애플리케이션에서 재시도 로직이 필요하다.

| 방식 | 장점 | 단점 |
|---|---|---|
| InnoDB 갭 락 | 자동으로 동작 | 동시성 저하 (대기 발생) |
| PostgreSQL EXCLUDE 제약 | 선언적, DB가 보장 | btree_gist 확장 필요 |
| Advisory Lock | 간단, 유연 | 애플리케이션이 규칙을 지켜야 함 |
| SERIALIZABLE | 표준적 | 충돌 시 재시도 로직 필요 |

#### 5.3.1.3 넥스트 키 락

- 레코드 락 + 갭 락의 결합이다. InnoDB의 REPEATABLE READ 격리 수준에서 잠금이 필요한 쿼리(`UPDATE`, `DELETE`, `SELECT ... FOR UPDATE/SHARE`)의 기본 잠금 방식이다. READ COMMITTED에서는 레코드 락만 걸린다.

  | 격리 수준 | 기본 잠금 방식 |
  |---|---|
  | READ COMMITTED | 레코드 락만 (갭 락 없음) |
  | REPEATABLE READ | 넥스트 키 락 (레코드 + 갭) |

- 갭이란 인덱스에 정렬되어 저장된 레코드 사이의 빈 공간이다. 예를 들어 age 인덱스에 20, 25, 30이 있을 때:
  ```
  [20] ---- [25] ---- [30]
        ↑         ↑
     갭(20~25)  갭(25~30)
  ```
  `UPDATE ... WHERE age = 25`를 REPEATABLE READ에서 실행하면, age=25 레코드 자체 + 앞쪽 갭(20~25) + 뒤쪽 갭(25~30)이 잠긴다. 다른 세션이 age=23이나 age=27을 삽입하려면 대기해야 한다.

- age에 인덱스가 없으면 클러스터 인덱스(PK) 풀스캔이 되고, 모든 레코드 + 모든 갭에 넥스트 키 락이 걸려 사실상 테이블 전체 잠금이 된다. INSERT도 전부 차단된다.

- 갭 락/넥스트 키 락의 주목적은 소스 → 레플리카 복제 시 동일한 결과를 보장하기 위함이다. 스캔이란 인덱스를 순서대로 하나씩 따라가며 조건에 맞는 레코드를 찾는 것인데, 한 번에 모든 레코드를 동시에 처리하는 게 아니라 순차적으로 방문한다. 갭 락이 없으면 스캔 도중 이미 지나간 위치에 새 레코드가 삽입될 수 있다.
  ```
  -- 소스 서버, age 인덱스: [20, 25, 30]
  -- 세션 A: DELETE FROM tab WHERE age BETWEEN 20 AND 30;
  -- 세션 B: INSERT INTO tab (age) VALUES (23);

  세션 A: age=20 삭제 → age=25 삭제 → (아직 30은 안 봄)
  세션 B:                              age=23 삽입 (이미 스캔 지나간 위치)
  세션 A:                                          age=30 삭제
  -- 결과: 소스에는 age=23이 남음

  -- 바이너리 로그(STATEMENT)에는 커밋 순서대로 기록:
  -- INSERT 23 → DELETE 20~30
  -- 레플리카에서: INSERT 먼저 → 23 생김 → DELETE 실행 → 23까지 삭제됨
  -- → 소스에는 23 있고, 레플리카에는 23 없음 (데이터 불일치)
  ```
  갭 락이 있으면 세션 A의 DELETE가 20~30 갭을 잠그므로 세션 B의 INSERT가 대기하게 되어 순서가 보장된다. 바이너리 로그란 MySQL에서 데이터 변경 내역을 기록하는 로그로, 복제와 복구에 사용된다. 기록 방식에 따라 두 가지 포맷이 있다:
  - **STATEMENT 포맷**: 실행한 SQL 문 자체를 기록(예: `DELETE FROM tab WHERE age BETWEEN 20 AND 30` 한 줄). 레플리카에서 SQL을 그대로 다시 실행한다. 로그 크기가 작지만, 소스와 레플리카의 데이터 상태가 다르면 다른 결과가 나올 수 있어 갭 락이 필요하다.
  - **ROW 포맷**: 실제로 어떤 행이 변경됐는지 결과를 행 단위로 기록(예: id=1 삭제, id=2 삭제, ...). 레플리카에서 해당 행을 직접 변경한다. SQL을 다시 실행하지 않으므로 결과가 항상 동일하여 갭 락의 필요성이 줄어든다. 대량의 행이 변경되는 경우 행 수만큼 기록해야 하므로 로그 크기가 클 수 있다.

  넥스트 키 락과 갭 락으로 인해 데드락이나 대기가 자주 발생하므로, 가능하다면 ROW 포맷으로 변경하여 갭 락을 줄이는 것이 좋다. MySQL 8.0에서는 ROW 포맷의 바이너리 로그가 기본 설정으로 변경됐다.

##### PostgreSQL과의 비교

MySQL은 로그가 두 개로 분리되어 있다: InnoDB의 **redo log**(장애 복구용, 물리적 로그)와 **바이너리 로그**(복제 + 시점 복구용, 논리적 로그). 반면 PostgreSQL은 **WAL(Write-Ahead Log)** 하나로 장애 복구, 복제, 시점 복구를 모두 처리한다. WAL이란 "Write-Ahead Log", 즉 데이터를 실제 디스크에 쓰기 전에 로그를 먼저 기록한다는 뜻이다. 데이터 변경은 항상 shared buffers(메모리) 위에서 일어난다. 해당 페이지가 shared buffers에 없으면 디스크에서 먼저 올린 뒤, shared buffers 안에서 직접 수정한다. 수정된 페이지는 "dirty page"로 표시되고, WAL에 변경 내용을 기록한 뒤 클라이언트에 성공을 응답한다. 실제 데이터 파일(디스크)에는 나중에 체크포인트 시 dirty page를 기록한다. 서버가 죽어도 WAL에 기록이 남아있으니 복구할 수 있다.

- 조회 시 shared buffers에 해당 페이지가 있으면 바로 읽고, 없으면 디스크에서 읽어서 shared buffers에 올린 뒤 읽는다. 어떤 페이지가 올라와 있는지는 해시 테이블로 관리하며, O(1)로 빠르게 조회한다. 데이터는 **페이지(8KB 블록) 단위**로 관리되므로, 페이지 안의 일부 행만 메모리에 있고 나머지는 디스크에 있는 상황은 발생하지 않는다. 페이지를 올린 뒤 그 안의 행들에 대해 WHERE 조건을 적용하여 결과를 반환한다.
- WAL과 shared buffers의 관계: shared buffers는 데이터를 읽고 쓰는 **작업 공간**(메모리, 성능을 위한 것)이고, WAL은 변경 내용을 기록하는 **안전장치**(로그, 메모리가 날아가도 복구 가능). 둘은 독립적으로 디스크에 기록된다. WAL은 변경 즉시 디스크에 기록(fsync)되고, shared buffers는 나중에 체크포인트 시 디스크에 기록된다. 서버가 죽으면 shared buffers(메모리)는 날아가지만 WAL(디스크)은 남아있으므로, WAL을 읽어서 복구한다.

MySQL의 바이너리 로그(ROW 포맷)가 "employees 테이블의 id=1 행을 삭제" 같은 논리적 수준으로 기록하는 반면, WAL은 "파일 base/16384/16385의 블록 3, 오프셋 160에서 8바이트를 0x4A6F686E으로 변경" 같은 물리적 수준(디스크 블록 위치 + 바이트 값)으로 기록한다. 테이블 이름이나 컬럼 이름, 행 ID 같은 개념이 없다. 디스크 변경을 그대로 복사하는 것이라 레플리카에서 해석이 필요 없고 데이터 불일치 가능성 자체가 없으므로, 복제 일관성을 위한 갭 락이 애초에 필요 없다.

#### 5.3.1.4 자동 증가 락

- `AUTO_INCREMENT` 컬럼에 값을 채번할 때 사용하는 테이블 수준의 잠금이다. 동시에 여러 INSERT가 들어와도 중복/빈 번호 없이 순서대로 할당한다.
- 트랜잭션과 무관하게 INSERT 또는 REPLACE 문에서 `AUTO_INCREMENT` 값을 가져오는 순간에만 잠금이 걸렸다가 즉시 해제된다 (트랜잭션 끝까지 유지되지 않음).
- 롤백 시에도 이미 할당된 번호는 소모된다 (빈 번호 발생).

##### 용어: 락(Lock)과 래치(Latch)

자동 증가 락의 모드별 설명에서 "래치(뮤텍스)"라는 용어가 등장하는데, **락과 래치는 다른 개념**이다.

| | 락 (Lock) | 래치 (Latch) |
|---|---|---|
| 보호 대상 | 논리적 자원 (행, 테이블) | 메모리 자료구조 (카운터, 버퍼) |
| 유지 시간 | 트랜잭션 단위 (길다) | 수십 나노초 ~ 수 마이크로초 (매우 짧다) |
| 구현 | 락 매니저, 큐 대기, 데드락 감지 | 뮤텍스(mutex) / 스핀락(spinlock) |
| 비용 | 무거움 (대기열, 컨텍스트 스위치) | 가벼움 (대부분 즉시 획득) |

자동 증가 값을 가져오는 일은 메모리 안의 카운터 변수를 +1 하는 짧은 작업이다. 트랜잭션 단위로 보호할 필요가 없고 동시 접근만 막으면 되므로, 무거운 자동 증가 락 대신 메모리 보호용 래치(뮤텍스)로 충분하다.

###### 뮤텍스(Mutex)와 스핀락(Spinlock)

래치는 데이터베이스 용어이고, 뮤텍스/스핀락은 그것을 실제로 구현하는 OS/CPU 수준의 동기화 도구다.

```
래치 (DB 추상 개념)
   ↓ 실제 구현
뮤텍스 또는 스핀락 (OS/CPU 메커니즘)
```

- **뮤텍스(Mutex, MUTually EXclusive)**: 한 번에 한 스레드만 진입 가능한 잠금. 다른 스레드가 잠금을 잡고 있으면 OS에 "나 재워줘"라고 요청하고 잠든다. 잠금이 풀리면 OS 스케줄러가 깨운다. 컨텍스트 스위치 비용이 들지만, 대기가 길어질 때 CPU를 다른 일에 양보할 수 있어 효율적이다.

  ```
  스레드 A: mutex.lock()    → 획득, 임계 구역 진입
  스레드 B: mutex.lock()    → A가 갖고 있음 → 잠들기 (OS 대기 큐에 진입)
  스레드 A: mutex.unlock()  → OS가 B를 깨움
  스레드 B: 깨어남 → 획득, 진입
  ```

- **스핀락(Spinlock)**: 잠금이 풀릴 때까지 CPU에서 빙빙 돌며(spin) 재시도하는 잠금. 잠들지 않으니 컨텍스트 스위치 비용이 없다. 대기 시간이 매우 짧을 거라 확신될 때만 효율적이며, 길어지면 CPU를 낭비한다.

  ```
  스레드 A: lock = 1 (획득)
  스레드 B: while (lock == 1) { /* CPU 점유한 채 반복 검사 */ }
  스레드 A: lock = 0 (해제)
  스레드 B: 즉시 lock = 1, 진입
  ```

| 상황 | 적합한 도구 | 이유 |
|---|---|---|
| 임계 구역이 길다 (수십 μs 이상) | 뮤텍스 | 잠들어 CPU 양보가 이득 |
| 임계 구역이 매우 짧다 (수 ns ~ μs) | 스핀락 | 컨텍스트 스위치 비용 > 회전 비용 |
| 멀티프로세서 환경의 짧은 임계 구역 | 스핀락 | 다른 CPU에서 곧 풀어줄 것 |
| 단일 CPU 환경 | 뮤텍스 | 회전해봐야 잠금 푼 스레드가 못 도니 의미 없음 |

실제로는 두 가지를 섞은 **adaptive mutex**(InnoDB가 사용하는 방식)도 흔하다. 먼저 짧게 스핀해보고, 못 잡으면 잠드는 방식이다. 자동 증가 카운터 보호처럼 수 나노초 단위의 작업에는 스핀락 또는 adaptive mutex가 흔히 쓰인다.

##### innodb_autoinc_lock_mode

자동 증가 락의 동작 방식은 `innodb_autoinc_lock_mode` 시스템 변수로 제어한다. MySQL 8.0의 기본값은 `2`이며, 모드별 동작은 다음과 같다.

- **`innodb_autoinc_lock_mode=0` (전통 모드, Traditional)**: MySQL 5.0과 동일한 방식. 모든 INSERT 문이 자동 증가 락을 사용하며, INSERT가 끝날 때까지 락이 유지되어 다른 커넥션의 INSERT가 대기한다.
- **`innodb_autoinc_lock_mode=1` (연속 모드, Consecutive)**: MySQL 서버가 INSERT 될 레코드의 건수를 미리 예측할 수 있는 단순 INSERT(`INSERT INTO ... VALUES (...), (...)` 등)에서는 자동 증가 락 대신 훨씬 가벼운 래치(뮤텍스)를 사용한다. 래치는 자동 증가 값을 가져오는 짧은 순간만 잠금을 걸고 즉시 해제한다. 반면 `INSERT ... SELECT`처럼 건수를 미리 예측할 수 없는 대량 INSERT에서는 MySQL 5.0과 동일한 자동 증가 락을 사용한다. 이때 InnoDB는 자동 증가 값을 한 번에 여러 개 할당받아 사용하므로, 해당 INSERT 문 내부에서는 레코드 사이에 빈 번호 없이 연속된 값이 보장된다. 다만 한 번에 할당받은 값이 남으면 폐기되므로, 대량 INSERT **이후** 실행되는 INSERT에서는 번호가 누락될 수 있다.
- **`innodb_autoinc_lock_mode=2` (인터리빙 모드, Interleaved)**: 자동 증가 락을 절대 걸지 않고 항상 경량 래치만 사용한다. 동시성은 가장 높지만, 하나의 INSERT 문 내에서도 연속된 자동 증가 값을 보장하지 않는다 (다른 커넥션의 채번과 섞일 수 있다). 유일성만 보장한다. `INSERT ... SELECT` 같은 대량 INSERT 도중에도 다른 커넥션이 INSERT를 수행할 수 있다. 단, STATEMENT 포맷 바이너리 로그를 사용하는 복제 환경에서는 소스와 레플리카의 자동 증가 값이 달라질 수 있으므로 ROW 포맷이 전제되어야 한다 (자세한 시나리오는 아래 참고).

| 모드 | 락 방식 | 단일 INSERT 연속성 | 대량 INSERT 동시성 |
|---|---|---|---|
| 0 (전통) | 항상 자동 증가 락 | 보장 | 낮음 (대기) |
| 1 (연속) | 단순 INSERT는 래치, 대량 INSERT는 자동 증가 락 | 보장 | 중간 |
| 2 (인터리빙) | 항상 래치 | 보장하지 않음 (유일성만 보장) | 높음 |

##### 모드 2 + STATEMENT 포맷이 위험한 이유

STATEMENT 포맷 바이너리 로그는 실행한 SQL 문장 자체만 기록하고 **자동 증가 값은 기록하지 않는다**. 레플리카는 SQL을 다시 실행하면서 자동 증가 값을 새로 채번하는데, 모드 2에서는 동시에 실행된 INSERT의 채번이 섞이므로 소스와 레플리카의 값이 어긋난다.

```
-- 소스 서버에서 동시 실행 (모드 2, 인터리브 발생)
세션 A: INSERT INTO t SELECT ...  (3건)   → id = 1, 3, 5 할당
세션 B: INSERT INTO t SELECT ...  (3건)   → id = 2, 4, 6 할당

-- STATEMENT 바이너리 로그 기록 내용:
"INSERT INTO t SELECT ..."   (A의 SQL)
"INSERT INTO t SELECT ..."   (B의 SQL)
-- 실제 할당된 id 값은 기록되지 않는다.

-- 레플리카에서 로그를 순차 재실행:
A의 INSERT 먼저 실행 → id = 1, 2, 3 할당
B의 INSERT 다음 실행 → id = 4, 5, 6 할당

→ 소스: A가 1,3,5  /  레플리카: A가 1,2,3  → 데이터 불일치
```

ROW 포맷이면 실제 행의 자동 증가 값(`id=1, id=3, id=5` 등)이 그대로 기록되므로 레플리카에서도 동일한 값이 들어가 불일치가 없다. MySQL 8.0부터 ROW 포맷이 기본값이라 사실상 자동 충족된다.

##### PostgreSQL과의 비교

PostgreSQL의 자동 증가는 **시퀀스(Sequence)**라는 별도의 **데이터베이스 객체**로 처리된다.

###### 데이터베이스 객체란

데이터베이스 객체란 데이터베이스 안에서 이름을 가지고 독립적으로 존재하는 1급 요소를 말한다. 시스템 카탈로그(메타데이터 테이블)에 등록되어 `CREATE` / `ALTER` / `DROP`으로 관리하고, `GRANT` / `REVOKE`로 권한을 부여할 수 있는 것들이다.

핵심은 **"정의(definition)"** 와 **"데이터"** 의 구분이다. 객체는 구조와 동작의 정의이고, 행이나 값은 그 안에 담긴 데이터다.

| 객체이다 | 객체가 아니다 |
|---|---|
| 테이블, 인덱스, 뷰, 시퀀스, 함수, 트리거 | 테이블 안의 행 |
| 컬럼 (이름·타입·제약을 가진 정의 단위) | 컬럼의 특정 값 |
| 스키마, 데이터베이스, 역할 | 시퀀스가 발급한 숫자 |

MySQL과 PostgreSQL의 객체 종류는 거의 비슷하지만(테이블/인덱스/뷰/함수/트리거 등), PostgreSQL은 **시퀀스**가 1급 객체로 따로 존재하는 반면 MySQL에는 시퀀스가 없고 `AUTO_INCREMENT`가 테이블 컬럼의 속성으로만 존재한다.

###### 시퀀스 객체

비유하자면 MySQL은 "각 가게에 자기만의 번호표 기계가 붙박이로 설치됨"이고, PostgreSQL은 "번호표 기계가 광장에 따로 있고 가게들이 그걸 가져다 씀"이다.

```sql
-- 명시적 시퀀스 생성과 사용
CREATE SEQUENCE order_seq START WITH 1 INCREMENT BY 1 CACHE 20;
SELECT nextval('order_seq');   -- 1
SELECT nextval('order_seq');   -- 2
SELECT currval('order_seq');   -- 2 (현재 세션의 마지막 값)
SELECT setval('order_seq', 100);  -- 강제로 100으로 설정

-- SERIAL은 내부적으로 시퀀스 객체를 자동 생성
CREATE TABLE users (
    id SERIAL PRIMARY KEY,    -- users_id_seq 시퀀스 자동 생성
    name TEXT
);
```

시퀀스가 별도 객체이기 때문에 생기는 특성:

- **여러 테이블이 한 시퀀스 공유 가능** — 글로벌 ID처럼 사용할 수 있다.
- **트랜잭션 밖에서 동작** — `nextval()`은 트랜잭션을 따르지 않는다. 롤백해도 값이 되돌아오지 않는다 (MySQL `AUTO_INCREMENT`와 동일). 다른 세션의 동시 채번을 막지 않으려고 일부러 트랜잭션 의미론에서 분리한 것이다.
- **세션별 캐시(`CACHE n`)** — 각 세션이 시퀀스에서 한 번에 n개를 미리 받아 로컬에 보관한다. 그 다음 `nextval()`은 시퀀스 객체를 건드리지 않고 로컬 캐시에서 즉시 반환한다. 세션 종료 시 남은 값은 폐기되어 빈 번호가 발생한다.
- **잠금이 시퀀스 객체 단위** — 테이블이 아니라 시퀀스 객체에만 짧은 spinlock을 건다. 테이블 DML과 완전히 독립이라 동시성이 매우 높다.

###### CACHE 옵션의 동작

`CACHE n`은 **한 번에 가져올 묶음 크기**이지 총량 상한이 아니다. 캐시가 소진되면 시퀀스 객체에 다시 접근해 또 n개를 받아오며, 필요한 만큼 반복된다.

```
세션 A에서 100개 채번 (CACHE 20)
[1회차] nextval() → 캐시 비어있음
                  → 시퀀스 객체에 락 잡고 1~20 가져옴 → 캐시: [2..20], 반환: 1
        nextval() × 19번 → 캐시에서 즉시 반환 (시퀀스 객체 안 건드림)
[2회차] nextval() → 캐시 비어있음 → 시퀀스 객체에서 21~40 받음 → 반환: 21
        ... 캐시에서 22~40
[3~5회차] 41~60, 61~80, 81~100

→ 시퀀스 객체 락 빈도: 100번이 아니라 5번
→ 나머지 95번은 메모리(로컬 캐시)에서 즉시 처리
```

`CACHE`의 목적은 시퀀스 객체에 거는 락의 빈도를 줄여 동시성을 높이는 것이다.

| 항목 | 값 |
|---|---|
| `CACHE n`의 의미 | 한 번에 미리 받아오는 묶음 크기 |
| 한 세션이 받을 수 있는 총량 | 무제한 (캐시 소진 시 계속 보충) |
| `CACHE 20`으로 1,000,000개 채번 시 | 시퀀스 객체 접근 50,000번, 캐시 히트 950,000번 |
| 트레이드오프 | n↑ → 시퀀스 락 빈도↓ (동시성↑), 세션 종료 시 폐기 값↑ (빈 번호↑) |

`CACHE 1`(기본값)이면 매번 시퀀스 객체에 접근하므로 빈 번호는 없지만 동시성이 낮다. `CACHE 1000`이면 시퀀스 객체 접근은 1/1000로 줄지만 세션 비정상 종료 시 최대 999개의 번호가 사라진다.

**캐시로 인한 인터리브와 단조성 깨짐**: 세션별로 따로 캐시를 가져가므로 세션 간 채번 순서가 섞인다.

```
세션 A: 시퀀스에서 1~20 캐시로 가져감
세션 B: 시퀀스에서 21~40 캐시로 가져감

세션 B가 먼저 INSERT → id=21이 먼저 들어감
세션 A가 나중에 INSERT → id=1이 나중에 들어감

→ 테이블에는 id=1(A)과 id=21(B)이 시간 역순으로 존재
→ id가 큰 값이 시간상 더 먼저일 수 있음 (단조성 깨짐)
```

그래서 PostgreSQL은 단일 INSERT 내 연속성도, 시간순 ID 순서도 보장하지 않으며 **유일성만 보장**한다.

###### 시퀀스 공유: 기본 동작 vs 글로벌 ID

"여러 테이블이 시퀀스를 공유 가능"은 선택지이지 기본값이 아니다. 보통은 `SERIAL`이나 `IDENTITY`를 쓰면 **테이블마다 시퀀스가 자동으로 따로 생성**되어 각자 1부터 시작한다 (MySQL `AUTO_INCREMENT`와 동일한 동작).

```sql
-- 기본 동작: 테이블마다 별도 시퀀스
CREATE TABLE users (id SERIAL PRIMARY KEY, name TEXT);
CREATE TABLE orders (id SERIAL PRIMARY KEY, amount INT);
-- 내부적으로 users_id_seq, orders_id_seq 두 개가 따로 생성된다.

INSERT INTO users (name) VALUES ('a');   -- users.id  = 1
INSERT INTO users (name) VALUES ('b');   -- users.id  = 2
INSERT INTO orders (amount) VALUES (10); -- orders.id = 1  ← 자기 시퀀스, 1부터
INSERT INTO orders (amount) VALUES (20); -- orders.id = 2
```

공유는 **명시적으로 한 시퀀스를 가리키도록 정의할 때만** 발생한다. 여러 테이블의 ID가 절대 겹치지 않게 만들고 싶을 때(글로벌 ID) 사용한다.

```sql
-- 글로벌 ID: 두 테이블이 한 시퀀스를 공유
CREATE SEQUENCE global_id_seq;

CREATE TABLE orders (
    id BIGINT PRIMARY KEY DEFAULT nextval('global_id_seq'),
    amount INT
);
CREATE TABLE refunds (
    id BIGINT PRIMARY KEY DEFAULT nextval('global_id_seq'),
    reason TEXT
);

INSERT INTO orders  (amount) VALUES (10);     -- id = 1
INSERT INTO refunds (reason) VALUES ('x');    -- id = 2  ← orders와 같은 시퀀스를 보니까 2
INSERT INTO orders  (amount) VALUES (20);     -- id = 3
INSERT INTO refunds (reason) VALUES ('y');    -- id = 4
```

마이크로서비스 간 메시지 라우팅처럼 "이 ID가 어느 테이블 것이든 상관없이 유일해야 한다"를 보장하고 싶을 때 유용하다. MySQL은 시퀀스 자체가 없어 이런 패턴이 불가능하다.

| 사용 방식 | 결과 |
|---|---|
| `CREATE TABLE t (id SERIAL ...)` (기본) | 테이블마다 시퀀스 자동 생성 → 각자 1부터 |
| `CREATE SEQUENCE s; ... DEFAULT nextval('s')` (수동) | 가리키는 모든 테이블이 같은 카운터 → 글로벌 ID |

###### 정리 비교

| | InnoDB | PostgreSQL |
|---|---|---|
| 자동 증가 | `AUTO_INCREMENT` (테이블 컬럼 속성) | `SERIAL` / `IDENTITY` (시퀀스 객체 기반) |
| 잠금 방식 | 모드에 따라 자동 증가 락 또는 래치 | 시퀀스 객체의 spinlock (항상 경량) |
| 연속성 | 모드 0/1은 단일 문장 내 연속 보장, 모드 2는 보장 안 함 | 보장하지 않음 (세션별 캐시로 인해 인터리브 발생) |
| 다중 테이블 공유 | 불가 | 가능 (한 시퀀스를 여러 테이블이 참조) |
| 롤백 시 | 번호 소모 | 번호 소모 (동일) |

PostgreSQL은 `nextval()` 호출 시 시퀀스 객체에 짧은 spinlock만 잡아 다음 값을 가져오므로 InnoDB의 `innodb_autoinc_lock_mode=2`와 가장 유사하다. 시퀀스는 `CACHE n` 옵션으로 세션별로 값을 미리 가져올 수 있는데, 이 경우 세션 간 채번이 인터리브되어 단일 INSERT 문장 내에서도 연속성이 깨질 수 있다. 즉 PostgreSQL은 모드 선택 없이 항상 "인터리빙 모드"로 동작하며, 단일 문장 내 연속 채번이 필요한 경우 `CACHE 1`로 설정하거나 애플리케이션에서 별도 처리해야 한다.

### 5.3.2 인덱스와 잠금

InnoDB의 잠금은 **레코드(행) 자체가 아니라 인덱스 엔트리에 걸리는 방식**으로 처리된다. 클러스터형 인덱스 구조(PK가 곧 데이터)와 맞물려 인덱스 설계가 곧 동시성 설계가 된다.

##### "인덱스를 잠근다"의 의미

InnoDB 테이블은 다음과 같이 구성된다.

- Primary Key (클러스터형 인덱스) = 인덱스이자 실제 데이터
- Secondary Index = (인덱스 키 → PK 값) 매핑

`UPDATE` / `DELETE` / `SELECT ... FOR UPDATE` 시 WHERE 조건과 매칭된 **인덱스 엔트리**에 락이 걸린다. 매칭된 행 자체에 직접 거는 게 아니다.

##### 인덱스가 없으면 모든 레코드가 잠긴다

WHERE 조건의 컬럼에 인덱스가 없으면 InnoDB는 클러스터 인덱스(PK)를 풀스캔하며, 방문하는 모든 인덱스 엔트리(=모든 행)에 락을 건다.

```sql
-- name 컬럼에 인덱스 없음
UPDATE users SET grade = 'A' WHERE name = 'kim';

-- 동작:
-- 1. PK 풀스캔으로 모든 행을 방문
-- 2. 방문할 때마다 인덱스 엔트리에 락
-- 3. name = 'kim'이 아니면 락 해제 (REPEATABLE READ에선 유지될 수 있음)
-- → 일시적으로 테이블 전체가 잠긴다.
```

REPEATABLE READ에서는 조건 불일치 락이 즉시 해제되지 않고 트랜잭션 끝까지 유지될 수 있다. 그래서 인덱스 설계가 곧 동시성 설계가 된다.

##### 세컨더리 인덱스로 조회해도 PK가 잠긴다

```sql
CREATE INDEX idx_email ON users (email);
SELECT * FROM users WHERE email = 'a@b.com' FOR UPDATE;

-- InnoDB 동작:
-- 1. idx_email에서 'a@b.com' 인덱스 엔트리에 락
-- 2. 그 엔트리의 PK 값을 따라가 클러스터 인덱스의 PK 엔트리에도 락
-- → 세컨더리 인덱스 + 클러스터 인덱스 양쪽 잠금
```

다른 트랜잭션이 같은 PK 행을 다른 인덱스 경로로 접근해도 결국 클러스터 인덱스에서 만나 대기한다.

##### 갭 락도 인덱스 기반

5.3.1.2에서 다룬 갭 락/넥스트 키 락도 **인덱스 엔트리 사이의 갭**에 거는 잠금이다. 데이터 자체에 거는 게 아니므로, 같은 영역이라도 어느 인덱스에서 보느냐에 따라 다른 갭이 된다.

##### PostgreSQL과의 비교

PostgreSQL은 락을 **인덱스가 아니라 행(튜플) 자체에** 표시한다. 락의 저장 위치와 MVCC 동작 방식이 InnoDB와 근본적으로 다르다.

###### 락 정보가 행에 저장된다

보통 락은 별도의 자료구조(락 매니저, 메모리 안의 해시 테이블)에 "행 ID=42는 트랜잭션 7이 잠금" 식으로 저장한다. InnoDB가 이 방식이며, 락이 행과 분리된 곳에 산다.

PostgreSQL은 튜플(행) 헤더에 락 정보를 직접 적어 둔다.

```
PostgreSQL의 튜플 헤더:
┌─────────────────────────────────────────────────────┐
│ xmin     │ 이 행을 만든 트랜잭션 ID                  │
│ xmax     │ 이 행을 삭제/수정한 트랜잭션 ID            │ ← 락 정보가 들어감
│ infomask │ 비트 플래그 (xmax가 락이냐 삭제냐 등)      │
├─────────────────────────────────────────────────────┤
│ 실제 컬럼 값들                                       │
└─────────────────────────────────────────────────────┘
```

`SELECT ... FOR UPDATE`로 행에 락을 걸면 그 행의 `xmax`에 **잠금을 건 트랜잭션 ID**를 써넣고, `infomask` 비트로 "이건 삭제가 아니라 락"임을 표시한다. 즉 행 자체가 자기 락 상태를 들고 다닌다. (행 락이 매우 많아지면 PostgreSQL도 별도 락 매니저로 옮기지만, 시작점은 항상 행이다.)

###### MVCC로 읽기/쓰기 충돌 회피

MVCC(Multi-Version Concurrency Control, 다중 버전 동시성 제어)는 같은 행에 대해 여러 버전을 동시에 유지하는 방식이다.

```
트랜잭션 A: UPDATE users SET name = 'Bob' WHERE id = 1;
            (아직 커밋 안 함, 행에 락 걸려 있음)

이 시점에 두 버전이 공존:
  버전 1 (옛것): name = 'Alice'  ← 다른 트랜잭션이 이걸 봄
  버전 2 (새것): name = 'Bob'    ← A만 봄

트랜잭션 B: SELECT name FROM users WHERE id = 1;
  → 버전 1을 읽음 → 'Alice' 반환
  → A가 락을 잡고 있어도 B는 안 막힘
```

쓰는 트랜잭션이 락을 갖고 있어도 읽는 트랜잭션은 옛 버전을 보므로 대기하지 않는다. 이게 "읽기/쓰기 충돌 회피"의 의미다. InnoDB도 MVCC를 쓰지만, 명시적 락(`FOR UPDATE`, `UPDATE`)이 인덱스 엔트리에 걸려 차단을 만든다는 차이가 있다.

###### 인덱스 없는 UPDATE의 동작 차이

같은 쿼리를 두 DB에서 실행하면 결과가 완전히 다르다.

```sql
-- name 컬럼에 인덱스 없음
UPDATE users SET grade = 'A' WHERE name = 'kim';
```

**InnoDB**: 클러스터 인덱스 풀스캔 + 방문하는 모든 인덱스 엔트리에 락. 다른 트랜잭션이 다른 행을 UPDATE하거나 INSERT해도 락 때문에 대기. 사실상 테이블 전체 잠금.

**PostgreSQL**: 풀스캔하지만 읽기는 MVCC로 처리하므로 락 안 검. `name = 'kim'`인 행을 발견했을 때만 그 행의 `xmax`에 락 표시. 다른 행은 락 없음 → 다른 트랜잭션 자유롭게 동작.

| | InnoDB | PostgreSQL |
|---|---|---|
| 풀스캔 여부 | O | O |
| 락이 걸리는 행 | 방문한 전부 | 매칭된 것만 |
| 원칙 | 방문 = 잠금 | 매칭 = 잠금 |

핵심은 PostgreSQL의 풀스캔이 **MVCC 스냅샷 읽기**라는 점이다. 행의 `xmin`/`xmax`를 보고 "이 트랜잭션에서 보이는 버전인지" 판단만 할 뿐, 락 정보를 쓰지 않는다. 락은 매칭된 행에 도달했을 때만 `xmax`에 새겨진다.

###### 다른 트랜잭션이 락을 걸려고 할 때의 동작

위 UPDATE 실행 중에 다른 트랜잭션이 어떻게 영향을 받는지 시나리오별로 본다 (트랜잭션 A: `UPDATE ... WHERE name = 'kim'` 실행 중, kim 행 3개가 잠긴 상태).

**시나리오 1**: 다른 행에 락을 걸려는 경우

```sql
-- 트랜잭션 B
UPDATE users SET grade = 'B' WHERE name = 'park';
-- → park 행에는 락 없음 → 즉시 진행 (InnoDB라면 대기)

SELECT * FROM users WHERE name = 'lee' FOR UPDATE;
-- → lee 행에도 락 없음 → 즉시 락 획득 (InnoDB라면 대기)
```

**시나리오 2**: 같은 행에 락을 걸려는 경우

```sql
-- 트랜잭션 B
UPDATE users SET email = 'x' WHERE name = 'kim';
-- → kim 행의 xmax를 보니 A가 잠금 중 → 대기 (A가 commit/rollback까지)

SELECT * FROM users WHERE name = 'kim' FOR UPDATE;
-- → 동일하게 대기
```

매칭된 행이 겹치면 PostgreSQL도 당연히 대기한다. 락은 락이니까.

**시나리오 3**: 단순 SELECT (락 없음)

```sql
-- 트랜잭션 B
SELECT * FROM users WHERE name = 'kim';
-- → MVCC로 A 커밋 전 버전(이전 grade)을 읽음 → 안 막힘
```

쓰기 락이 읽기를 막지 않는다. MVCC의 핵심 효과다.

| 시나리오 | InnoDB | PostgreSQL |
|---|---|---|
| 다른 행 UPDATE / FOR UPDATE | 대기 | 즉시 진행 |
| 같은 행 UPDATE / FOR UPDATE | 대기 | 대기 (동일) |
| 단순 SELECT | 안 막힘 (MVCC) | 안 막힘 (MVCC) |

PostgreSQL의 동시성 영향은 **매칭된 행에만 국소적으로 발생**한다. 매칭 안 된 행을 건드리는 다른 트랜잭션은 영향받지 않는다.

###### 동시성 ≠ 성능

여기서 핵심은 **동시성과 성능은 다른 문제**라는 것이다.

| 개념 | 의미 | 측정 |
|---|---|---|
| 동시성 | 여러 트랜잭션이 동시에 일할 수 있는가 | "다른 트랜잭션이 막히나?" |
| 성능 | 한 쿼리가 얼마나 빠른가 | "이 쿼리가 몇 ms 걸리나?" |

PostgreSQL에서 인덱스 없는 UPDATE는 **성능은 나쁘지만 동시성은 멀쩡**하다. 테이블 전체를 읽어야 하니 느리지만, 다른 트랜잭션을 막지는 않는다.

| | 인덱스 없는 UPDATE의 결과 |
|---|---|
| InnoDB | 성능 망함 + **동시성도 망함** (모든 인덱스 엔트리 락) |
| PostgreSQL | 성능 망함, 하지만 **동시성은 멀쩡** (락은 매칭된 행만) |

###### 실무적 함의

- **InnoDB**: WHERE 조건에 인덱스가 없으면 동시성이 무너진다. UPDATE/DELETE의 WHERE는 반드시 인덱스를 타도록 설계해야 한다.
- **PostgreSQL**: 인덱스가 없어도 락 자체는 매칭된 행에만 걸린다. 다만 풀스캔으로 인한 성능 문제는 별개다.

###### 전체 비교표

| | InnoDB | PostgreSQL |
|---|---|---|
| 락 대상 | 인덱스 엔트리 | 힙 튜플(행) 자체 |
| 락 표현 위치 | 락 매니저 + 인덱스 페이지 | 튜플 헤더의 `xmax`, `infomask` 비트 |
| 인덱스 없는 WHERE | 풀스캔 + 모든 인덱스 엔트리 잠금 | 풀스캔하지만 락은 매칭된 행에만 |
| 갭 락 | 인덱스 갭에 존재 | 없음 (MVCC 스냅샷 + SSI로 대체) |
| MVCC 읽기 | 지원 (락과 무관한 스냅샷 읽기) | 지원 (동일) |

### 5.3.3 레코드 수준의 잠금 확인 및 해제

## 5.4 MySQL의 격리 수준

### 5.4.1 READ UNCOMMITTED

### 5.4.2 READ COMMITTED

### 5.4.3 REPEATABLE READ

### 5.4.4 SERIALIZABLE
