<div align="center">

# Deep Dive  
Deep Dive into Real My SQL 8.0(2)

--- 
</div>

# MySQL 서브쿼리(Subquery)

서브쿼리가 사용되는 위치별로 어떻게 최적화되는지, 그리고 어떻게 쿼리를 작성해야 성능에 도움이 될지를 살펴보자.


## SELECT 절에 사용된 서브쿼리

SELECT 절에 사용된 서브쿼리는 내부적으로 임시 테이블을 만들거나 쿼리를 비효율적으로 실행하게 만들지는 않기 때문에 서브쿼리가 적절히 인덱스를 사용할 수 있다면 크게 주의할 사항은 없다.

일반적으로 SELECT 절에 서브쿼리를 사용하면 그 서브쿼리는 항상 칼럼과 레코드가 하나인 결과를 반환해야 한다. 그 값이 NULL이든 아니든 관계없이 레코드가 1건이 존재해야 한다는 것인데, MySQL에서는 이 체크 조건이 조금은 느슨하다.

```sql
# 결과가 0건인 경우 - 에러 없이 NULL로 채워져서 반환
SELECT emp_no, (SELECT dept_name FROM departments WHERE dept_name='Sales1')
FROM dept_emp LIMIT 10;

# 결과가 2건 이상인 경우 - 에러 발생
# ERROR 1242 (21000): Subquery returns more than 1 row
SELECT emp_no, (SELECT dept_name FROM departments)
FROM dept_emp LIMIT 10;

# 2개 이상의 칼럼을 반환하는 경우 - 에러 발생
# ERROR 1241 (21000): Operand should contain 1 column(s)
SELECT emp_no, (SELECT dept_no, dept_name FROM departments WHERE dept_name='Sales1')
FROM dept_emp LIMIT 10;
```

### SELECT 절 서브쿼리 규칙

- 서브쿼리 결과가 0건이면 NULL로 채워져서 반환된다
- 서브쿼리가 2건 이상의 레코드를 반환하면 에러가 발생한다
- 서브쿼리가 2개 이상의 칼럼을 반환하면 에러가 발생한다

즉, SELECT 절의 서브쿼리에는 로우 서브쿼리를 사용할 수 없고, 오로지 스칼라 서브쿼리만 사용할 수 있다.

> 참고: 서브쿼리는 만들어 내는 결과에 따라 스칼라 서브쿼리(Scalar subquery)와 로우 서브쿼리(Row subquery)로 구분할 수 있다. 스칼라 서브쿼리는 레코드의 칼럼이 각각 하나인 결과를 만들어 내는 서브쿼리며, 스칼라 서브쿼리보다 레코드 건수가 많거나 칼럼 수가 많은 결과를 만들어 내는 서브쿼리를 로우 서브쿼리 또는 레코드 서브쿼리라고 한다.

### 조인으로 변환 권장

가끔 조인으로 처리해도 되는 쿼리를 SELECT 절의 서브쿼리를 사용해서 작성할 때도 있다. 하지만 서브쿼리로 실행될 때보다 조인으로 처리할 때가 조금 더 빠르기 때문에 가능하다면 조인으로 쿼리를 작성하는 것이 좋다.

```sql
# 서브쿼리 사용 (평균 0.78초)
SELECT
  COUNT(CONCAT(e1.first_name,
              (SELECT e2.first_name FROM employees e2 WHERE e2.emp_no=e1.emp_no))
  ) FROM employees e1;

# 조인으로 변환 (평균 0.65초) - 권장
SELECT COUNT(CONCAT(e1.first_name, e2.first_name))
FROM employees e1, employees e2
WHERE e1.emp_no=e2.emp_no;
```

처리해야 하는 레코드 건수가 많아지면 많아질수록 성능 차이가 커질 수도 있으므로 가능하면 조인으로 쿼리를 작성하는 방법을 권장한다.

### 래터럴 조인 활용 (MySQL 8.0+)

SELECT 절에 동일한 서브쿼리가 여러 번 사용되는 경우가 있다.

