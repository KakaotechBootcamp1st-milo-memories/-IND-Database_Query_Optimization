# 데이터베이스 쿼리 최적화 및 인덱스 설계

## 문제 정의: 최적화가 필요한 데이터베이스 쿼리와 인덱스 문제 정의

데이터를 다룰 때 쿼리가 비효율적이라면 프로그램이 느리게 실행되어 전체 시스템의 성능을 저하시킬 수 있기 떄문에 데이터를 효율적으로 관리하고 빠르게 조회하는 것은 사용자 경험에 큰 영향을 미칩니다. 이에 대한 원인으로는 잘못된 쿼리 작성, 비효율적인 서브쿼리 사용, 적절한 인덱스 부재 등이 있습니다. 이러한 문제를 해결하기 위해 데이터베이스 쿼리 최적화와 인덱스 설계가 필요합니다.

## 솔루션 도출: 최적화 방안 및 인덱스 설계 솔루션 도출

1. 서브쿼리를 조인으로 대체하여 복잡성을 줄입니다.
2. 인덱스를 추가하여 검색 성능을 향상시킵니다.
3. 복합 인덱스를 설계하여 다중 열 검색을 최적화합니다.
4. 쿼리 캐시를 활용하여 반복적인 쿼리에 대한 성능을 개선합니다.

## 설계: 최적화된 쿼리와 인덱스 설계
- 저는 1995년 1월 1일 이전에 고용된 직원 중에서 2000년 1월 1일 이후의 기간에서 가장 높은 급여를 받은 직원들의 정보를 입사일을 기준으로 내림차순으로 정렬하여 가져오는 쿼리를 사용했습니다.
### 기존 쿼리
```
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
WHERE s.salary = (
    SELECT MAX(salary)
    FROM salaries
    WHERE from_date > '2000-01-01'
)
AND e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC;
```
- 우선 EXPLAIN으로 이 쿼리의 실행 계획을 한번 보겠습니다.
```
EXPLAIN SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
WHERE s.salary = (
    SELECT MAX(salary)
    FROM salaries
    WHERE from_date > '2000-01-01'
)
AND e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC, s.salary DESC;
```
<img width="955" alt="image" src="https://github.com/user-attachments/assets/8c1d329c-6b69-40b6-a74c-49f198ec8643">
- 우선 employees와 salaries의 type이 ALL으로 되어있는데 이는 테이블을 풀 스캔한다는 의미로 효율성이 매우 낮습니다.
- 그리고 Extra에서의 Using where은 WHERE 조건이 사용되었다는 의미이고 Using filesort는 파일 정렬 사용했다는 것으로 효율성 낮다는 것을 의미합니다.
- s 테이블의 type인 ref는 인덱스를 통한 단일 레코드 접근을 의미하며 이는 효율성 높음을 의미합니다.
- 저는 이 쿼리의 문제점을 WHERE 절에서 서브쿼리가 반복하여 비교되어 성능에 문제가 생긴다고 생각하고 있습니다. mysql에서도 적절한 캐싱 전략을 사용하겠지만 캐시 데이터가 사용된다고 해도 한 번의 서브쿼리 실행으로 이 쿼리가 구성된다고 생각하지는 않기 떄문에 서브쿼리의 사용이 비효율적이라 생각합니다.
- 그래서 저는 서브쿼리를 CTE로 만들어 한번의 불러오기로 최적화를 진행해보았습니다.
### 최적화된 쿼리

```
WITH MaxSalary AS (
    SELECT MAX(salary) AS max_salary
    FROM salaries
    WHERE from_date > '2000-01-01'
)
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
JOIN MaxSalary ms ON s.salary = ms.max_salary
WHERE e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC;
```
  
- 우선 EXPLAIN으로 이 쿼리의 실행 계획을 한번 보겠습니다.
```
EXPLAIN WITH MaxSalary AS (
    SELECT MAX(salary) AS max_salary
    FROM salaries
    WHERE from_date > '2000-01-01'
)
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
JOIN MaxSalary ms ON s.salary = ms.max_salary
WHERE e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC;
```

<img width="893" alt="image" src="https://github.com/user-attachments/assets/5d989423-2dc4-4fd9-918b-0439c60fa9cb">

- 우선 이 쿼리에서는 원래 쿼리의 서브 쿼리가 하는 역할의 테이블이 derived2로 생성되어 있는 것을 확인할 수 있습니다.
- 그 외에는 원래 쿼리와 동일하지만 e 테이블의 Using filesort가 derived2 테이블로 옮겨 갔고 e 테이블의 rows 갯수는 299733개이지만 derived2의 rows 갯수는 1개이기 때문에 이 부분에서 속도적인 차이가 난 것 같습니다.
- s 테이블의 type인 ref는 인덱스를 통한 단일 레코드 접근을 하여 효율성이 높지만 그 외 테이블은 적절한 인덱스가 없기 때문에 테이블을 풀스캔하여 효율이 떨어진다고 생각해볼 수 있습니다.
- 인덱스를 적절히 설계하여 사용한다면 s 테이블과 salaries 테이블에 대한 최적화를 할 수 있을 것이라 생각됩니다.

