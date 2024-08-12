# 데이터베이스 쿼리 최적화 및 인덱스 설계

## 문제 정의: 최적화가 필요한 데이터베이스 쿼리와 인덱스 문제 정의

데이터를 다룰 때 쿼리가 비효율적이라면 프로그램이 느리게 실행되어 전체 시스템의 성능을 저하시킬 수 있기 떄문에 데이터를 효율적으로 관리하고 빠르게 조회하는 것은 사용자 경험에 큰 영향을 미칩니다. 이에 대한 원인으로는 잘못된 쿼리 작성, 비효율적인 서브쿼리 사용, 적절한 인덱스 부재 등이 있습니다. 이러한 문제를 해결하기 위해 데이터베이스 쿼리 최적화와 인덱스 설계가 필요합니다.

## 솔루션 도출: 최적화 방안 및 인덱스 설계 솔루션 도출

1. 서브쿼리를 조인으로 대체하여 복잡성을 줄입니다.
2. 인덱스를 추가하여 검색 성능을 향상시킵니다.
3. 복합 인덱스를 설계하여 다중 열 검색을 최적화합니다.
4. 쿼리 캐시를 활용하여 반복적인 쿼리에 대한 성능을 개선합니다.

## 인덱스 사용 이유
### 조회 성능 개선
데이터베이스에서 성능 최적화는 디스크 I/O와 관련이 많습니다.
조회 성능 개선의 핵심은 디스크 I/O를 줄이는 것입니다.
인덱스를 사용하면 조회에서는 이득을 얻지만 수정/삭제에서는 손해를 보게 됩니다. 하지만 웹 애플리케이션에서 조회의 빈도가 8~9할이기 때문에 조회에서 성능 최적화를 하는 것이 중요합니다.
### 정렬
인덱스를 사용하면 정렬이 이미 처리되어있기 떄문에 ORDER BY를 통해 따로 정렬할 필요없이 인덱스 순서대로 파일을 읽기만 하면 되기 때문에 정렬에도 이점이 있다.
## 설계: 최적화된 쿼리와 인덱스 설계
- 컬럼에 인덱스를 걸어야한다면 서비스의 특성을 고려하여 조회가 많이 일어나는 컬럼을 파악하여 인덱스를 적용하거나 카디널리티가 높은 컬럼에 인덱스를 적용해야합니다.
- 우선 예제를 통해 실행 계획을 알아보고 인덱스를 설계해보겠습니다.
- 저는 1995년 1월 1일 이전에 고용된 직원 중에서 2000년 1월 1일 이후의 기간에서 가장 높은 급여를 받은 직원들의 정보를 입사일을 기준으로 내림차순으로 정렬하여 가져오는 쿼리를 사용했습니다.
### 기존 쿼리
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
ORDER BY e.hire_date DESC;
``` 
- 우선 EXPLAIN으로 이 쿼리의 실행 계획을 한번 보겠습니다.
```sql
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