```sql
# 동일한 서브쿼리가 3번 반복 - 비효율적
SELECT e.emp_no, e.first_name,
  (SELECT s.salary FROM salaries s
   WHERE s.emp_no=e.emp_no
   ORDER BY s.from_date DESC LIMIT 1) AS salary,
  (SELECT s.from_date FROM salaries s
   WHERE s.emp_no=e.emp_no
   ORDER BY s.from_date DESC LIMIT 1) AS salary_from_date,
  (SELECT s.to_date FROM salaries s
   WHERE s.emp_no=e.emp_no
   ORDER BY s.from_date DESC LIMIT 1) AS salary_to_date
FROM employees e
WHERE e.emp_no=499999;
```

이 쿼리의 경우, 서브쿼리에 "LIMIT 1" 조건 때문에 salaries 테이블을 조인으로 사용할 수가 없었다. 하지만 MySQL 8.0 버전부터 도입된 래터럴 조인을 이용하면 이렇게 동일한 레코드의 각 칼럼을 가져오기 위해서 서브쿼리를 3번씩이나 남용하지 않아도 된다.

```sql
# 래터럴 조인으로 변환 - salaries 테이블을 한 번만 읽음
SELECT e.emp_no, e.first_name,
       s2.salary, s2.from_date, s2.to_date
FROM employees e
INNER JOIN LATERAL (
  SELECT * FROM salaries s
  WHERE s.emp_no=e.emp_no
  ORDER BY s.from_date DESC
  LIMIT 1) s2 ON s2.emp_no=e.emp_no
WHERE e.emp_no=499999;
```

3번의 서브쿼리를 하나의 래터럴 조인으로 변경했기 때문에 이제 salaries 테이블을 한 번만 읽어서 쿼리를 처리할 수 있다.


## FROM 절에 사용된 서브쿼리

이전 버전의 MySQL 서버에서는 FROM 절에 서브쿼리가 사용되면 항상 서브쿼리의 결과를 임시 테이블로 저장하고 필요할 때 다시 임시 테이블을 읽는 방식으로 처리했다. 그래서 가능하면 FROM 절의 서브쿼리를 외부 쿼리로 병합하는 형태로 쿼리 튜닝을 했다. 하지만 MySQL 5.7 버전부터는 옵티마이저가 FROM 절의 서브쿼리를 외부 쿼리로 병합하는 최적화를 수행하도록 개선됐다.

```sql
# 이 쿼리는 MySQL 5.7+에서 서브쿼리가 외부 쿼리와 병합됨
# 실행 계획의 select_type이 DERIVED가 아닌 SIMPLE로 표시됨
SELECT * FROM (SELECT * FROM employees) y;
```

EXPLAIN 명령을 실행한 후 SHOW WARNINGS 명령을 실행하면 MySQL 서버가 서브쿼리를 병합해서 재작성한 쿼리의 내용을 확인할 수 있다.

서브쿼리의 외부 쿼리 병합은 꼭 FROM 절의 서브쿼리에 대해서만 적용되는 최적화는 아니다. FROM 절에 사용된 뷰(View)의 경우에도 MySQL 옵티마이저는 뷰 쿼리와 외부 쿼리를 병합해서 최적화된 실행 계획을 사용한다.

### 병합이 불가능한 경우

FROM 절의 모든 서브쿼리를 외부 쿼리로 병합할 수 있는 것은 아니다. 대표적으로 다음과 같은 기능이 서브쿼리에 사용되면 FROM 절의 서브쿼리는 외부 쿼리로 병합되지 못한다.

- 집합 함수 사용(SUM(), MIN(), MAX(), COUNT() 등)
- DISTINCT
- GROUP BY 또는 HAVING
- LIMIT
- UNION(UNION DISTINCT) 또는 UNION ALL
- SELECT 절에 서브쿼리가 사용된 경우
- 사용자 변수 사용(사용자 변수에 값이 할당되는 경우)

### ORDER BY 절 처리

외부 쿼리와 병합되는 FROM 절의 서브쿼리가 ORDER BY 절을 가진 경우에는 외부 쿼리가 GROUP BY나 DISTINCT 같은 기능을 사용하지 않는다면 서브쿼리의 정렬 조건을 외부 쿼리로 같이 병합한다. 외부 쿼리에서 GROUP BY나 DISTINCT와 같은 기능이 사용되고 있다면, 서브쿼리의 정렬 작업은 무의미하기 때문에 서브쿼리의 ORDER BY 절은 무시된다.

