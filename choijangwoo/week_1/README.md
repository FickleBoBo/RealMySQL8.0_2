<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(2)

--- 
</div>

## Contents
- [MySQL 연산자와 내장 함수](#113-mysql-연산자와-내장-함수)
---

## 11.3 MySQL 연산자와 내장 함수

### 11.3.1 리터럴 표기법

#### 문자열
- SQL 표준에서 문자열은 홑따옴표(')를 사용해 표기
- MySQL에서는 쌍따옴표(")도 문자열 표기에 사용 가능
- 역따옴표(`)로 감싸서 예약어와의 충돌을 피할 수 있음
```sql
SELECT * FROM departments WHERE dept_name = 'Sales';
SELECT * FROM departments WHERE dept_name = "Sales";
```

#### 숫자
문자 타입을 숫자로 조회하거나 숫자를 문자 조건으로 사용할 수 있다. 단, 형변환에 따른 성능 저하나 잘못된 결과를 반환할 수 있으므로 조회 시 타입을 맞추어 사용해야 한다.
```sql
SELECT * FROM tab_test WHERE number_column = '10001';
SELECT * FROM tab_test WHERE string_column = 10001;
```

위 두 쿼리 모두 실행 가능하지만 타입이 일치하지 않으면 인덱스를 사용하지 못하거나 풀 테이블 스캔이 발생할 수 있다.

#### 날짜
MySQL의 정해진 날짜 포맷으로 표기하면 자동으로 DATE/DATETIME 값으로 변환된다. Oracle처럼 명시적으로 문자열을 DATE 타입으로 변환할 필요가 없다.
```sql
SELECT * FROM orders WHERE order_date = '2024-01-15';
SELECT * FROM orders WHERE order_date = '20240115';
```

#### 불리언
- 컬럼 타입을 BOOLEAN으로 생성 시 내부적으로 TINYINT 타입으로 생성됨
- TRUE, FALSE 문자로 삽입 가능하며 각각 1, 0으로 저장됨
- 0과 1 외의 숫자 값도 저장 가능하므로 주의가 필요함
```sql
CREATE TABLE tb_boolean (bool_value BOOLEAN);
INSERT INTO tb_boolean VALUES (TRUE), (FALSE), (2);
SELECT * FROM tb_boolean WHERE bool_value = TRUE;
```

위 쿼리 결과에서 2는 조회되지 않는다. TRUE는 1과 동등하기 때문이다.

---

### 11.3.2 MySQL 연산자

#### 동등 비교 (=, <=>)
- =: 일반적인 동등 비교 (NULL 비교 시 NULL 반환)
- <=>: NULL-safe 동등 비교 연산자 (NULL끼리 비교 시 TRUE 반환)
```sql
SELECT 1 = 1, NULL = NULL, 1 = NULL;
SELECT 1 <=> 1, NULL <=> NULL, 1 <=> NULL;
```

#### AND와 OR의 우선순위
AND가 OR보다 우선순위가 높다. 혼합 사용 시 괄호로 명확히 구분하는 것이 좋다.
```sql
SELECT * FROM employees WHERE dept_id = 1 OR dept_id = 2 AND salary > 5000;
SELECT * FROM employees WHERE (dept_id = 1 OR dept_id = 2) AND salary > 5000;
```

두 쿼리의 결과는 다르다. 첫 번째는 dept_id=1 이거나 (dept_id=2이면서 salary>5000)인 결과를 반환한다.

#### REGEXP 연산자
- RLIKE와 REGEXP는 동일한 기능 수행 (정규표현식 매칭)
- POSIX 표준을 따름
- REGEXP 조건의 비교는 인덱스(Range scan)를 사용할 수 없음
```sql
SELECT * FROM employees WHERE name REGEXP '^Kim';
SELECT * FROM employees WHERE name RLIKE 'son$';
```

#### LIKE 연산자
- Range scan을 사용할 수 있음
- 와일드카드(%, _)를 문자 앞에 사용하면 인덱스를 활용하지 못함
```sql
SELECT * FROM users WHERE name LIKE 'Kim%';
SELECT * FROM users WHERE name LIKE '%Kim';
```

첫 번째 쿼리는 인덱스 사용 가능, 두 번째는 풀 테이블 스캔 발생

#### BETWEEN 연산자
- 크다/작다를 비교하는 연산자 (문자 타입에서도 사용 가능)
- Range scan 사용 가능
- 시작값과 끝값 모두 포함 (inclusive)
```sql
SELECT * FROM employees WHERE salary BETWEEN 3000 AND 5000;
SELECT * FROM employees WHERE dept_code BETWEEN 'A' AND 'D';
```

#### IN 연산자
- 여러 번의 동등 비교로 실행하여 성능상 이점이 있음
- NOT IN은 Full scan을 유발할 수 있음
- IN절 아이템 수는 max_allowed_packet 옵션에 영향을 받음
```sql
SELECT * FROM employees WHERE dept_id IN (1, 2, 3);
SELECT * FROM employees WHERE dept_id NOT IN (SELECT dept_id FROM closed_depts);
```

서브쿼리와 함께 사용 시 성능에 주의가 필요하다.

---

### 11.3.3 MySQL 내장 함수

#### 11.3.3.1 NULL 값 비교 및 대체 (IFNULL, ISNULL, NULLIF, COALESCE)
```sql
SELECT IFNULL(NULL, 'default');
SELECT ISNULL(NULL);
SELECT NULLIF(1, 1);
SELECT COALESCE(NULL, NULL, 'value');
```

- IFNULL(expr1, expr2): expr1이 NULL이면 expr2 반환
- ISNULL(expr): expr이 NULL이면 1, 아니면 0 반환
- NULLIF(expr1, expr2): 두 값이 같으면 NULL, 다르면 expr1 반환
- COALESCE(expr1, expr2, ...): NULL이 아닌 첫 번째 값 반환

#### 11.3.3.2 현재 시각 조회 (NOW, SYSDATE)

| 함수 | 특징 |
|------|------|
| NOW() | 하나의 SQL 문 내에서 항상 같은 값 반환 |
| SYSDATE() | 호출 시점에 따라 결과값이 다를 수 있음 |
```sql
SELECT NOW(), SLEEP(2), NOW();
SELECT SYSDATE(), SLEEP(2), SYSDATE();
```

NOW()는 두 번 호출해도 같은 값이지만 SYSDATE()는 SLEEP 이후 다른 값을 반환한다.

SYSDATE() 사용 시 주의사항
- 레플리카 서버에 영향을 줄 수 있음
- 인덱스를 효율적으로 사용하지 못함
- NOW() 사용을 권장
- sysdate-is-now 시스템 변수로 SYSDATE()가 NOW()처럼 동작하게 설정 가능

#### 11.3.3.3 날짜와 시간의 포맷 (DATE_FORMAT, STR_TO_DATE)
```sql
SELECT DATE_FORMAT(NOW(), '%Y-%m-%d %H:%i:%s');
SELECT STR_TO_DATE('2024-01-15', '%Y-%m-%d');
```

주요 포맷 지정자
- %Y: 4자리 연도
- %m: 2자리 월
- %d: 2자리 일
- %H: 24시간 형식 시
- %i: 분
- %s: 초

#### 11.3.3.4 날짜와 시간의 연산 (DATE_ADD, DATE_SUB)
```sql
SELECT DATE_ADD(NOW(), INTERVAL 1 DAY);
SELECT DATE_SUB(NOW(), INTERVAL 1 MONTH);
SELECT DATE_ADD(NOW(), INTERVAL -1 DAY);
```

주로 파티션 삭제와 같은 배치성 작업에 사용한다. INTERVAL 키워드 뒤에 DAY, MONTH, YEAR, HOUR, MINUTE, SECOND 등을 지정할 수 있다.

#### 11.3.3.5 타임스탬프 연산 (UNIX_TIMESTAMP, FROM_UNIXTIME)
```sql
SELECT UNIX_TIMESTAMP('2024-01-15 12:00:00');
SELECT FROM_UNIXTIME(1705312800);
```

- UNIX_TIMESTAMP: 날짜를 1970-01-01 00:00:00 UTC 기준 초 단위 정수로 변환
- FROM_UNIXTIME: 정수를 날짜로 변환

#### 11.3.3.6 문자열 처리 (RPAD, LPAD, TRIM)
```sql
SELECT RPAD('Hello', 10, '*');
SELECT LPAD('Hello', 10, '*');
SELECT TRIM('  Hello  ');
SELECT TRIM(BOTH 'x' FROM 'xxxHelloxxx');
```

- RPAD: 오른쪽에 문자 채움
- LPAD: 왼쪽에 문자 채움
- TRIM: 공백 또는 지정 문자 제거

#### 11.3.3.7 문자열 결합 (CONCAT, CONCAT_WS)
```sql
SELECT CONCAT('Hello', ' ', 'World');
SELECT CONCAT_WS('-', '2024', '01', '15');
```

- CONCAT: 문자열 단순 결합
- CONCAT_WS: 첫 번째 인자를 구분자로 사용하여 결합 (With Separator)
- CONCAT은 인자 중 NULL이 있으면 NULL 반환, CONCAT_WS는 NULL을 무시

#### 11.3.3.8 GROUP BY 문자열 결합 (GROUP_CONCAT)
```sql
SELECT dept_id, GROUP_CONCAT(emp_name ORDER BY emp_name SEPARATOR ', ')
FROM employees
GROUP BY dept_id;

SELECT dept_id, GROUP_CONCAT(DISTINCT emp_name ORDER BY emp_name)
FROM employees
GROUP BY dept_id;
```

- 정렬, 중복 제거, 구분자 변경 가능
- group_concat_max_len 시스템 변수로 메모리 버퍼 크기 조정 가능 (기본값 1024)
- MySQL에서는 초과 시 경고 처리하지만 JDBC에서는 에러 발생 가능

#### 11.3.3.9 값의 비교와 대체 (CASE WHEN...THEN...END)
```sql
SELECT name,
       CASE 
         WHEN salary >= 5000 THEN 'High'
         WHEN salary >= 3000 THEN 'Medium'
         ELSE 'Low'
       END AS salary_level
FROM employees;

SELECT name,
       CASE dept_id
         WHEN 1 THEN 'Sales'
         WHEN 2 THEN 'Dev'
         ELSE 'Other'
       END AS dept_name
FROM employees;
```

프로그래밍 언어의 SWITCH 구문과 유사하다. 두 가지 형식으로 사용 가능하다.

#### 11.3.3.10 타입 변환 (CAST, CONVERT)
```sql
SELECT CAST('123' AS SIGNED);
SELECT CONVERT('123', SIGNED);
SELECT CAST('2024-01-15' AS DATE);
SELECT CAST(1234567 AS CHAR);
```

- 문자열↔숫자, 문자열↔날짜 변환은 자동으로 처리되는 경우가 많음
- SIGNED, UNSIGNED의 경우 명시적 변환이 필요
- 변환 가능 타입: BINARY, CHAR, DATE, DATETIME, DECIMAL, SIGNED, UNSIGNED, TIME 등

#### 11.3.3.11 이진값과 16진수 문자열 변환 (HEX, UNHEX)
```sql
SELECT HEX('MySQL');
SELECT UNHEX('4D7953514C');
```

- HEX: 문자열을 16진수로 변환
- UNHEX: 16진수를 문자열로 변환
- 바이너리 데이터 저장이나 디버깅 시 유용

#### 11.3.3.12 암호화 및 해시 함수 (MD5, SHA, SHA2)
```sql
SELECT MD5('password');
SELECT SHA('password');
SELECT SHA2('password', 256);
```

- MD5: 128비트 해시값 생성 (32자리 16진수 문자열)
- SHA (SHA1): 160비트 해시값 생성 (40자리 16진수 문자열)
- SHA2: 두 번째 인자로 비트 수 지정 (224, 256, 384, 512)

비밀번호 저장보다는 체크섬 비교나 데이터 무결성 검증에 주로 사용한다. 비밀번호 저장에는 bcrypt 등 별도의 라이브러리 사용을 권장한다.