- 우선 employees와 salaries의 type이 ALL으로 되어있는데 이는 테이블을 풀 스캔(데이터를 하나하나 읽음)한다는 의미로 효율성이 매우 낮습니다.
- 그리고 Extra에서의 Using where은 WHERE 조건이 사용되었다는 의미이고 Using filesort는 파일 정렬 사용했다는 것으로 효율성 낮다는 것을 의미합니다.
- s 테이블의 type인 ref는 인덱스를 통한 단일 레코드 접근을 의미하며 이는 효율성 높음을 의미합니다.
- 저는 이 쿼리의 문제점을 WHERE 절에서 서브쿼리가 반복하여 비교되어 성능에 문제가 생긴다고 생각하고 있습니다. mysql에서도 적절한 캐싱 전략을 사용하겠지만 캐시 데이터가 사용된다고 해도 한 번의 서브쿼리 실행으로 이 쿼리가 구성된다고 생각하지는 않기 떄문에 서브쿼리의 사용이 비효율적이라 생각합니다.
- 그래서 저는 서브쿼리를 CTE로 만들어 한번의 불러오기로 최적화를 진행해보았습니다.
### 최적화된 쿼리
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
ORDER BY e.hire_date DESC;
```
  
- EXPLAIN으로 이 쿼리의 실행 계획을 한번 보겠습니다.
```sql
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
- 위에서 인덱스를 적용할 때 서비스의 특성을 파악하여 자주 사용되는 컬럼에 사용해야 한다고 얘기했습니다.
- 저는 이 데이터베이스에서 사용할 쿼리를 정했기 때문에 쿼리에서 사용하는 컬럼을 인덱스로 적용하여 테스트해보겠습니다.
- 인덱스는 where 절에서 사용되는 컬럼들을 모두 만들 것이며 총 3개를 만들 것입니다.
- salaries 테이블에서 salary와 from_date를, employees 테이블에서 hire_date를 이용하여 인덱스를 만들 것입니다!
```
CREATE INDEX idx_salaries_salary ON salaries(salary);
CREATE INDEX idx_salaries_from_date ON salaries(from_date);
CREATE INDEX idx_employees_hire_date ON employees(hire_date);
```

## 데이터베이스 스키마 분석 및 이해
- 우선 현재 테이블을 살펴보겠습니다.

<img width="560" alt="image" src="https://github.com/user-attachments/assets/1c72f336-fdb0-46eb-9194-82b61bc8f1db">

- 총 8개의 테이블이 있습니다.
- 테이블을 조사하다가 이 테이블 중 뷰가 있는 것을 확인하였기 때문에 뷰를 먼저 확인해보겠습니다.
### VIEW
<img width="1148" alt="image" src="https://github.com/user-attachments/assets/edf2dfd4-a129-4a18-bcb3-319cbacb3eb5">

- current_dept_emp뷰에 대한 정보부터 먼저 보겠습니다.
<img width="1918" alt="image" src="https://github.com/user-attachments/assets/69d7aafa-742b-440a-b825-61d4b5216cb7">

- current_dept_emp 뷰는 dept_emp 테이블과 dept_emp_latest_date 테이블을 조인하여 생성된 뷰인 것을 확인할 수 있습니다.
- 다음으로 dept_emp_latest_date뷰에 대한 정보를 보겠습니다.
<img width="1918" alt="image" src="https://github.com/user-attachments/assets/5f89adf6-d667-4547-a9ad-0e80e78306ff">

- dept_emp_latest_date 뷰는 dept_emp 테이블에서 emp_no로 그룹화하여 가장 최신의 from_date와 to_date를 선택하는 뷰인 것을 확인할 수 있습니다.
- 이제 테이블을 살펴보겠습니다.
### TABLE

- 우선 dept_emp부터 보겠습니다.

<img width="560" alt="image" src="https://github.com/user-attachments/assets/1bdb2f94-c0df-49a2-8236-66787e8d98c8">
<img width="560" alt="image" src="https://github.com/user-attachments/assets/5faa7d50-f2d2-46f8-af51-26c8d49dae7f">

- 331604개의 데이터가 있고 컬럼은 직원 번호, 부서 번호, 입사 날짜, 퇴사 날짜가 있습니다.
- 직원의 부서를 표시해주는 테이블인 것을 확인할 수 있습니다.
- 다음으로 current_dept_emp 뷰를 보겠습니다.

<img width="560" alt="image" src="https://github.com/user-attachments/assets/e54571f9-d49d-4b6e-bf35-4331ea7e759f">

<img width="560" alt="image" src="https://github.com/user-attachments/assets/44e76125-2595-42ad-9e3f-85ef705853b7">

- dept_emp 테이블과 dept_emp_latest_date 테이블을 조인하여 생성된 뷰이기 떄문에 current_dept_emp와 거의 유사한 것을 확인할 수 있습니다.
- 300024개의 직원 데이터가 있는 것으로 확인됩니다.
- 다음으로는 dept_emp_latest_date 뷰를 보겠습니다. 

