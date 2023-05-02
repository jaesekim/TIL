# Database DML(Data Manipulation Language)

**환경: SQLite**

# SQLite 시작하기

## alias 설정

1. `cd ~` : git bash here
2. `code .` : vscode 열기
3. `.bashrc` 파일 만들고 `alias sqlite3="winpty sqlite3"` 작성
4. 윈도우 검색: 시스템 환경 변수 편집
    1. 환경변수 클릭
    2. path 더블클릭
    3. 새로만들기 클릭
    4. `C:\sqlite` : \는 원화 표시로 해도 된다.
    5. 확인 눌러서 종료
5. git bash 에서 `sqlite3` (alias 설정 했을 때)또는  `winpty sqlite3` (alias 설정 안 했을 때)입력 후 작동 확인

**alias** 설정 됐음을 가정하고 실습 진행

1. sqlite 실행: `sqlite3`
2. 데이터베이스 파일 열기
    
    `sqlite> .open mydb.sqlite3` 
    
    `sqlite3 mydb.sqlite3`
    
3. sqlite3 종료하기: `.exit`

# CSV 파일 SQLite 테이블로 가져오기

1. DML.sql, mydb.sqlite3 파일 생성
2. 테이블 생성하기
    
    ```sql
    CREATE TABLE users(
        first_name TEXT NOT NULL,
        last_name TEXT NOT NULL,
        age INTEGER NOT NULL,
        country TEXT NOT NULL,
        phone TEXT NOT NULL,
        balance INTEGER NOT NULL
    );
    ```
    
3. `Run Query` 해서 `.sqlite3` 파일과  `.sql` 파일을 연결 시켜준다.
4. 데이터베이스 파일 열기: `sqlite3 mydb.sqlite3`
5. 모드(.mode)를 csv로 설정: `.mode csv`
6. .import 명령어를 사용하여 csv 데이터를 테이블로 가져오기: `.import users.csv users`
7. import 된 데이터 확인하기
    
    (sqlite3 tool에서도 SQL문을 사용할 수 있지만 실습 편의와 명령어 기록을 위해 sql 확장자 파일에서 진행하도록 한다.)
    

# Simple Query

**`SELECT`문을 사용하여 간단하게 단일 테이블에서 데이터를 조회하기**

## SELECT statement

`SELECT column1, column2 FROM table_name;`

- 특정 테이블에서  데이터를 조회하기 위해 사용
- 문법 규칙
    - SELECT 절에서 컬럼 또는 쉼표로 구분된 컬럼 목록을 지정
    - FROM 절(clause)에서 데이터를 가져올 테이블을 지정

### ORDER BY

```sql
SELECT first_name, age, balance 
FROM users
ORDER BY age, balance DESC;
```

- ORDER BY절은 하나 이상의 컬럼을 정렬할 경우 첫번째 열을 사용해서 행을 정렬하고 이후 두번째 컬럼을 사용해서 정렬돼 있는 행을 정렬
- age 오름차순 정렬 이후 balance 기준으로 내림차순 정렬
- SQLite에서 NULL은 다른 값보다 작은 것으로 간주.
    - ASC: 결과의 시작 부분에 NULL 표시
    - DESC: 결과의 끝 부분에 NULL 표시

## Filtering Data

### SELECT DISTINCT

```sql
SELECT DISTINCT country
FROM users;
# 중복없이 모든 지역 조회
```

- 각 컬럼의 중복을 따로 계산하는 것이 아니라 두 컬럼을 하나의 집합으로 보고 중복을 제거
    
    ```sql
    SELECT DISTINCT first_name, country
    FROM users;
    ```
    
- SQLite는 NULL값을 중복으로 간주한다.
- NULL 값이 있는 컬럼에 DISTINCT 절을 사용하면 SQLite는 NULL값의 한 행을 유지한다.

### WHERE

- 조회 시 특정 검색 조건을 지정
- WHERE 절은 SELECT 문에서 선택적으로 사용할 수 있는 절
    
    (SELECT 외에도 UPDATE 및 DELETE 문에서도 WHERE 절 사용 가능)
    
- FROM절 뒤에 작성

```sql
WHERE column_1 = 10

WHERE column_2 LIKE 'Ko%'

WHERE column_3 IN (1, 2)

WHERE column_4 BETWEEN 10 AND 20
```

- 비교 연산자
    
    `=`, `<> 또는 !=`, `<`, `<=`
    