## 인덱스 설계
- 인덱스는 where 절에서 사용되는 컬럼들을 모두 만들 것이며 총 3개를 만들 것입니다.
- salaries 테이블에서 salary와 from_date를, employees 테이블에서 hire_date를 이용하여 인덱스를 만들 것입니다!
```
CREATE INDEX idx_salaries_salary ON salaries(salary);
CREATE INDEX idx_salaries_from_date ON salaries(from_date);
CREATE INDEX idx_employees_hire_date ON employees(hire_date);
```

## 데이터베이스 스키마 분석 및 이해

## 샘플 데이터 생성 및 데이터베이스에 삽입

- MySQL의 예제 데이터베이스인 'employees'를 사용하여 샘플 데이터를 생성하고 데이터베이스에 삽입합니다.

```
mysql -u root -p < employees.sql
```

## 기존 쿼리 성능 측정 (실행 계획 분석)
```sql
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
WHERE s.salary = (
    SELECT MAX(salary)
    FROM salaries
    WHERE from_date > '2000-01-01'
)
AND e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC, s.salary DESC;

```
<img width="905" alt="image" src="https://github.com/user-attachments/assets/977ffd49-b982-4530-b9ab-63a450a18d87">

- 이 쿼리는 2.66초가 걸렸습니다.

## 쿼리 리팩토링 및 최적화 (JOIN, 서브쿼리 등 최적화)
- 서브쿼리를 CTE(Common Table Expression)로 분리하고 조인문으로 바꿔 쿼리를 최적화합니다.

```sql
WITH MaxSalary AS (
    SELECT MAX(salary) AS max_salary
    FROM salaries
    WHERE from_date > '2000-01-01'
)
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
JOIN MaxSalary ms ON s.salary = ms.max_salary
WHERE e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC, s.salary DESC;

```
<img width="905" alt="image" src="https://github.com/user-attachments/assets/0c4d7efa-2b56-493d-a0e3-11d3830d3489">

- 최적화된 쿼리는 1.94초가 걸렸습니다.

## 인덱스 추가 및 기존 인덱스 재구성
- 인덱스 설계 때 설계했던 인덱스를 추가해줍니다.

```sql
CREATE INDEX idx_salaries_salary ON salaries(salary);
CREATE INDEX idx_salaries_from_date ON salaries(from_date);
CREATE INDEX idx_employees_hire_date ON employees(hire_date);
```

## 성능 재측정 (원래 쿼리와 최적화된 쿼리 비교)
- 인덱스 추가 후 최적화된 쿼리를 다시 실행해보겠습니다.
<img width="1304" alt="image" src="https://github.com/user-attachments/assets/22219826-fb10-40d0-b069-8d448ea3cd10">
- 0.58초로 시간이 크게 단축된 것을 확인할 수 있었습니다.

- 그렇다면 원래의 쿼리도 실행해보겠습니다.
- <img width="1299" alt="image" src="https://github.com/user-attachments/assets/e60bf29e-bd88-4bc0-85e9-ccdb81f64536">
- 0.59초로 최적화된 쿼리와 크게 차이가 나지 않는 것을 확인할 수 있습니다.

### 실행 계획 확인
- 그렇다면 서브 쿼리를 사용한 쿼리문과 최적화된 쿼리문이 왜 속도 차이가 나지 않는지 실행 계획을 한번 살펴보겠습니다.
- 서브 쿼리 사용 원래 쿼리문
```
EXPLAIN SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
WHERE s.salary = (
    SELECT MAX(salary)
    FROM salaries
    WHERE from_date > '2000-01-01'
)
AND e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC, s.salary DESC;
```
<img width="1315" alt="image" src="https://github.com/user-attachments/assets/eaf91641-7590-49fe-a326-0b83bb2e026d">

- 최적화된 쿼리
```
EXPLAIN WITH MaxSalary AS (
    SELECT MAX(salary) AS max_salary
    FROM salaries
    WHERE from_date > '2000-01-01'
)
SELECT e.emp_no, e.first_name, e.last_name, s.salary
FROM employees e
JOIN salaries s ON e.emp_no = s.emp_no
JOIN MaxSalary ms ON s.salary = ms.max_salary
WHERE e.hire_date < '1995-01-01'
ORDER BY e.hire_date DESC;
```
<img width="1250" alt="image" src="https://github.com/user-attachments/assets/72874bb6-1556-4150-91dd-c139ddc1ca38">

### 원인 분석
Type:
	•	ALL: 테이블 풀 스캔 (효율성 낮음)
	•	index: 인덱스 풀 스캔 (효율성 중간)
	•	range: 인덱스 범위 스캔 (효율성 높음)
	•	ref: 인덱스를 통한 단일 레코드 접근 (효율성 높음)
	•	eq_ref: 조인을 통한 단일 레코드 접근 (효율성 높음)
	•	const, system: 상수 값 접근 (효율성 가장 높음)
	2.	Extra:
	•	Using where: WHERE 조건이 사용됨
	•	Using temporary: 임시 테이블 사용 (효율성 낮음)
	•	Using filesort: 파일 정렬 사용 (효율성 낮음)
	•	Dependent subquery: 서브쿼리가 메인 쿼리의 각 행마다 실행됨 (효율성 낮음)

- 두 쿼리문 index(인덱스 풀 스캔: 효율성 중간), ref(인덱스를 통한 단일 레코드 접근:효율성 높음), eq_ref(조인을 통한 단일 레코드 접근:효율성 높음) 