<img width="1918" alt="image" src="https://github.com/user-attachments/assets/7677036e-cbd2-4511-aec3-d217e353cb54">
<img width="1918" alt="image" src="https://github.com/user-attachments/assets/db5c611a-dcb5-4865-9335-b8b0896f9166">

- emp_no, to_date, from_date가 있고 총 300024개의 데이터가 있는 것으로 확인됩니다.
- 각 직원들의 입사, 퇴사 날짜를 나타내는 뷰인 것을 확인할 수 있습니다.
- 다음으로는 departments 테이블을 확인해보겠습니다.

<img width="560" alt="image" src="https://github.com/user-attachments/assets/d98c8ef5-b05b-4270-ab2e-bf366ad0a517">

- 부서 번호와 부서 이름이 컬럼으로 있고 부서는 총 9개가 있습니다.
- 부서의 정보를 나타내는 테이블임을 알 수 있습니다.
- 다음으로는 dept_manager 테이블을 확인해보겠습니다.
<img width="931" alt="image" src="https://github.com/user-attachments/assets/bcc2ad78-e077-41a1-a812-737f57f9f704">

- 직원 번호, 부서 번호, 입사 날짜, 퇴사 날짜가 있고 퇴사 날짜에 9999년 1월 1일로 되어있는 사람이 각 부서에 한 명 있는 것으로 보아 각 부서의 전임 관리자와 현임 관리자 정보가 있다는 것을 유추할 수 있습니다.
- 다음으로 employees 테이블을 보겠습니다.
<img width="931" alt="image" src="https://github.com/user-attachments/assets/eb577b6c-0234-4226-93f2-3b3cbb5cfcba">
<img width="931" alt="image" src="https://github.com/user-attachments/assets/b5b1d741-e393-47ce-8538-db98f2300402">

- 직원 번호, 생년월일, 성, 이름, 성별, 고용 날짜가 있고 직원들의 정보를 담고있는 테이블인 것을 확인할 수 있습니다.
- 총 300024개의 데이터가 있습니다.
- 다음으로 salaries 테이블을 확인해보겠습니다.
<img width="931" alt="image" src="https://github.com/user-attachments/assets/ce9c7d2f-bd04-4600-8177-10edad111b53">
<img width="931" alt="image" src="https://github.com/user-attachments/assets/2feb542a-717a-49b1-93d7-04a51b410e74">

- 직원 번호, 급여, 입사 날짜, 퇴사 날짜가 나와있고 총 데이터는 2844047개로 엄청 많은 데이터를 가지고 있는 것을 확인할 수 있습니다.
- 각 직원의 급여 정보를 나타내는 테이블입니다.
- 마지막으로 titles 테이블을 확인해보겠습니다.
<img width="931" alt="image" src="https://github.com/user-attachments/assets/db2d133c-c592-4233-ab93-4531a9c33ba5">
<img width="931" alt="image" src="https://github.com/user-attachments/assets/eddc41f7-abbe-49c9-bf14-4437654b292c">

- 직원 정보, 역할, 입사 날짜, 퇴사 날짜가 있고 총 443308개의 데이터가 있습니다.
- 직원들의 역할을 담고있는 테이블입니다.
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

## 성능 재측정 (인덱스 추가 후 원래 쿼리와 최적화된 쿼리 비교)
- 인덱스 추가 후 최적화된 쿼리를 다시 실행해보겠습니다.
<img width="1304" alt="image" src="https://github.com/user-attachments/assets/22219826-fb10-40d0-b069-8d448ea3cd10">
- 0.58초로 시간이 크게 단축된 것을 확인할 수 있었습니다.

- 그렇다면 원래의 쿼리도 실행해보겠습니다.
- <img width="1299" alt="image" src="https://github.com/user-attachments/assets/e60bf29e-bd88-4bc0-85e9-ccdb81f64536">
- 0.59초로 최적화된 쿼리와 크게 차이가 나지 않는 것을 확인할 수 있습니다.