- 논리연산자
    - 표현식의 truth를 테스트 할 수 있다.
    - 1, 0 또는 NULL  값 반환
    - SQLite는 Boolean 데이터 타입을 제공하지 않으므로 1은 True를 의미하고 0은 False를 의미한다.
    - ALL, AND, ANY, BETWEEN, IN, LIKE, NOT, OR 등

### LIKE

- Query data based on pattern matching
- 패턴 일치를 기반으로 데이터를 조회
- SELECT, DELETE, UPDATE 문의 WHERE 절에서 사용
- 기본적으로 대소문자는 구분하지 않는다
    
    → ‘A’ LIKE ‘a’는 True
    
- SQLite는 패턴 구성을 위한 두 개의 와일드카드를 제공한다.
    1. `%` : 0개 이상의 문자가 올 수 있음을 의미
        
        `‘%.txt’` 패턴은 `‘.txt’` 로 끝나는 모든 문자열을 찾는 것이다.
        
    2. `_` : 단일 문자가 있음을 의미.

```sql
# 이름에 '호'가 포함되는 사람들의 이름과 성 조회

SELECT first_name, last_name
FROM users
WHERE first_name LIKE '%호%';
```

```sql
# 서울 지역번호 가지고 있는 사람들 조회

SELECT first_name, phone
FROM users
WHERE phone LIKE '02-%';
```

```sql
# 나이가 20대인 사람들 이름과 나이 조회

SELECT first_name, age
FROM users
WHERE age LIKE '2_';
```

```sql
# 전화번호 중간 4자리가 51로 시작하는 사람들의 이름과 전화번호 조회

SELECT first_name, phone
FROM users
WHERE phone LIKE '%-51__-%';
```

### IN

- Determine whether a value matches any value in a list of values
- 값이 값 목록 결과에 있는 값과 일치하는지 확인
- 표현식이 값 목록의 값과 일치하는지 여부에 따라 True or False 반환
- IN 연산자의 결과를 부정하려면 NOT IN 연산자를 사용

```sql
# 경기도 혹은 강원도에 사는 사람들의 이름과 지역 조회

# 버전 1
SELECT first_name, country
FROM users
WHERE country IN ('경기도', '강원도');

# 버전 2
SELECT first_name, country
FROM users
WHERE country = '경기도' OR country = '강원도';

# 틀린 버전 1
SELECT first_name, country
FROM users
WHERE country IN ('경기도' OR '강원도');

# 틀린 버전 2
SELECT first_name, country
FROM users
WHERE country IN '경기도' OR country IN '강원도';
```

### BETWEEN

- 값이 값 범위에 있는지 테스트
- 값이 지정된 범위에 있으면 True 반환
- SELECT, DELETE, UPDATE 문의 `WHERE` 절에서 사용 가능하다.
- BETWEEN 연산자의 결과를 부정하려면 `NOT BETWEEN` 연산자를 사용한다.

```sql
# 나이가 20살 이상 30살 이하인 사람들의 이름과 나이 조회

SELECT first_name, age
FROM users
WHERE age BETWEEN 20 and 30;

# 나이가 20살 이상 30살 이하가 아닌 사람들의 이름과 나이 조회

SELECT first_name, age
FROM users
WHERE age NOT BETWEEN 20 and 30;
```

### LIMIT

- “Constrain the number of rows returned by a query”
- 쿼리에서 반환되는 행 수를 제한
- SELECT 문에서 선택적으로 사용할 수 있는 절

```sql
SELECT rowid, first_name
FROM users
LIMIT 10;
```

```sql
# 계좌 잔고가 가장 많은 10명의 이름과 계좌 잔고 조회

SELECT first_name, balance
FROM users
ORDER BY balance DESC
LIMIT 10;

# 나이가 가장 어린 5명의 이름과 나이 조회

SELECT first_name, age
FROM users
ORDER BY age
LIMIT 5;
```

### OFFSET

- LIMIT 절을 사용하면 첫번째 데이터부터 지정한 수만큼 데이터를 받아올 수 있지만 OFFSET을 함께 사용하면 특정 지정된 위치에서부터 데이터를 조회할 수 있다.

```sql
# 11번째부터 20번째 데이터의 rowid와 이름 조회하기

SELECT rowid, first_name
FROM users
LIMIT 10 OFFSET 10;
```