MySQL 서버에서 FROM 절의 서브쿼리를 외부 쿼리로 병합하는 최적화는 optimizer_switch 시스템 변수로 제어할 수 있는데, 서브쿼리 병합에 대한 더 자세한 내용과 제어 방법은 9.3.1.16절 '파생 테이블 머지(derived_merge)'를 참조하자.


## WHERE 절에 사용된 서브쿼리

WHERE 절의 서브쿼리는 SELECT 절이나 FROM 절보다는 다양한 형태(연산자)로 사용될 수 있는데, 크게 다음 3가지로 구분해서 살펴보겠다. 이는 MySQL 옵티마이저가 최적화하는 형태를 기준으로 구분해본 것이다.

- 동등 또는 크다 작다 비교(= (subquery))
- IN 비교(IN (subquery))
- NOT IN 비교(NOT IN (subquery))

### 동등 또는 크다 작다 비교

MySQL 5.5 이전 버전까지는 서브쿼리 외부의 조건으로 쿼리를 실행하고, 최종적으로 서브쿼리를 체크 조건으로 사용했다. 하지만 이러한 처리 방식의 경우 풀 테이블 스캔이 필요한 경우가 많아서 성능 저하가 심각했다.

```sql
SELECT * FROM dept_emp de
WHERE de.emp_no=(SELECT e.emp_no
                 FROM employees e
                 WHERE e.first_name='Georgi' AND e.last_name='Facello' LIMIT 1);
```

MySQL 5.5 이전 버전까지는 위 쿼리의 경우 dept_emp 테이블을 풀 스캔하면서 서브쿼리의 조건에 일치하는지 여부를 체크했다. 하지만 이는 많은 사용자가 기대하는 실행 순서는 아니었을 것이다.

MySQL 5.5 버전부터는 이 쿼리의 실행이 정반대로 개선됐다.

1. 서브쿼리를 먼저 실행
2. 서브쿼리 결과를 상수로 변환
3. 상숫값으로 서브쿼리를 대체해서 나머지 쿼리 부분을 처리

이렇게 되면 dept_emp 테이블을 풀 스캔하지 않고 (emp_no, from_date) 조합의 인덱스를 사용할 수 있다. 여기서는 동등 비교만 예시로 살펴봤지만 동등 비교 대신 크다 또는 작다 비교가 사용돼도 동일한 실행 계획을 사용한다.

### 튜플 비교 방식 주의사항

다음과 같이 단일 값 비교가 아닌 튜플 비교 방식이 사용되면 서브쿼리가 먼저 처리되어 상수화되긴 하지만 외부 쿼리는 인덱스를 사용하지 못하고 풀 테이블 스캔을 실행한다.

```sql
# 튜플 비교 - 인덱스 사용 불가, 풀 테이블 스캔 발생
SELECT *
FROM dept_emp de WHERE (emp_no, from_date) = (
  SELECT emp_no, from_date
  FROM salaries
  WHERE emp_no=100001 limit 1);
```

MySQL 8.0 버전이라고 하더라도 아직 튜플 형태의 비교는 주의해서 사용해야 한다.

### IN 비교

실제 조인은 아니지만 다음 예제와 같이 테이블의 레코드가 다른 테이블의 레코드를 이용한 표현식(또는 칼럼 그 자체)과 일치하는지를 체크하는 형태를 세미 조인(Semi-Join)이라고 한다. 즉 WHERE 절에 사용된 IN (subquery) 형태의 조건을 조인의 한 방식인 세미 조인이라고 보는 것이다.

```sql
SELECT *
FROM employees e
WHERE e.emp_no IN
      (SELECT de.emp_no FROM dept_emp de WHERE de.from_date='1995-01-01');
```

MySQL 5.5 버전까지는 세미 조인의 최적화가 매우 부족해서 대부분 풀 테이블 스캔을 했다. 그래서 이런 세미 조인 형태는 MySQL 서버에서 사용하면 안 되는 패턴으로 기억하는 사용자가 많을 것이다. 하지만 MySQL 5.6 버전부터 8.0 버전까지 세미 조인의 최적화가 많이 개선되면서 이제 더 이상은 IN (subquery) 형태를 2개의 쿼리로 쪼개어 실행하거나 다른 우회 방법을 찾을 필요가 없어졌다.