### 실행 계획 확인
- 그렇다면 서브 쿼리를 사용한 쿼리문과 최적화된 쿼리문이 왜 속도 차이가 나지 않는지 실행 계획을 한번 살펴보겠습니다.
#### 서브 쿼리 사용 원래 쿼리문
```sql
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
- 인덱스 사용 전 실행 계획
<img width="955" alt="image" src="https://github.com/user-attachments/assets/8c1d329c-6b69-40b6-a74c-49f198ec8643">

- 인덱스 사용 후 실행 계획
<img width="1315" alt="image" src="https://github.com/user-attachments/assets/eaf91641-7590-49fe-a326-0b83bb2e026d">

#### 최적화된 쿼리
```sql
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

- 인덱스 사용 전 실행 계획
<img width="893" alt="image" src="https://github.com/user-attachments/assets/5d989423-2dc4-4fd9-918b-0439c60fa9cb">

- 인덱스 사용 후 실행 계획
<img width="1250" alt="image" src="https://github.com/user-attachments/assets/72874bb6-1556-4150-91dd-c139ddc1ca38">

### 원인 분석
- 이전 서브쿼리를 사용한 쿼리문에는 rows가 299733개인 테이블에서 filesort가 사용되었고 최적화된 쿼리문에서는 rows가 1개인 테이블에서 filesort가 사용되기 때문에 이에 대한 영향으로 시간 차이가 났다고 가정한다면 인덱스를 사용한 후에는 각각의 s, derived2 테이블이 rows가 1이고 이 테이블에서 인덱스 사용, 임시 테이블 사용, 파일 정렬 사용을 하기 때문에 둘의 속도 차이가 비슷한 것으로 추측해볼 수 있습니다.

## 복합 인덱스 사용
- 복합 인덱스는 두 개 이상의 컬럼을 합쳐 인덱스를 만드는 것입니다.
- 하나의 컬럼으로 인덱스를 만들었을 때보다 더 적은 데이터 분포를 보여 탐색할 데이터 수가 줄어든다는 장점이 있습니다.
- 결합 인덱스, 다중 컬럼 인덱스, Composite Index라고도 불립니다.
- 인덱스 효율성을 높이기 위해 복합 인덱스를 사용하겠습니다.
- salaries에서 사용하는 from_date와 salary를 사용하여 복합 인덱스를 만들어보겠습니다.
```sql
CREATE INDEX idx_salaries_emp_date_salary ON salaries(from_date, salary);
```

### 실행 계획 확인
- 우선 실행 계획에서 달라진 것이 있는지 확인해보겠습니다.
#### 서브 쿼리 사용 원래 쿼리문
```sql
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
- 단일 인덱스 사용 실행 계획
<img width="1315" alt="image" src="https://github.com/user-attachments/assets/eaf91641-7590-49fe-a326-0b83bb2e026d">

- 복합 인덱스 추가 후 실행 계획
<img width="1544" alt="image" src="https://github.com/user-attachments/assets/b7584863-b2af-4f79-966d-d3be3eb7903d">

#### 최적화된 쿼리
```sql
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

- 단일 인덱스 사용 실행 계획
<img width="1250" alt="image" src="https://github.com/user-attachments/assets/72874bb6-1556-4150-91dd-c139ddc1ca38">

- 복합 인덱스 추가 후 실행 계획
<img width="1476" alt="image" src="https://github.com/user-attachments/assets/e2fd41d4-0520-4bcb-9e56-9a5263e30c3c">

#### 차이점
- 복합 인덱스를 추가한 후 salaries 테이블에서 Key_len, Rows, Filtered가 달라진 것을 확인할 수 있습니다.
- Key_len: Key_len은 인덱스 키의 길이를 나타내는데 복헙 인덱스를 사용하여 두 개의 컬럼(from_date, salary)이 결합되어 줄어든 것 같습니다.
- Rows: Rows는 검색해야하는 행의 수를 추정한 값인데 복합 인덱스를 사용하면 인덱스가 두 개의 조건(from_date, salary)을 동시에 커버하기 때문에 검색하는 행의 수가 줄어든 것 같습니다.
- Filtered: Filtered는 조건을 충족하는 행의 비율을 백분율로 나타낸 것인데 복합 인덱스를 사용하여 쿼리가 더 많은 행을 필터링하지 않아도 되기 때문에 조건을 충족하는 행의 비율이 증가한 것 같습니다.

## 성능 재측정 (복합 인덱스 추가 전과 복합 인덱스 추가 후 성능 비교)
### 복합 인덱스 추가 전 성능 
#### 서브쿼리를 사용한 원래 쿼리
<img width="1299" alt="image" src="https://github.com/user-attachments/assets/e60bf29e-bd88-4bc0-85e9-ccdb81f64536">
- 0.59초

#### 조인을 사용한 쿼리
<img width="1304" alt="image" src="https://github.com/user-attachments/assets/22219826-fb10-40d0-b069-8d448ea3cd10">
- 0.58초

### 복합 인덱스 추가 후 성능
#### 서브쿼리를 사용한 원래 쿼리
<img width="1462" alt="image" src="https://github.com/user-attachments/assets/2151885c-9a1e-4a52-95ff-8a6ade098a80">
- 0.16초

#### 조인을 사용한 쿼리
<img width="1462" alt="image" src="https://github.com/user-attachments/assets/b8faeaad-84ba-4bc9-9ff6-f8373984e7ad">
- 0.17초

### 정리
- 성능이 약 3배가량 빨라진 것을 확인할 수 있습니다.

## 인덱스 통계 수집 및 분석
### 인덱스 통계 수집
- 우선 information_schema.STATISTICS테이블을 이용하여 데이터베이스의 인덱스와 관련된 기본적인 메타데이터를 확인해보겠습니다.

``` mysql
SELECT TABLE_NAME, INDEX_NAME, SEQ_IN_INDEX, COLUMN_NAME, CARDINALITY
FROM information_schema.STATISTICS
WHERE TABLE_SCHEMA = 'employees';
```

<img width="997" alt="image" src="https://github.com/user-attachments/assets/e67384cd-4cd1-48cf-977e-3cdee5150dac">

- TABLE_NAME: 인덱스가 적용된 테이블의 이름.
- INDEX_NAME: 인덱스의 이름.
- SEQ_IN_INDEX: 인덱스에서 열(column)이 나오는 순서.
- COLUMN_NAME: 인덱스가 적용된 열(column)의 이름.
- CARDINALITY: 인덱스가 커버하는 고유 값의 수, 값이 높을수록 인덱스의 효율성이 높다고 판단할 수 있습니다.
- 테이블을 분석하기 전 카디널리티에 대해 더 알아보자면 카디널리티는 해당 컬럼의 중복된 수치인데, 중복도가 낮으면 카디널리티가 높다고 표현하고, 중복도가 높으면 카디널리티가 낮다고 표현합니다.
- 테이블 조회시 카디널리티가 높은 순으로 필터링하여 조회하기 때문에 여러 컬럼을 인덱스로 생성할 경우, 카디널리티가 높은 순에서 낮은 순으로 구성하는 것이 좋습니다.
​
- 그러면 전체 행 개수와 카디널리티를 비교하며 테이블 별 인덱스를 분석해보겠습니다.

#### 테이블별 인덱스 분석

<img width="997" alt="image" src="https://github.com/user-attachments/assets/b63cf585-ec5d-45e8-894e-fb32da80d6ca">

### 1. **departments 테이블**
- **행 개수 (`TABLE_ROWS`)**: 9
- **카디널리티 (`CARDINALITY`)**: 9 (`dept_name`, `dept_no`)
- `departments` 테이블의 행 개수와 `dept_name`, `dept_no` 인덱스의 카디널리티가 모두 9이며 이는 각 행이 고유한 부서 번호와 이름을 가지고 있음을 나타냅니다.
- 인덱스의 카디널리티가 테이블의 전체 행 개수와 같으므로, 이 인덱스는 검색에 있어 매우 효율적일 것입니다.

### 2. **dept_emp 테이블**
- **행 개수 (`TABLE_ROWS`)**: 331,143
- **카디널리티 (`CARDINALITY`)**:
    - `emp_no` (PRIMARY): 299,600
    - `dept_no`: 8
- 이 테이블의 대부분의 행이 고유한 `emp_no`를 가지고 있어, 이 열을 기준으로 하는 검색은 효율적일 것으로 예상됩니다.
- `dept_no`는 8개의 고유 값을 가지므로, 인덱스의 카디널리티가 낮아 `dept_no` 인덱스가 특정 부서를 검색할 때는 적합할 수 있지만, 인덱스 자체가 고유 값을 가진 데이터와 비교하면 효율성은 매우 떨어질 것으로 예상됩니다.

### 3. **dept_manager 테이블**
- **행 개수 (`TABLE_ROWS`)**: 24
- **카디널리티 (`CARDINALITY`)**:
    - `emp_no` (PRIMARY): 24
    - `dept_no`: 9
- `emp_no`의 카디널리티는 행 개수와 동일하므로, 이 인덱스는 각 행이 고유한 직원 번호를 가지고 있음을 나타냅니다. 이는 인덱스가 매우 효율적으로 작동할 수 있음을 의미합니다.
- `dept_no`의 카디널리티는 9로, 직원 별로 동일한 부서 번호를 가지고 있다는 것을 의미합니다. 이로 인해 `dept_no`를 기준으로 하는 검색에서는 인덱스의 효율성이 낮을 것으로 예상됩니다.

### 4. **employees 테이블**
- **행 개수 (`TABLE_ROWS`)**: 299,556
- **카디널리티 (`CARDINALITY`)**:
    - `emp_no` (PRIMARY): 299,556
    - `hire_date`: 4,747
- `emp_no`의 카디널리티가 전체 행 수와 일치하므로 모든 행이 고유한 `emp_no`를 가지므로, `emp_no`를 기준으로 검색할 때 인덱스의 성능이 효율적일 것입니다.
- `hire_date`의 카디널리티는 4,747로, 전체 행 수에 비해 많이 낮은 편입니다. 이 열의 인덱스가 특정한 날짜 범위를 검색할 때 효율적일 수 있지만, 전체적으로 인덱스의 고유성이 떨어질 것입니다.

### 5. **salaries 테이블**
- **행 개수 (`TABLE_ROWS`)**: 2,838,426
- **카디널리티 (`CARDINALITY`)**:
    - `emp_no` (PRIMARY): 303,596
    - `salary`: 71,272
    - `from_date`: 5,926
- 카디널리티가 가장 높은 `emp_no`의 카디널리티도 전체 행 수에 비해서는 낮지만, 다른 행의 비해서는 직원별 데이터를 검색할 때 효율적일 것입니다..
- `salary`와 `from_date`의 카디널리티는 상대적으로 낮아, 이 인덱스들이 특정 범위의 값을 검색할 때는 유용할 수 있으나, 전체적으로는 인덱스의 고유성이 떨어집니다.

### 6. **titles 테이블**
- **행 개수 (`TABLE_ROWS`)**: 442,129
- **카디널리티 (`CARDINALITY`)**:
    - `emp_no` (PRIMARY): 300,103
    - `title`: 442,129
    - `from_date`: 442,129
- `title`과 `from_date`의 카디널리티가 전체 행 수와 동일하므로 인덱스가 매우 효율적으로 작동할 수 있음을 나타냅니다.
- `emp_no`의 카디널리티가 전체 행 수보다는 낮지만, 그래도 높은 값을 가지고 있기 때문에 이를 이용한 검색도 아주 비효율적이지는 않을 것입니다.
